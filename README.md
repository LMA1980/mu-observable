# mu-observable

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs">
    <img src="https://hook.lma1980.duckdns.org/cards/LMA1980/mu-observable/card.svg" alt="mu-observable Scorecard" width="520"/>
  </a>
</p>

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-observable/status.svg" alt="CI Status"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-observable/score.svg" alt="Quality Score"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-observable/coverage.svg" alt="Coverage"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-observable/benchmark.svg" alt="Benchmark"/></a>
</p>

______________________________________________________________________

Unified telemetry, observability, and regression analytics workspace for the **Jero-Mu** operating platform. `mu-observable` aggregates distributed OpenTelemetry (OTEL) logs, W3C trace waterfalls, hardware-stratified benchmark regressions, 30-day test flakiness analytics, and canary deployment SLO evaluation across hybrid edge and central storage nodes.

______________________________________________________________________

## Workspace Architecture

`mu-observable` is organized into four modular crates located under `code/`:

| Submodule                                   | Target                 | License                                         | Responsibility                                                                                                                                                         |
| :------------------------------------------ | :--------------------- | :---------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`mu-observable-hub`](code/hub)             | native daemon / server | `AGPL-3.0-only`                                 | Central telemetry aggregator, hybrid DuckDB (7-day hot) + Hive-partitioned Parquet (cold) analytics engine, mTLS ingestion receiver, and state store                   |
| [`mu-observable-collector`](code/collector) | native daemon          | `AGPL-3.0-only`                                 | Edge agent ingesting local OTLP/HTTP JSON (`127.0.0.1:4318`) and Arrow streams, secret scrubbing via `EdgeSecretScrubber`, bounded ring-buffer WAL, and mTLS forwarder |
| [`mu-observable-sentinel`](code/sentinel)   | native daemon          | `AGPL-3.0-only`                                 | Edge host watchdog monitoring `/dev/kmsg`, PSI, block I/O latency, Btrfs errors, and socket pressure under `MuIsolation` confinement                                   |
| [`mu-observable-ipc`](code/ipc)             | native library         | `LGPL-3.0-only WITH LGPL-3.0-linking-exception` | Shared zero-copy serialization, Apache Fory schemas, and Apache Arrow wire framing (`MUOB` magic protocol)                                                             |

______________________________________________________________________

## Tooling & Interfaces

### 1. Model Context Protocol (MCP) Server: `mu-observable-mcp`

The workspace builds `mu-observable-mcp`, an Antigravity-native MCP server providing 10 structured tools over `stdio` or HTTP (`POST http://127.0.0.1:9480/mcp`):

- **`check_benchmark_regressions`**: Evaluates candidate runs against rolling `main` baselines stratified by `(arch, cpu_cluster, vlen_bits)`.
- **`get_test_flakiness`**: Computes rolling 30-day pass/fail flip-flop flakiness indices per test case.
- **`compare_branches`**: Compares benchmark performance across candidate and baseline branches.
- **`query_trace_waterfall`**: Reconstructs hierarchical parent-child W3C distributed trace spans.
- **`correlate_logs_by_trace_id`**: Retrieves structured logs correlated with a specific W3C `trace_id`.
- **`search_error_anomalies`**: Scans recent `ERROR` and `FATAL` logs across services and edge nodes (`node-a`, `worker-riscv64`).
- **`assert_service_slo`**: Evaluates continuous delivery canary bake-window SLOs (`max_error_logs`, `max_p99_latency_ms`).
- **`inspect_node_utilization`**: Inspects hardware telemetry, memlock ceilings, and CPU cluster utilization.
- **`compare_cluster_efficiency`**: Analyzes SpacemiT K3 Cluster 0 (`X100`, `VLEN=256`) vs Cluster 1 (`A100`, `VLEN=1024`) vs `x86_64` efficiency.
- **`execute_analytics_sql`**: Runs guarded read-only DuckDB SQL queries over unified hot and cold partitions.

### 2. Command-Line Interface: `mu-observable-cli`

Unified diagnostic CLI for local administration, regression evaluations, canary assertions, and Symposium agent integration (`cargo agents mu-observable`):

```bash
# Evaluate benchmark regressions against rolling main baselines
mu-observable-cli --db /var/lib/muobserve/db/muobserve.duckdb \
  regressions --repo lma1980/k3-core --commit HEAD --baseline main

# Assert canary deployment SLO over a 300s window
mu-observable-cli --db /var/lib/muobserve/db/muobserve.duckdb \
  slo --service action-runner-controller --digest HEAD --window-secs 300 --max-errors 0 --max-p99-ms 250.0

# Inspect trace waterfalls
mu-observable-cli --db /var/lib/muobserve/db/muobserve.duckdb \
  traces --trace-id <32-hex-trace-id>

# Run analytical SQL across unified views
mu-observable-cli --db /var/lib/muobserve/db/muobserve.duckdb \
  sql --query "SELECT repo_key, status_state, COUNT(*) AS jobs FROM v_jobs_history GROUP BY repo_key, status_state ORDER BY jobs DESC"
```

______________________________________________________________________

## Local Verification & Build Sequence

To verify the entire workspace locally:

```bash
# 1. Check workspace compilation
(cd code && LIBRARY_PATH=/opt/homebrew/lib DYLD_LIBRARY_PATH=/opt/homebrew/lib cargo check --workspace)

# 2. Check Rust formatting
(cd code && cargo fmt --all -- --check)

# 3. Check documentation formatting
./scripts/format_docs.sh --check

# 4. Check workspace linter rules
(cd code && LIBRARY_PATH=/opt/homebrew/lib DYLD_LIBRARY_PATH=/opt/homebrew/lib cargo clippy --workspace --all-targets -- -D warnings)

# 5. Execute test suite
(cd code && LIBRARY_PATH=/opt/homebrew/lib DYLD_LIBRARY_PATH=/opt/homebrew/lib cargo test --workspace)

# 6. Validate Symposium manifest
cargo agents plugin validate Symposium.toml --no-check-crates
```

______________________________________________________________________

## Two-Tier Licensing Policy

This repository adheres to the platform two-tier licensing policy:

- **Library & IPC Crate**: [`code/ipc`](code/ipc) is licensed under `LGPL-3.0-only WITH LGPL-3.0-linking-exception`.
- **Daemons & Standalone Services**: [`code/hub`](code/hub), [`code/collector`](code/collector), [`code/sentinel`](code/sentinel), and top-level workspace are licensed under `AGPL-3.0-only`. See [`LICENSE`](LICENSE) for complete terms.
