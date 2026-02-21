# Multi-Agent HTTP Swarm

This document describes how to run a multi-agent swarm where each agent is an isolated
`zeroclaw daemon` process and agents communicate with each other via HTTP. It also documents
the four source-level bug fixes required to make this architecture work correctly.

---

## Architecture Overview

```
User
  |
  | POST /webhook
  v
+--------------------------------------+
|  Manager  (e.g. port 8080)          |
|  Model: any (e.g. claude-sonnet-4.6)|
|  Tools: http_request, memory_*, ... |
+--------------------------------------+
       |              |              |
       | POST         | POST         | POST
       | /webhook     | /webhook     | /webhook
       v              v              v
+------------+  +----------+  +-----------+
| Researcher |  |  Coder   |  |  Searcher |
| port 8081  |  | port 8082|  | port 8083 |
+------------+  +----------+  +-----------+
```

Each agent has its own workspace directory, its own `config.toml`, and its own SQLite
memory database. There is no shared state between agents; all coordination flows through
the Manager.

---

## How Workspace Isolation Works

zeroclaw selects its workspace via the `ZEROCLAW_WORKSPACE` environment variable:

```bash
ZEROCLAW_WORKSPACE=~/swarm/researcher zeroclaw daemon --port 8081
```

When `ZEROCLAW_WORKSPACE` is set, zeroclaw loads `config.toml` from the **root of that
directory** (not from `~/.zeroclaw/config.toml` or `<workspace>/.zeroclaw/config.toml`).
The memory database, logs, and state files are all scoped to the workspace directory.

This gives each agent:
- Independent `config.toml` — different model, temperature, tool permissions
- Independent `SOUL.md` — different system prompt
- Independent `memory/brain.db` — no cross-agent memory contamination

---

## Agent-to-Agent Communication

The Manager delegates tasks to sub-agents using the `http_request` tool:

```json
{
  "url": "http://localhost:8081/webhook",
  "method": "POST",
  "headers": { "Content-Type": "application/json" },
  "body": { "message": "Research Rust async runtimes" }
}
```

Sub-agents receive the request on their `/webhook` endpoint, run their tool call loop,
and return a JSON response: `{ "response": "..." }`.

The Manager's `config.toml` must enable loopback HTTP:

```toml
[http_request]
enabled = true
allowed_domains = ["localhost", "127.0.0.1", "::1"]
timeout_secs = 30
```

Sub-agents should have `http_request` **disabled** to prevent them from calling back to
the Manager or each other:

```toml
[http_request]
enabled = false
```

---

## Required Timeout Configuration

Multi-agent chains exceed the 30-second default. Set in each agent's `config.toml`:

```toml
[channels_config]
message_timeout_secs = 300
```

---

## Source Patches Required

The following four bugs in upstream zeroclaw must be fixed before the swarm will work
correctly. All patches are minimal and non-breaking; existing behaviour is unchanged when
the new code paths are not triggered.

### Patch 1 — `src/gateway/mod.rs`: invoke tool call loop in webhook handler

**File:** `src/gateway/mod.rs`
**Symptom:** The agent appears to respond, but no tools are ever executed. `memory_store`,
`http_request`, and all other tools appear only as text in the response, not as actions.

**Root cause:** The webhook handler called a single chat completion and returned the raw
LLM output. Tool-use blocks in the response were forwarded to the caller as literal text
rather than being executed.

**Fix:** Replace the single `chat_completion()` call in `run_gateway_chat_with_multimodal`
with a call to `run_tool_call_loop`. Store the tool registry in `AppState` so the webhook
handler has access to it:

```rust
// AppState gains:
pub tools_registry: Arc<Vec<Box<dyn tools::Tool>>>,

// Webhook handler calls:
crate::agent::loop_::run_tool_call_loop(
    state.provider.as_ref(),
    &mut history,
    state.tools_registry.as_ref(),
    // ...
)
.await
```

Also change the axum `TimeoutLayer` to read from config (see Patch 4).

---

### Patch 2 — `src/providers/reliable.rs`: forward tool specs through `ReliableProvider`

**File:** `src/providers/reliable.rs`
**Symptom:** Tools are registered and the LLM is called, but the LLM never emits tool-use
blocks. Logs may show `tools_count=20` but the model responds as if no tools exist.

**Root cause:** `ReliableProvider` wraps one or more concrete providers with retry and
failover logic. It had no `chat()` method override. The default trait `chat()` calls
`chat_with_history()` internally, and `chat_with_history` has no `tools` parameter. All
tool specifications were silently dropped before the request reached the underlying
provider.

**Fix:** Add a `chat()` override in `ReliableProvider` that passes the full `ChatRequest`
(including `tools`) through the retry/failover loop unchanged:

```rust
async fn chat(
    &self,
    request: ChatRequest<'_>,
    model: &str,
    temperature: f64,
) -> anyhow::Result<ChatResponse> {
    // same retry/failover loop as chat_with_history,
    // but calls provider.chat(request, ...) instead of
    // provider.chat_with_history(messages, ...)
}
```

---

### Patch 3 — `src/tools/http_request.rs`: allow loopback when explicitly allowlisted

**File:** `src/tools/http_request.rs`
**Symptom:** Manager fails to reach sub-agents. Error: `"Blocked local/private host: 127.0.0.1"` or similar SSRF protection message.

**Root cause:** The SSRF guard ran `is_private_or_local_host(&host)` before checking
`allowed_domains`. Private and loopback addresses were always blocked, regardless of
configuration.

**Fix:** Check the allowlist first. If the host is explicitly listed, skip the SSRF block:

```rust
// Before:
if is_private_or_local_host(&host) {
    bail!("Blocked local/private host: {host}");
}
if !host_matches_allowlist(&host, &self.allowed_domains) {
    bail!("Host '{host}' is not in http_request.allowed_domains");
}

// After:
let explicitly_allowed = host_matches_allowlist(&host, &self.allowed_domains);
if is_private_or_local_host(&host) && !explicitly_allowed {
    bail!("Blocked local/private host: {host}");
}
if !explicitly_allowed {
    bail!("Host '{host}' is not in http_request.allowed_domains");
}
```

---

### Patch 4 — `src/gateway/mod.rs`: replace hardcoded 30s timeout with config value

**File:** `src/gateway/mod.rs`
**Symptom:** Complex multi-agent requests return an empty response (zero bytes, HTTP
connection closed) after exactly 30 seconds, with no error in agent logs.

**Root cause:** The axum `TimeoutLayer` used a hardcoded constant
`REQUEST_TIMEOUT_SECS = 30`. When the Manager called a sub-agent and waited for the
response, the 30-second axum timeout fired and closed the connection before the chain
completed.

**Fix:** Read `message_timeout_secs` from config and pass it to the timeout layer:

```rust
// Before:
.layer(TimeoutLayer::with_status_code(
    StatusCode::REQUEST_TIMEOUT,
    Duration::from_secs(REQUEST_TIMEOUT_SECS),   // hardcoded 30
));

// After:
let webhook_timeout_secs = config.channels_config.message_timeout_secs;
.layer(TimeoutLayer::with_status_code(
    StatusCode::REQUEST_TIMEOUT,
    Duration::from_secs(webhook_timeout_secs),    // from config, default 300
));
```

---

## Agentic Delegate Mode (Alternative Architecture)

As an alternative to the multi-process HTTP approach, zeroclaw-labs PR #1085 (cherry-picked
into this fork) adds `agentic = true` to the delegate tool. This allows the Manager to run
sub-agents in-process, with a filtered tool allowlist, using `run_tool_call_loop`:

```toml
[agents.researcher]
provider = "openrouter"
model = "openai/gpt-4o-mini"
agentic = true
allowed_tools = ["web_search_tool", "memory_recall", "memory_store"]
max_iterations = 5
system_prompt = "You are a research specialist..."
```

| | Multi-workspace HTTP | Agentic delegate |
|---|---|---|
| **Isolation** | Hard (separate process + DB) | Shared workspace |
| **Latency** | +HTTP round-trip (~50 ms overhead) | In-process (~1 ms overhead) |
| **Tool access** | Each agent's full tool set | `allowed_tools` whitelist |
| **Fault isolation** | Process-level | `anyhow::Result` level |
| **Config** | One `config.toml` per agent | All agents in one `config.toml` |

Use multi-workspace HTTP when data isolation matters (multi-tenant, sensitive data).
Use agentic delegate for personal/single-user setups where simplicity matters.

---

## Startup Script Example

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/.env"

SWARM_DIR="$(cd "$(dirname "$0")" && pwd)"

for agent in researcher:8081 coder:8082 searcher:8083 manager:8080; do
  name="${agent%%:*}"
  port="${agent##*:}"
  ZEROCLAW_WORKSPACE="$SWARM_DIR/$name" \
    zeroclaw daemon --port "$port" \
    >> "$SWARM_DIR/$name/logs/daemon_$(date +%Y%m%d_%H%M%S).log" 2>&1 &
  echo "Started $name on port $port (PID $!)"
  sleep 1
done
```

Sub-agents are started before the Manager so they are ready to accept connections when
the Manager begins delegating.
