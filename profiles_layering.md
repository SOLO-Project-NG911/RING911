# RING911 Profiles and Layering Strategy

This document captures the detailed design for separating the **reference implementation** from the **production deployment** of RING911, while keeping both in a single codebase.

## 1. Goals
- Preserve a **clean, readable reference implementation** that aligns with STA-010 and is easy to review, test, and reason about.
- Provide a **fully operational, HA-capable production deployment** without polluting reference logic with deployment details.
- Ensure both profiles share the same business logic, schemas, and interface definitions.

## 2. Profiles Overview
| Profile | Purpose | Characteristics |
|---------|----------|------------------|
| `ref` | Reference implementation | In-memory repos, minimal dependencies, no HA logic, works with CONG |
| `prod` | Production deployment | Postgres, HA, mTLS, observability, retries, backpressure, GitOps config |

## 3. Workspace Structure (Proposed)

    ring/
    ├─ crates/
    │  ├─ core/                     # Core types, state machines, protocol logic (no I/O)
    │  ├─ repo-trait/               # Repository trait definitions
    │  ├─ repo-mem/                 # In-memory repository (reference)
    │  ├─ repo-postgres/            # Postgres repository (production)
    │  ├─ transport-sip/            # Safe wrapper over reSIProcate
    │  ├─ transport-http/           # Shared HTTP server (tower/hyper)
    │  ├─ esrp-core/                # ESRP logic (no networking, no database)
    │  ├─ esrp-server/              # HTTP/SIP adapters, no HA logic (ref)
    │  ├─ esrp-prod/                # Composes core + adapters + HA, metrics, mTLS, etc.
    │  └─ ... (repeat for other FEs)
    ├─ profiles/
    │  ├─ ref/                      # Config, docker-compose, example deployments
    │  └─ prod/                     # Config, ansible roles, secrets via sops, HAProxy configs, bind zones
    └─ Cargo.toml

## 4. Layering Rules
- Core FEs **never import databases, TLS, metrics, or tower middleware**.
- All external dependencies (DBs, SIP stack, HTTP server, PKI, logging) live in **adapters**.
- Production logic is composed using **dependency injection + tower layers**.

## 5. Repository Pattern

    // Rust trait (example)
    pub trait Repository {
        fn get_call(&self, id: &CallId) -> Result<Option<CallState>, RepoError>;
        fn save_call(&self, state: &CallState) -> Result<(), RepoError>;
    }

    // Reference profile
    let repo = InMemoryRepo::new();

    // Production profile
    let repo = PostgresRepo::connect(pg_config).await?;

## 6. Transport and Middleware Differences
| Concern | Reference (`ref`) | Production (`prod`) |
|---------|--------------------|---------------------|
| SIP stack | reSIProcate wrapper | Same + strict TLS, overload control |
| HTTP server | Basic Hyper/Tower | Tower layers: mTLS, rate-limit, tracing, metrics |
| Proxy | None | **HAProxy** sidecar handles mTLS, retries, timeouts, health checks, and metrics (Envoy optional) |
| Metrics | Simple counters | `/metrics` endpoint, OTLP/Prometheus export |
| TLS | Dev certificates | PCA-issued mTLS, cert rotation |
| Config | Inline/local files | GitOps, sops+age secrets, signed bundles |
| Persistence | In-memory | Postgres |
| Call state | Local only | Replicated/idempotent upserts or tokenized state per FE design |

### 6.1 Proxy Choice
Default proxy is **HAProxy** (ingress/egress mTLS, health checks, retries, metrics).  
**Envoy** is supported as an alternate profile where native OTLP/gRPC filters or mesh features are mandated by the environment.

## 7. CI/CD Enforcement
- `cargo test --features ref` and `cargo test --features prod` both MUST pass.
- **CONG** is a **Python** suite (pytest + pjsip) and runs only against the `ref` profile over mTLS endpoints; it does not compile/link against RING.
- Smoke/HA tests run on `prod` profile.
- Reject PRs if:
  - Core logic references production crates.
  - Interface schemas diverge between profiles.

## 8. Future Work
- Provide a full ESRP example wiring for ref vs prod.
- Generate dependency diagrams.
- Automate profile builds using Nix or Make.

---
This document is the detailed companion to the brief **Profiles & Layering** section in the main project definition.
