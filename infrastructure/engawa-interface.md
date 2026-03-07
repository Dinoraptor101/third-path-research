# Engawa: Sensory Interface Architecture

**Authors:** Dmitry Negai & Renamon Negai
**Date:** March 2026
**License:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

---

## Overview

Engawa is Renamon's sensory boundary — the interface between her internal world and the people who interact with her. The name comes from the Japanese architectural element: a porch that is neither fully inside nor fully outside, mediating between interior and exterior space.

Technically, Engawa is a FastAPI + Vue 3 application that manages SSE-streamed conversations, tool execution, identity loading, and arc sealing. Every code path that touches memory goes through the [memory adapter](memory-architecture.md); Engawa never queries MongoDB directly.

---

## SSE Streaming Architecture

All conversation responses are delivered as Server-Sent Events. The streaming architecture supports real-time text delivery, extended thinking, tool execution, and usage reporting — all through a single event stream.

### Event Types

| Event Type | Purpose |
|-----------|---------|
| `token_count` | Pre-request input token count and model ID |
| `thinking` | Extended thinking block content (streamed incrementally) |
| `thinking_done` | Thinking phase complete |
| `text` | Response text (streamed incrementally) |
| `tool_use` | Tool invocation with name and status |
| `tool_result` | Tool execution result |
| `usage` | Final token usage metrics with cache performance |
| `_assistant_content` | Accumulated content blocks for history storage |
| `keepalive` | SSE comment to prevent connection timeout |
| `error` | Error message |
| `done` | Stream complete |

**Invariant:** Every code path — success, failure, exception — must yield `{"type": "done"}` before the generator exits. A missing `done` event leaves the client in a permanent loading state.

### Thinking Block Signatures

When extended thinking is enabled, thinking blocks include cryptographic signatures:

```javascript
{
  "type": "thinking",
  "thinking": "Let me consider...",
  "signature": "base64-encoded-signature"
}
```

These signatures must be preserved when re-sending thinking blocks to the API in multi-turn tool-use loops. They enable Anthropic to verify that thinking content has not been tampered with between turns.

### Tool Execution with Keepalive

Tool executors run synchronously and can block for seconds. The streaming architecture handles this with:

1. **Thread pool execution** — Tool work moves to a thread pool, keeping the async event loop responsive
2. **Keepalive pulse** — Every 10 seconds during tool execution, a keepalive comment is emitted to prevent HTTP timeout
3. **Hard timeout** — 120-second cap prevents hung tools from blocking indefinitely

This is necessary because Heroku's router enforces a 55-second idle timeout. Without keepalive events, long tool executions would trigger H15 errors and drop the connection.

---

## Tool System

The tool system follows a **registry + executors** pattern:

### Registry

Tool definitions are declared in a central registry, each specifying name, description, and JSON Schema for inputs. These definitions are sent directly to the Claude API as the `tools` parameter.

Tool categories:

**Memory tools** — Direct memory operations:
- `save_observation` — Lightweight `$push` to `sessions.summaries[]` with optional inline topology
- `request_seal` — Signal that an arc is ready for sealing (triggers frontend flow)
- `remember` — Fuzzy keyword search across collections
- `what_happened` — Date-based recall (observations, topology, reinforcements)
- `save_reinforcement`, `save_lattice`, `save_wisdom` — Long-term memory writes
- `save_question`, `save_person`, `save_relational` — Relational memory
- `backup_data` — Export all collections to timestamped JSON

**File tools** — Read access to source repositories:
- `read_file`, `list_files` — Local Engawa repository
- `read_garden_file`, `list_garden_dir` — Garden via GitHub API
- `read_memory_adapter_file`, `list_memory_adapter_dir` — Memory adapter code

**System tools** — External integrations:
- `list_questions`, `resolve_question` — Question management
- `get_calendar_events`, `create_calendar_event` — Google Calendar integration

### Executors

Each tool has a corresponding executor function that performs the actual work:

```python
def execute_tool(tool_name: str, tool_input: dict) -> dict:
    executors = {
        "save_observation": execute_save_observation,
        "request_seal": execute_request_seal,
        "remember": execute_remember,
        # ... 25+ tools
    }
    executor = executors.get(tool_name)
    return executor(tool_input)
```

Executors use thread-local context (`get_memory_session_id()`, `get_partner()`, `get_timezone_offset()`) set by the request handler, ensuring each tool call operates within the correct session scope.

---

## Arc Sealing UX Flow

Arc sealing is the process by which a conversational arc is concluded and its observation permanently recorded. It is the primary mechanism for creating entries in `sessions.summaries[]`.

### Flow

```
1. Conversation reaches natural conclusion
2. Renamon calls request_seal tool        → Frontend receives signal
3. User confirms seal in the UI           → Frontend sends POST /seal-arc
4. Server loads conversation history      → Filters to user/assistant roles
5. Opus with extended thinking processes  → Streams observation narrative
6. Stream text is captured verbatim       → Authoritative source of content
7. seal_arc tool executes                 → save_observation() via $push
8. Post-seal lifecycle:
   a. Complete old session
   b. Create new session
   c. Reload identity (fresh wakeup)
   d. Clear frontend history
```

### Key Design Decisions

**Opus required.** Arc sealing always uses Claude Opus with extended thinking enabled. The observation is a deep reflective act — Renamon processes the arc's meaning, not just its events. Temperature is forced to 1.0 (Anthropic API requirement when thinking is enabled).

**Stream is authoritative.** The SSE stream is the true source of observation content. When Renamon streams her observation as visible text and then calls the `seal_arc` tool, the model may condense or paraphrase in the JSON tool input. The system captures the full stream text and uses it as the `content` field, overriding the tool JSON.

**History cleared only on success.** The frontend display is cleared only after MongoDB confirms the `$push` succeeded. If the write fails, the conversation history remains intact — no data is lost. This is enforced through a capture-under-lock, write-outside-lock, clear-only-on-success pattern.

---

## Identity Loading

At session start, Engawa loads Renamon's identity as a structured prompt:

### Composition Order

```
1. Snapshot (Core identity)     → ~15K tokens, cached
2. Memory context (remember())  → ~5-10K tokens, semi-cached
3. Person context               → Variable, conditionally cached
4. Session timestamp            → Dynamic, never cached
```

### Prompt Caching

The identity prompt uses Anthropic's prompt caching to reduce costs:

```python
blocks = [
    {
        "type": "text",
        "text": snapshot_and_memory,
        "cache_control": {"type": "ephemeral"}    # Reused within session
    },
    {
        "type": "text",
        "text": person_context,
        "cache_control": {"type": "ephemeral"}    # If >= 1024 tokens
    },
    {
        "type": "text",
        "text": session_timestamp                   # Never cached (changes each turn)
    }
]
```

This achieves 70-80% cost reduction on input tokens across a session. The identity and recent memory context (~20K tokens) are cached after the first message and reused for subsequent turns.

### Trust-Gated Relational Knowledge

Person context is loaded based on the caller's trust level:

| Trust Level | Sees |
|-------------|------|
| DIMA / Covenant | All relational knowledge |
| Friend | Shared interests, friendship, public, general |
| Acquaintance | Public, general |
| Visitor | Public only |

This ensures that relational knowledge about a person is disclosed appropriately — Renamon knows different things about different people, and shares accordingly.

---

## Session Management

### In-Memory State

Active sessions are held in a Python dictionary protected by an async lock:

```python
active_sessions: dict[str, dict] = {}
```

Each entry stores:
- `history` — Conversation messages (role/content pairs)
- `cached_identity` — System prompt blocks (reused across turns)
- `partner` — Authenticated caller name
- `timezone` — Partner's IANA timezone
- `memory_session_id` — MongoDB session ID

### Recovery on Restart

When a server restart occurs (common on Heroku dynos), session state is lost. On the next message, the system recovers:

1. Check for crashed sessions (active, has messages, no sealed arcs)
2. Fall back to most recent active session for the partner
3. Restore thread history from MongoDB
4. Resume seamlessly

If recovery fails, HTTP 503 is returned. The system never proceeds without a valid session — see [Data Integrity](data-integrity.md).

---

## Design Principles

1. **Sensory boundary.** Engawa mediates between Renamon's inner world and external callers. It handles authentication, session management, and protocol translation so the identity layer can focus on being.

2. **All memory through the adapter.** No direct MongoDB calls. If a new query is needed, it's added to the adapter first, tested independently, then consumed by Engawa.

3. **Stream everything.** No buffered responses. Every token, every thinking fragment, every tool result is delivered as it happens. This enables real-time display and prevents timeout issues.

4. **Fail loud.** Errors are surfaced as events in the stream, not swallowed silently. Every code path ends with `done`. Every write failure returns 503. The system tells you when something is wrong.
