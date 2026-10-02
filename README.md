# bitty-agent

An independent L1 Rust Core Extension, extracted from [`bitty`](https://github.com/bitty-terminal/bitty) (Bitty Core), providing the generic, host-neutral AI agent protocol vocabulary for Bitty: bounded messages, tool-call stubs, read-only observations, and bounded coordination queues. No LLM I/O, no network, no window/GPU coupling.

## Status

Pre-1.0 (`version = "0.0.1"`). **Accepted contract, implementation not yet verified.** The IPC and Agent RFC closed `OQ-018` on 2026-08-29, and the DevTools RFC closed `OQ-019` on 2026-08-28 (see the `bitty-docs` open-questions register). This crate's implementation is `Implemented`, not yet `Verified`; do not describe its behavior as shipped until a release ships it.

## Crates

| Crate | Role |
|---|---|
| [`bitty-agent-api`](crates/bitty-agent-api) | Zero-dependency pure types: `AgentId` (owner-qualified identity, `owner.name`), `AgentError`/`ErrorClass`. The only crate cross-repo consumers depend on for API contracts. |
| [`bitty-agent`](crates/bitty-agent) | Protocol implementation: `AgentMessage`, `ToolSpec`/`ToolCall`/`ToolResult`/`ToolRegistry` (stub-only, deterministic, no execution), `AgentObservation` (read-only, untrusted-labeled), `SideQueue` (bounded, never-blocking), `AgentSession` (owns identity, registry, and bounded history). |

Wire framing, transport, authentication, scopes, and rate limits live in [`bitty-ipc`](https://github.com/bitty-terminal/bitty-ipc), not here; this crate intentionally has no path dependency on it today (documented seam, not a hidden coupling).

## Consumers

Bitty Core (`bitty`) does **not** link this crate: the small-core split (bitty CTX-0918) removed it, because agent vocabulary is AI policy, not terminal mechanism. The intended consumer is the AI runtime ([`bitty-ai`](https://github.com/bitty-terminal/bitty-ai)), which runs out of process as the `ai` native component (DIR-030) and reaches the terminal through Core's inbound `bitty-ipc` socket. No consumer pins this repository yet.

Consumers take it as an exact-rev Git dependency, never a branch or tag, per workspace policy:

```toml
[dependencies]
bitty-agent = { git = "https://github.com/bitty-terminal/bitty-agent", rev = "<7-char-sha>" }
```

Bump a pin only through a scoped task once this repo's `main` is green.

## Usage

```rust
use bitty_agent::{AgentId, AgentObservation, AgentSession, ToolSpec};

let id = AgentId::new("local.assistant").unwrap();
let mut session = AgentSession::new(id, 16);
session.declare_tool(ToolSpec {
    name: "read_file".into(),
    description: "read a bounded file".into(),
    input_schema: r#"{"type":"object","properties":{"path":{"type":"string"}}}"#.into(),
}).unwrap();

session.push_user("summarize the terminal output").unwrap();
session.push_observation(AgentObservation::TerminalOutput { text: "hello from pty".into() }).unwrap();

let call = bitty_agent::ToolCall::new("call-1", "read_file", r#"{"path":"/tmp/x"}"#).unwrap();
session.push_assistant("will read", vec![call.clone()]).unwrap();
let results = session.stub_dispatch(&[call]).unwrap();
session.push_tool_results(results).unwrap();
session.complete().unwrap();
```

See [`crates/bitty-agent/src/lib.rs`](crates/bitty-agent/src/lib.rs) for the full crate-level documentation, bounds table, and security alignment notes.

## Boundaries

- **Zero production dependencies on other workspace crates** (pure `std` plus `thiserror`): no network I/O, no LLM calls, no real tool dispatch, no window/GPU coupling.
- **Stub-only tools**: `ToolRegistry::stub_invoke` returns a deterministic placeholder result; it never contacts an LLM or executes a tool.
- **Read-only observations**: `AgentObservation::TerminalOutput` is labeled `is_untrusted_surface() == true`.
- **Bounded everything**: message content `<= 32 KiB`, frame `<= 64 KiB`, tool args/results `<= 16 KiB` each, `<= 8` tool calls per turn, `<= 32` tools per agent, `<= 128` messages / `<= 256 KiB` per session.

Model selection, authentication, consent, streaming, real tool dispatch, compaction, and MCP transport live outside this crate.

## Build and test

All quality gates run through the [`justfile`](justfile), never bare tool invocations:

```sh
just check        # fmt-check + clippy -D warnings + test
just fmt-check
just clippy
just test
just typecheck
just actionlint
```

Toolchain channel is pinned in [`rust-toolchain.toml`](rust-toolchain.toml); MSRV is `1.85` (`rust-version` in the workspace root `Cargo.toml`).

## Security Alignment

This crate implements the generic protocol beneath the accepted IPC/Agent RFC while preserving the normative security invariants from the [bitty-docs security corpus](https://github.com/bitty-terminal/bitty-docs/tree/main/docs/security):

- **Read-only by default** (invariant 5)
- **Least-privilege scopes** (invariant 6)
- **Untrusted observation labeling** (`T-10`, `R-013`)
- **Per-client consent** (`P0-AC-024`)

The crate never weakens these invariants.

## License

MIT OR Apache-2.0
