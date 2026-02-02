# CLAUDE.md

## Project Overview

Dora Studio is a desktop GUI for managing [Dora](https://github.com/dora-rs/dora) dataflows, with an integrated AI chat (Claude API) and SigNoz trace visualization. Built with the [Makepad](https://github.com/makepad/makepad) GPU-accelerated UI framework.

## Build & Run

```bash
cargo build            # dev build
cargo build --release  # release build
cargo run              # run the app
```

## Test Commands

```bash
cargo test --lib                    # unit tests (102 tests)
cargo test --workspace              # all tests
cargo test --test integration       # integration tests
cargo test --test e2e -- --ignored  # e2e tests (requires running Dora daemon)
```

## Architecture

### Module Layout

```
src/
├── app.rs              # Main App: event loop, tab switching, auto-refresh
├── lib.rs              # Module exports with cfg gating
├── api.rs              # Claude API async bridge (global statics + Tokio)
├── tools.rs            # Dora CLI tool wrappers (native only)
├── chat/               # Chat UI widget
├── dataflow/           # Dataflow list table widget
├── otlp/               # OTLP telemetry client (native only)
│   ├── bridge.rs       # Async bridge: env config, background runtime, channels
│   ├── config.rs       # BackendConfig, AuthMethod, SigNozConfig
│   ├── types.rs        # Span, TraceQuery, LogQuery, MetricQuery
│   ├── backend.rs      # TelemetryBackend trait
│   └── signoz/         # SigNoz client, query builder, response parser
└── traces/             # Traces panel widget (native only)
```

### Key Patterns

- **Makepad widgets**: `live_design!` macro for declarative UI, `#[derive(Live, LiveHook, Widget)]`, `PortalList` for virtual scrolling
- **Async bridge**: Global `Mutex<Option<Runtime>>` statics, background Tokio thread, `mpsc::unbounded_channel` for requests, polling via `take_*_responses()` in frame loop
- **Platform gating**: `#[cfg(not(target_arch = "wasm32"))]` on `tools`, `otlp`, `traces` modules
- **Panel switching**: `apply_over(cx, live! { height: Fill/0 })` to toggle view visibility
- **Auto-refresh**: `NextFrame` scheduling at 5-second intervals

### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `ANTHROPIC_API_KEY` | Claude API key for chat | (none) |
| `SIGNOZ_BASE_URL` | SigNoz server URL | `http://localhost:8080` |
| `SIGNOZ_API_KEY` | SigNoz API key auth | (none) |
| `SIGNOZ_EMAIL` | SigNoz login email (JWT auth) | (none) |
| `SIGNOZ_PASSWORD` | SigNoz login password (JWT auth) | (none) |

### Dependencies

- **UI**: `makepad-widgets` (git, branch=dev)
- **Async**: `tokio` (rt-multi-thread on native)
- **HTTP**: `reqwest` (rustls-tls)
- **Serialization**: `serde`, `serde_json`

## Code Style

- Follow existing Makepad widget patterns (see `dataflow_table.rs` as reference)
- Use `log!()` macro for debug logging (Makepad's built-in)
- Use `eprintln!()` for bridge/backend startup messages
- Tests go in `#[cfg(test)] mod tests` at bottom of each file
- Env-var-touching tests must acquire `ENV_LOCK` mutex to avoid races
