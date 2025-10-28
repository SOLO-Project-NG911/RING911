# agents_ring.md – AI Development Guidelines for RING911

This file defines how AI-assisted developers (or LLM-based tools) must generate code for the RING911 reference implementation.

## 1. Purpose

RING911 is a reference implementation of NG9-1-1 Core Services. The code must:
- Match STA-010 and relevant RFCs (especially SIP / LoST).
- Be clear, deterministic, and verifiable.
- Keep core logic free of deployment, database, or infrastructure dependencies.
- Be written in safe Rust with tests and documentation.

These rules ensure AI-generated code remains correct and maintainable.

---

## 2. Core Rules for AI

1. Do not invent behavior — follow STA-010, RFC 3261, or clearly mark as TODO if unclear.
2. Use safe Rust only; no `unsafe` unless inside approved `ffi_*` modules.
3. No `.unwrap()` or `.expect()` in non-test code unless justified with comments.
4. All new code must compile, follow Rustfmt and Clippy defaults.
5. Every public function must have a Rustdoc comment.

---

## 3. Architecture and Layering Constraints

| Rule | Requirement |
|------|-------------|
| No direct DB access | Core crates must not import Postgres, SQLx, or any database drivers. |
| Repository pattern | All persistence must go through traits (e.g. `Repository`). Implementations live in separate crates (repo-mem, repo-postgres). |
| No production logic in core | TLS, metrics, retries, circuit-breakers, config loading belong only in `*-prod` crates. |
| No direct SIP stack in business logic | SIP is handled in transport adapters, not in core logic. Core only consumes normalized request objects. |

If AI tries to import `sqlx`, `hyper`, `tokio-postgres`, or `openssl` inside `*-core`, it is a violation.

---

## 4. Crate Structure for Each Functional Element (Example: ESRP)

crates/
  esrp-core/      → routing logic, state machines, trait definitions
  esrp-server/    → reference HTTP/SIP server, minimal dependencies
  esrp-prod/      → production wiring (mTLS, metrics, retries, HA)
  transport-sip/  → safe wrapper over reSIProcate
  repo-trait/     → defines repository traits
  repo-mem/       → in-memory repository
  repo-postgres/  → Postgres-based repository

---

## 5. Coding Guidelines

- Target Rust 2021 edition.
- Use `thiserror`, `serde`, `async_trait`, and `tracing` when appropriate.
- Errors must use typed enums, not strings.
- Logging uses `tracing::info!`, `tracing::warn!`, etc. No println!.
- Use dependency injection: pass traits into constructors rather than creating connections internally.

---

## 6. Testing Requirements

- All new functionality must include unit tests.
- Tests must not require network, database, or external files unless in `*-prod`.
- Use in-memory repos or mocks in tests.
- If logic depends on SIP or LoST messages, include static sample messages as test fixtures.

---

## 7. Documentation Requirements

Each module must have:
- A brief `README.md` describing what it implements and which STA-010 section it relates to.
- Rustdoc comments on all public items.
- Inline comments when behavior is tied to a specific requirement.

Example of a Rustdoc comment:
Implements ESRP routing for STA-010 Section 3.5.2: “The ESRP shall consult the ECRF with the location…”

---

## 8. Dev & Deploy Rules (Single-Host Lab)

**Intent:** We start on one Hetzner box for speed/cost, without baking single-machine assumptions into code.

### Service addressing
- ✅ Always use **service DNS names** (e.g., `esrp.local.ring`, `geocore.local.ring`), never `localhost`/`127.0.0.1` in code or configs.
- ✅ Add a local zone (bind or /etc/hosts) mapping service names to 127.0.0.x during single-box dev.
- ✅ All HTTP traffic goes through the local proxy (`Envoy` or `HAProxy`) using **mTLS** even on the same host.

### Profiles
- `ref` profile: minimal deps, in-proc axum/reqwest + rustls; no HA logic.
- `prod` profile: same handlers/IDLs, but **fronted by Envoy** with uniform timeouts, retries, health checks, metrics, tracing.

### HTTP/s Networking (standardized)
- Server (ref): axum on loopback; client (ref): reqwest built by `ring-net` client builder.
- Production: FE talks clear-HTTP to local proxy on loopback; proxy does mTLS to peers.
- Defaults (overridable per route): `connect=250ms (intra-site) / 1s (x-site)`, `request=2s (10s bulk)`, `idle=30s`,
  `retries=2 w/ jitter`, hedging allowed for **idempotent** GETs only.

### mTLS & Identity
- Each FE uses a PCA-issued cert with **role OID** and SAN.
- Proxies verify PCA chain + SAN + role; services authorize by role (XACML on external APIs).
- Rotate keys with overlap; export `certExpiringSoon`.

### Storage choices
- GeoCore data **only** via SDPI (snapshot + transactionId catch-up). No Patroni/etcd.
- Policy Store replicated by STA-010 policy replication (not DB HA).
- Logging: LogEvent replication per standard; **SIPREC fans-out media** to all loggers.

### SIP/state
- ESRP/BCF are stateless for dialog routing via **SST** (signed state in `To` tag, pref.; RR param fallback).
- No durable call-state replication; SIP retransmit + SST == failover.

### Concurrency
- Default to **OS threads**; use async selectively where high fan-out I/O shows benefit (e.g., SIPREC ingest).

### Observability contract
- Every FE **must** expose `/metrics` in Prometheus text format.
- Structured JSON logs with `call-id`, `incident-id`, `agent-id`; PII tagging & redaction rules.
- Tracing: W3C tracecontext (HTTP) + SIP Call-ID correlation.

### CI/quality gates (enforced)
- **Forbidden:** `localhost`/`127.0.0.1` in code/config (use service DNS).
- **Required:** `/metrics` endpoint; OpenAPI server+client generated from same YAML; `ref` and `prod` builds/tests pass.
- Every bug fix adds a regression test.

---

## 9. AI Uncertainty Rules

If AI is unsure:
- Do not guess.
- Insert one of the following:

// TODO: clarify exact behavior with human reviewer.
// FIXME: unclear per STA-010 – needs decision.
// NOTE: assumption based on RFC 3261; verify for NG911.

---

## 9. Out-of-Scope for AI (RING911 Side)

- No Kubernetes manifests, Terraform, or Ansible here.
- No TLS certificate provisioning code in core or reference servers.
- No HA clustering or Patroni logic in core crates.
- No direct modifications to CONG test suite in this file.

---



## 10. Summary

AI may generate:
- Core logic in Rust for FEs.
- Trait implementations, state machines, validators, and parsers.
- Unit tests and documentation.

AI must not:
- Mix production concerns into core logic.
- Hardcode database, TLS, or SIP stack details outside adapters.
- Invent protocol behavior that doesn't exist in STA-010 or RFCs.

All AI-generated code must pass human review before merge.

