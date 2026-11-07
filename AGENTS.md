# `mu-observable` Workspace Development Guidelines

> [!IMPORTANT]
> **Scope of `AGENTS.md`**: This file is reserved strictly for **immediate, universal behavioral rules and hard invariants** that govern every agent interaction. Do NOT place occasional notes, component designs, framework preferences, or operational runbooks here; those belong in repository-specific documentation (`README.md`, `docs/`).

## 1. Branching & Pull Request Workflow (feat/\*\*)

- **Feature Branch Invariant**: All ongoing changes intended for `main` must use `feat/**` branches (e.g. `feat/m8-naming-alignment`) and be merged via Pull Requests. Never commit or push directly to `main`.
- **Signed Commits**: `commit.gpgsign` stays enabled (`true`). Never disable, override, or bypass commit signing (`commit.gpgsign = false`, `--no-gpg-sign`).
- **Pre-PR Local Verification Gate Sequence**:
  1. `(cd code && cargo fmt --all -- --check)`
  1. `./scripts/format_docs.sh --check`
  1. `(cd code && cargo clippy --workspace --all-targets -- -D warnings)`
  1. `(cd code && cargo test --workspace)`
  1. `cargo agents plugin validate Symposium.toml --no-check-crates`

## 2. Two-Tier Licensing Policy & Public Repository Hygiene

- **Two-Tier Licensing Policy**:
  - `code/ipc` (`mu-observable-ipc`): `LGPL-3.0-only WITH LGPL-3.0-linking-exception`.
  - Daemons, query engines, and services (`code/hub`, `code/collector`, `code/sentinel`): `AGPL-3.0-only`.
- **Public Repository Privacy Boundary**:
  - Never commit internal lab hostnames, private LAN IP addresses, or internal DNS records.
  - Use generic placeholders (`node-a`, `node-b`, `worker-riscv64`, `192.0.2.x`) in public documentation, code, and issues.

## 3. SpacemiT K3 Target Architecture & Sandboxing Invariants

- **Cluster Architecture**:
  - Cluster 0 (`X100`, cores 0-7, mask `0x00FF`): General-purpose Out-of-Order cores, `VLEN=256` (`vlenb=32`).
  - Cluster 1 (`A100`, cores 8-15, mask `0xFF00`): AI/Matrix In-Order cores, `VLEN=1024` (`vlenb=128`), supports `XSMTVDotII`. Conditional `/proc/set_ai_thread` registration on SpacemiT-patched kernels.
- **Sandboxing & Memlock Constraints**:
  - Daemon processes run confined under `MuIsolation` `KernelSandbox` (`Landlock` V3+ and unprivileged namespaces).
  - Bounded `RLIMIT_MEMLOCK` (default `64 MiB`, never `unlimited`).

## 4. Rust Types & Conventions

- Use `usize` for collection indexing, lengths, or memory allocation.
- Use `u32` for domain-specific data, wire protocols, IDs, counters, or network packets where exact 32-bit width is required.
