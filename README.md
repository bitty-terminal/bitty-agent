# bitty-agent

Generic AI agent protocol layer for Bitty Terminal: messages, tools, observations, bounded queues.

## Overview

`bitty-agent` provides the host-neutral Agent protocol vocabulary for Bitty:

- Owner-qualified identity (`AgentId`)
- Bounded messages (`AgentMessage`, `<= 32 KiB` content, `<= 64 KiB` frame)
- Tool vocabulary (`ToolSpec`, `ToolCall`, `ToolResult`, `ToolRegistry`)
- Read-only observations (`AgentObservation`)
- Bounded coordination queues (`SideQueue`, `AgentSession`)

Wire framing, transport, authentication, scopes, and rate limits live in [`bitty-ipc`](https://github.com/bitty-terminal/bitty-ipc).

## Status

**Accepted contract, implementation not yet verified.** The IPC and Agent RFC closed `OQ-018` on 2026-08-29, and the DevTools RFC closed `OQ-019` on 2026-08-28 (see the bitty-docs open-questions register). This crate is `Implemented`, not yet `Verified`; do not describe its behavior as shipped until a release ships it.

## Architecture

```
bitty-agent-api (9K)   ← Zero-dependency pure types (AgentId, AgentError)
    ↓
bitty-agent (621K)     ← Protocol implementation (messages, tools, sessions)
```

### Crates

- **`bitty-agent-api`**: Pure API types with zero dependencies
  - `AgentId`: Owner-qualified identity (`owner.name`, bounded to 128 chars)
  - `AgentError`: Owned error types with classification
  - Suitable for plugin/cross-repo consumption

- **`bitty-agent`**: Complete protocol implementation
  - `AgentMessage`: Bounded message vocabulary (role, content, tool calls/results)
  - `ToolSpec`/`ToolCall`/`ToolResult`/`ToolRegistry`: Tool validation and stub results
  - `AgentObservation`: Read-only snapshots (terminal output, workspace state)
  - `SideQueue`: Bounded observation queue
  - `AgentSession`: Stateful session owning identity, registry, and history

## Usage

```rust
use bitty_agent::{AgentId, AgentMessage, Role, AgentSession};

// Create agent identity
let id = AgentId::new("bitty", "assistant")?;

// Create session
let mut session = AgentSession::new(id);

// Add message
let msg = AgentMessage::new(Role::User, "Hello, agent!".to_string())?;
session.add_message(msg)?;
```

## Boundaries

- **Zero production dependencies**: No network I/O, no LLM calls, no real tool dispatch
- **Stub-only tools**: Deterministic validation results, no external effects
- **Read-only observations**: Terminal output labeled as untrusted
- **Bounded everything**: Message content (32 KiB), frames (64 KiB), tool calls/results (8 each)

Model selection, authentication, consent, streaming, real tool dispatch, compaction, and MCP transport live outside this crate.

## Security Alignment

This crate implements the generic protocol beneath the accepted IPC/Agent RFC while preserving the normative security invariants from [bitty-docs security corpus](https://github.com/bitty-terminal/bitty-docs/tree/main/docs/security):

- **Read-only by default** (invariant 5)
- **Least-privilege scopes** (invariant 6)
- **Untrusted observation labeling** (`T-10`, `R-013`)
- **Per-client consent** (`P0-AC-024`)

The crate never weakens these invariants.

## License

MIT OR Apache-2.0
