# AGENTS.md — runtimed

Rust workspace providing libraries for working with Jupyter kernels and notebooks.

## Repository layout

```
Cargo.toml              # workspace root (resolver = "2")
rust-toolchain.toml     # pinned to stable 1.90.0 + rustfmt + clippy
crates/
  jupyter-protocol/     # core Jupyter message types (no transport)
  jupyter-websocket-client/ # connect to Jupyter servers over WebSockets (tokio + tungstenite)
  jupyter-zmq-client/   # native ZeroMQ kernel client
  jupyter-ysync/        # Y-sync / CRDT collaboration layer for notebooks
  nbformat/             # parse & serialize Jupyter notebook files (.ipynb)
  runtimelib/           # deprecated shim — re-exports jupyter-zmq-client
  mybinder/             # MyBinder build API (no I/O, serde only)
  ollama-kernel/        # reference Jupyter kernel backed by Ollama
```

`default-members` excludes `jupyter-zmq-client`, `runtimelib`, and `ollama-kernel` from a bare
`cargo build` / `cargo test` run because they require ZeroMQ or a running Ollama process.

## Build & test commands

```bash
# Check the default workspace members (no ZMQ, no Ollama)
cargo check

# Build default members
cargo build

# Run all unit/integration tests for default members
cargo test

# Full workspace (includes ZMQ crates — requires libzmq headers)
cargo test --workspace

# Single crate
cargo test -p nbformat
cargo test -p jupyter-protocol
cargo test -p jupyter-websocket-client
cargo test -p jupyter-ysync

# jupyter-zmq-client needs a runtime feature flag
cargo test -p jupyter-zmq-client --features tokio-runtime

# jupyter-ysync client integration tests (requires a live Jupyter server — see below)
cargo test -p jupyter-ysync --features client -- --ignored --test-threads=1 --nocapture

# Lint
cargo clippy --workspace
cargo fmt --check
```

## Crate notes

### `jupyter-protocol`
Foundation crate. Defines all Jupyter wire-format message structs, traits, and errors.
No transport, no async runtime — pure data types and serialization.

### `jupyter-websocket-client`
Connects to Jupyter servers over WebSockets using `tokio` and `async-tungstenite` with
`tokio-rustls`. The `execute` example demonstrates basic kernel execution.

### `jupyter-zmq-client`
Native ZeroMQ client. Feature flags:
- `ring` (default) — use `ring` for HMAC
- `aws-lc-rs` — alternative crypto backend
- `tokio-runtime` — tokio async executor (required for most uses)
- `async-dispatcher-runtime` — smol/async-std executor
- `test-kernel` — enables in-process test kernel utilities

**Tokio mutex lint** (`crates/jupyter-zmq-client/tests/tokio_mutex_lint.rs`): a CI test that
uses `async-rust-lsp` / tree-sitter to assert no `tokio::sync::Mutex` or `tokio::sync::RwLock`
guard is held across an `.await` anywhere in `src/`. Fix by scoping locks in their own block so
the guard drops before the next `.await`. Do not rely on `drop(guard)` — the lint is lexical.
Run with: `cargo test -p jupyter-zmq-client --features tokio-runtime`.

### `jupyter-ysync`
Y-sync (Yjs CRDT) protocol implementation for real-time notebook collaboration.
Feature flags:
- `client` — adds async WebSocket client (`tokio`, `async-tungstenite`, `reqwest`)
- `python` — PyO3 extension module

Integration tests (`crates/jupyter-ysync/tests/integration_test.rs`) are `#[ignore]` by default
and require a live Jupyter server with `jupyter-server-documents`:

```bash
cd /tmp/jupyter-test
uv run --with jupyter-server --with jupyter-server-documents --with jupyterlab \
  jupyter lab --port 18889 --IdentityProvider.token=testtoken123
```

Environment variables: `JUPYTER_URL`, `JUPYTER_TOKEN`, `NOTEBOOK_PATH`.

### `nbformat`
Parses and serializes `.ipynb` files. Supports v4.0–v4.5 plus a legacy shim for older formats.
Conformance tests live in `crates/nbformat/tests/conformance.rs` and load fixture notebooks from
`crates/nbformat/tests/notebooks/`. Add new fixture notebooks there for regression coverage.

### `runtimelib`
Deprecated. Thin re-export of `jupyter-zmq-client`. Feature flags mirror those of
`jupyter-zmq-client`. New code should depend on `jupyter-zmq-client` directly.

### `mybinder`
Minimal serde-only client for the MyBinder build API. No async runtime. Fixtures for
success/failure responses are in `crates/mybinder/fixtures/`.

### `ollama-kernel`
Binary crate (`src/main.rs`). A fully working Jupyter kernel that forwards execution to a local
Ollama instance. Not included in `default-members`; build explicitly with
`cargo build -p ollama-kernel`.

## Key conventions

- **Workspace dependencies**: add shared deps in the root `[workspace.dependencies]` table and
  reference them with `{ workspace = true }` in crate `Cargo.toml` files.
- **Error types**: each crate defines its own `Error` / `JupyterError` with `thiserror`.
- **Async runtime**: prefer `tokio` for new async code. `jupyter-zmq-client` additionally
  supports `smol`/`async-std` via the `async-dispatcher-runtime` feature.
- **Crypto**: default to the `ring` feature for HMAC; `aws-lc-rs` is an opt-in alternative.
- **No `tokio::sync::Mutex` across `.await`**: enforced by the lint test in `jupyter-zmq-client`.
  Use `std::sync::Mutex` with a scoped lock instead.
- **Toolchain**: pinned in `rust-toolchain.toml` (stable 1.90.0). Do not change without updating
  that file.
