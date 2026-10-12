
# Rackateebot

A high-performance, MEV-resistant Solana trading and arbitrage engine built in pure Rust. Built for low-latency execution, Rackateebot keeps its scan loop off the network and routes trades through Jito private bundles, so they are not exposed in the public mempool.

---

## Core Architecture

- **Blazing Async Engine** — Built on Tokio and Axum for non-blocking concurrency, streaming real-time account state changes over high-throughput WebSocket/gRPC data feeds without GC pauses.
- **Jito Bundle Integration** — Routes execution payloads directly to block-engine validator leaders via private tip bundles, guaranteeing atomic (all-or-nothing) execution and eliminating public mempool visibility.
- **In-Memory State & Slippage Guard** — Evaluates price impact, pool liquidity depth, and deterministic slippage bounds locally against `DashMap`-backed reserves before ever serializing a transaction instruction — no RPC round-trip on the hot path.
- **Redundant RPC Layer** — Health-checked failover across multiple RPC providers; a single provider incident doesn't stall the engine.
- **Fail-Closed Risk Engine** — Position limits, per-trade notional caps, daily loss limits, rate limiting, and a consecutive-failure circuit breaker sit between every opportunity and every transaction.
- **Full Observability** — Structured JSON tracing, Prometheus metrics, and `/healthz` / `/readyz` endpoints out of the box.

```
                    ┌─────────────┐        ┌──────────────┐
   gRPC / WS  ────▶ │ FeedManager │ ─────▶ │ SharedState   │◀─┐
   market data       └─────────────┘        │ (pool reserves)│ │
                                             └──────┬────────┘ │
                                                    │           │
                                          scan loop │           │ reads
                                                    ▼           │
                                          ┌───────────────────┐│
                                          │ ArbitrageEngine    ││
                                          │  - find_opportunities
                                          │  - SlippageGuard   │
                                          │  - RiskEngine ─────┼┘
                                          └─────────┬──────────┘
                                                    │ approved
                                                    ▼
                                          ┌───────────────────┐      ┌───────────────┐
                                          │ tx_builder         │────▶│ JitoBundleClient│──▶ block-engine
                                          │ (build+sign tx)    │     │ (tip + submit) │
                                          └───────────────────┘      └───────────────┘
```

---

## Tech Stack

| Layer            | Choice                                                        |
|-------------------|----------------------------------------------------------------|
| Language          | Rust (2021 edition, MSRV 1.80)                                  |
| Async runtime     | Tokio (multi-thread scheduler)                                  |
| HTTP/metrics      | Axum, Prometheus exporter                                       |
| Solana interfacing| `solana-client`, `solana-sdk`, gRPC (Geyser-compatible feeds)   |
| MEV protection    | Jito block-engine bundle relay (JSON-RPC)                        |
| Concurrency state | `DashMap`, `parking_lot`, `arc-swap`                             |
| Config            | Layered TOML + env overrides (`config` crate)                   |
| Deployment        | Multi-stage Docker (cargo-chef caching, distroless runtime)     |

---

## Repository Layout

```
rackateebot/
├── Cargo.toml
├── config/
│   └── config.toml          # layered defaults; override per-env or via RACKATEEBOT_* env vars
├── src/
│   ├── main.rs               # process entrypoint / task orchestration
│   ├── lib.rs                 # library root (used by main.rs, tests, benches)
│   ├── config.rs               # config loading + validation
│   ├── error.rs                  # crate-wide error taxonomy
│   ├── telemetry.rs                # tracing/log setup
│   ├── engine/
│   │   ├── arbitrage.rs             # scan/execute loop, opportunity discovery
│   │   ├── pathfinding.rs             # bounded-hop cycle search (2-hop and triangular)
│   │   ├── slippage_guard.rs         # in-memory constant-product simulation
│   │   └── state.rs                    # SharedState: pool reserves, stats, kill switch
│   ├── feeds/
│   │   ├── grpc.rs                      # Geyser-style gRPC feed (provider client is a plug-in point)
│   │   └── websocket.rs                  # public WS accountSubscribe fallback
│   ├── jito/
│   │   ├── bundle.rs                       # bundle submission, retry/backoff, status polling
│   │   ├── jitter.rs                          # OS-CSPRNG tip jitter
│   │   └── tip.rs                            # profit-scaled dynamic tip policy
│   ├── solana/
│   │   ├── client.rs                          # multi-endpoint RPC pool with failover
│   │   ├── dual_router.rs                       # bundle + SWQoS race with double-fill guard
│   │   └── tx_builder.rs                        # transaction/bundle construction
│   ├── risk/
│   │   └── limits.rs                              # position/notional/rate limits, circuit breaker
│   └── metrics/
│       └── server.rs                                # Axum health + Prometheus endpoint
├── tests/integration_test.rs
├── benches/arbitrage_bench.rs
├── docker/
│   ├── Dockerfile             # multi-stage, cargo-chef, distroless, non-root
│   ├── docker-compose.yml     # rackateebot + Prometheus + Grafana
│   └── prometheus.yml
├── scripts/deploy.sh          # scale-up/health-check/scale-down rollout
└── .github/workflows/ci.yml   # fmt, clippy, test, cargo-audit, docker build
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/your-org/rackateebot.git
cd rackateebot
```

### 2. Provision secrets

Rackateebot never reads secret material from config files directly — only paths. Generate or place your keypairs where the config points:

```bash
mkdir -p secrets
solana-keygen new --outfile secrets/dev-keypair.json        # trading wallet
solana-keygen new --outfile secrets/dev-jito-auth.json       # Jito auth identity
```

In production, mount these via Docker/K8s secrets rather than bind-mounting a plaintext directory (see `docker/docker-compose.yml`).

### 3. Configure

```bash
cp .env.example .env
# edit .env and config/config.toml — endpoints, watched pools, risk limits
```

**`engine.dry_run` defaults to `true`.** The engine will scan, evaluate, and log opportunities without submitting any transaction until you deliberately flip it off — see [Safety](#safety--operational-notes).

### 4. Run locally

```bash
cargo build --release
cargo run --release
```

### 5. Or run via Docker

```bash
docker compose -f docker/docker-compose.yml up -d --build
curl http://localhost:9090/healthz
```

Grafana is available at `http://localhost:3000` (default admin password set in `docker-compose.yml` — change it).

---

## Configuration Reference

Config loads in layers, later layers override earlier ones:

1. `config/config.toml` — checked-in defaults
2. `config/<RACKATEEBOT_ENV>.toml` — optional per-environment overrides (`development`, `staging`, `production`)
3. `RACKATEEBOT_*` environment variables, double-underscore nested (e.g. `RACKATEEBOT_ENGINE__DRY_RUN=false`, `RACKATEEBOT_RISK__MAX_SLIPPAGE_BPS=25`)

Key sections:

| Section     | Purpose |
|-------------|---------|
| `[solana]`  | RPC/WS endpoints, commitment level, trading keypair path |
| `[jito]`    | Block-engine URL, tip accounts, dynamic tip bounds/fraction |
| `[feeds]`   | Watched pools/programs, gRPC provider, reconnect/staleness thresholds |
| `[engine]`  | Enabled strategies, min profit floor, scan interval, `dry_run` |
| `[risk]`    | Position/notional/daily-loss caps, max slippage, rate limit, circuit breaker |
| `[metrics]` | Bind address, Prometheus toggle |
| `[telemetry]` | Log level, JSON vs. pretty logs |

---

## Advanced Execution Layer

Six additions push past "solid arbitrage bot" toward what a serious low-latency Solana operation actually runs. Worth being precise about what each one really buys you — the marketing language around this stuff ("see trades before the network does") tends to overstate it:

| Component | File | What it actually does |
|---|---|---|
| Sub-slot ingestion | `feeds/grpc.rs` | Real Yellowstone Geyser gRPC subscription (replaces the placeholder). Gets you account-update data with less lag than polling — it does **not** mean you observe a transaction before the network has processed it. |
| Zero-allocation hot path | `engine/precompiled.rs` | Pre-serializes transaction templates off the hot path (previous slot), patches fixed-offset amount/blockhash fields and re-signs at trigger time. Eliminates repeated instruction-vec/message-compile cost — signing itself still costs a scalar multiplication, that part can't be pre-computed. Requires venue-specific offset resolution before use (`enable_precompiled_hot_path`). |
| Dual-routing | `solana/dual_router.rs` | Races a Jito bundle against SWQoS `sendTransaction` fallbacks (`[swqos]` config), returns whichever lands first. Note: only the bundle path is atomic across hops — a bare SWQoS fallback is a single transaction with its own atomicity, not a second copy of a multi-hop atomic route. The router refuses to send unless the bundle and SWQoS transactions are identical, so both paths cannot land and execute the route twice. |
| Tip jitter | `jito/jitter.rs` | Adds bounded random jitter (default ±5%) to each tip, clamped to your configured min/max, using the OS CSPRNG. Makes bids harder to undercut by a single lamport. A hardware or quantum RNG would add latency without making the output less predictable. |
| Cycle pathfinding | `engine/pathfinding.rs` | Bounded-hop search over a log-rate pool graph that finds profitable 2-hop and triangular cycles on spot prices. A screening step only; candidates still need slippage sizing and the replay gate. Not yet wired into `ArbitrageEngine`. |
| Deterministic local replay | `engine/local_sim.rs` | Tier 1 (implemented): re-runs the constant-product simulation against the freshest in-memory reserves immediately before signing, aborting on drift beyond `max_reserve_drift_bps`. Tier 2 (sketched, `LocalSvmReplayer`): an actually-embedded SVM (litesvm/`solana-program-test`) that executes real program bytecode locally — catches paused pools/program-logic changes that pure AMM math can't, at real (not free) CPU cost. Left as an explicit extension point rather than faked. |

The execution-layer items above are opt-in where config exists via config (`engine.enable_precompiled_hot_path`, `engine.enable_svm_replay`, `[swqos].endpoints`) and default off/empty, consistent with the "start conservative" posture in [Safety](#safety--operational-notes).

## Extension Points

A few seams are deliberately left as explicit extension points rather than guessed at, because getting them wrong risks silently losing funds:

- **`src/solana/tx_builder.rs::build_swap_instruction`** — per-DEX instruction encoding (Raydium AMM, Orca Whirlpool, etc.). Each venue has its own account ordering and instruction discriminants; wire these up against the venue's own IDL/program docs and test against devnet before enabling in `engine.enabled_strategies`.
- **`src/feeds/grpc.rs::connect_and_stream`** — your Geyser-compatible provider's generated `tonic` client (Yellowstone, Triton, etc.). The reconnect/backoff/upsert shell around it is complete.
- **`src/feeds/websocket.rs`** — account payload decoding per pool layout, to convert raw base64 account data into `PoolState`.

---

## Safety & Operational Notes

- **Start in `dry_run`.** Watch logs and Prometheus metrics for opportunity/profit patterns before enabling live submission.
- **Start with a small `risk.max_notional_per_trade_lamports` and `risk.max_position_lamports`.** Scale up only after you've observed real fill/landing behavior.
- **The circuit breaker is fail-closed.** After `risk.circuit_breaker_loss_streak` consecutive failed/unlanded bundles, the risk engine rejects all new trades until manually reset.
- **Keys never touch config files or logs.** Only filesystem paths are configured; mount actual key material as secrets, not source-controlled files (see `.gitignore`).
- **Simulate before send.** `engine.simulate_before_send` runs `simulateTransaction` against the RPC pool before a bundle is submitted; keep this on unless you have a specific low-latency reason not to, and understand the tradeoff.
- This is trading software that moves real funds when `dry_run` is off. Review the risk config, the DEX instruction encoders you wire in, and test thoroughly on devnet/a small live position before scaling capital.

---

## Development

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-features
cargo bench
```

CI (`.github/workflows/ci.yml`) runs formatting, linting, tests, `cargo audit`, and a Docker build on every push/PR.

## License

MIT — see [LICENSE](LICENSE).
