---
description: "Use when bumping, upgrading, or auditing versions: Cargo dependency versions, crate package versions, Rust toolchain channel, devcontainer image tag, cargo tool installs (cargo-llvm-cov, cargo-insta), or Python packages in requirements.txt."
---

# Version Update Guidelines

## 1. External Cargo dependencies

All third-party versions live exclusively in `[workspace.dependencies]` in the root `Cargo.toml`.
Crate `Cargo.toml` files reference them with `{ workspace = true }` and must not repeat a version.

- Update the version **only** in the root `[workspace.dependencies]` table.
- Never add a bare version string (`foo = "1.2"`) to a crate `Cargo.toml` for a dep that is already in the workspace table.
- Exception: `nbformat` currently has `thiserror = "1.0"` as a direct dep (not workspace). Migrate it to `thiserror = { workspace = true }` when touching that crate.

After any version bump run:
```bash
cargo check
```

## 2. Internal (path) crate versions

Each publishable crate carries its own version in `[package].version`. When bumping:

1. Update `version` in the crate's own `Cargo.toml` under `[package]`.
2. Update the matching path entry in the root `[workspace.dependencies]` table — the `version` field there must match.

```toml
# Root Cargo.toml — keep in sync with crate's [package].version
jupyter-protocol = { path = "crates/jupyter-protocol", version = "2.0.3" }
```

Publish in dependency order (see `RELEASING.md`):
`jupyter-protocol` → `nbformat`, `jupyter-websocket-client`, `jupyter-zmq-client` → `runtimelib`, `jupyter-ysync` → `ollama-kernel`  
`mybinder` is standalone.

Use `cargo release` (dry run by default; add `--execute` when ready).

## 3. Rust toolchain

The pinned channel lives in `rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.90.0"   # ← update here
components = ["rustfmt", "clippy"]
```

Do not change this file without confirming the new channel builds cleanly with `cargo build` and all test matrix variants in `.github/workflows/linux.yml`.

## 4. devcontainer image

The Python version is set as a tag on the base image in `.devcontainer/devcontainer.json`:

```json
"image": "quay.io/jupyter/base-notebook:python-3.10"
```

When upgrading Python, also update `.github/workflows/linux.yml`:
```yaml
python-version: "3.10"
```

Both files must name the same Python version.

## 5. Cargo tool installs in devcontainer

Tools installed via `cargo install` in `postCreateCommand` use `--locked` to pin to the
published lockfile. To upgrade to a specific version:

```json
"postCreateCommand": "... && cargo install cargo-llvm-cov@0.6.15 --locked && cargo install cargo-insta@2.0.0 --locked"
```

Omitting the `@version` suffix always installs the latest; use it intentionally or pin explicitly.

## 6. Python packages (requirements.txt)

`requirements.txt` is currently unpinned (`jupyter`, `ipykernel`, `nbclient`).
When pinning or upgrading, use exact versions:

```
jupyter==7.x.x
ipykernel==6.x.x
nbclient==0.x.x
```

After changing `requirements.txt`, re-run the CI job or locally:
```bash
pip install -r .github/requirements.txt
python -m ipykernel install --user --name=python
python -m ipykernel install --user --name=python3
```

## Checklist

- [ ] Root `[workspace.dependencies]` updated for external dep changes
- [ ] Crate `[package].version` and its workspace path entry are in sync
- [ ] `rust-toolchain.toml` channel matches CI and devcontainer expectations
- [ ] devcontainer image tag and CI `python-version` are consistent
- [ ] `cargo check` (and ideally `cargo test`) passes after changes
