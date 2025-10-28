# SOLO Project Definition (Stabilized Draft)

> **Status:** Stabilized for external review.  
> **Companion doc:** `profiles_layering.md` (Profiles & Layering).

---

## Table of Contents
- [Part I – Vision & Scope](#part-i--vision--scope)
  - [Introduction](#introduction)
  - [Objectives](#objectives)
  - [Guiding Principles](#guiding-principles)
  - [Context & Motivation](#context--motivation)
  - [Deployment Reality](#deployment-reality)
  - [Order of Implementation](#order-of-implementation)
- [Part II – Architecture (Functional Elements & Platform)](#part-ii--architecture-functional-elements--platform)
  - [Functional Elements (Examples)](#functional-elements-examples)
  - [GeoCore + Query Services](#geocore--query-services)
  - [Call State Model (ESRP/BCF)](#call-state-model-esrpbcf)
  - [Stateless Dialog Routing via Signed State Token (SST)](#stateless-dialog-routing-via-signed-state-token-sst)
  - [Shared Platform](#shared-platform)
  - [Media Path Architecture](#media-path-architecture)
  - [SIP Stack Boundary & FFI Policy (reSIProcate ↔ Rust)](#sip-stack-boundary--ffi-policy-resiprocate--rust)
  - [HTTP Stack Standardization](#http-stack-standardization)
- [Part III – Operations & Infrastructure](#part-iii--operations--infrastructure)
  - [1. High Availability Model](#1-high-availability-model)
  - [2. GeoCore (SDPI-Based Replication)](#2-geocore-sdpi-based-replication)
  - [3. ESRP / BCF – Stateless Call Handling](#3-esrp--bcf--stateless-call-handling)
  - [4. Policy Store](#4-policy-store)
  - [5. Logging Service (LS)](#5-logging-service-ls)
  - [6. Configuration Management (GitOps)](#6-configuration-management-gitops)
  - [7. Networking & DNS](#7-networking--dns)
  - [8. Monitoring & Observability](#8-monitoring--observability)
  - [9. Security Architecture](#9-security-architecture)
- [Part IV – CONG (Conformance Suite)](#part-iv--cong-conformance-suite)
- [Part V – Engineering Practices](#part-v--engineering-practices)
  - [Code Quality & CI/CD](#code-quality--cicd)
  - [Release & Interop Position](#release--interop-position)
  - [Versioning & Compatibility Policy](#versioning--compatibility-policy)
  - [C/C++ Interop (SIP stacks)](#cc-interop-sip-stacks)
  - [Developer & AI Agent Guidance](#developer--ai-agent-guidance)
- [Part VI – Governance & Commercial Model](#part-vi--governance--commercial-model)
- [Part VII – Appendices](#part-vii--appendices)
  - [Appendix A – Decision Log](#appendix-a--decision-log)
  - [Appendix B – Implementation Order](#appendix-b--implementation-order)
  - [Appendix C – Glossary](#appendix-c--glossary)
  - [Appendix D – Deferred Elaborations](#appendix-d--deferred-elaborations)

---

## Part I – Vision & Scope

*This part defines why the project exists, what success looks like, and the context in which it will be deployed.*

### Introduction
The SOLO project will deliver a complete reference implementation of the Next Generation 9-1-1 (NG9-1-1) Core Services (NGCS), as defined in NENA STA-010.3f-2021 (“i3 Standard”). The implementation will be open-source, written in Rust, and designed for high reliability, security, and standards compliance. Alongside the reference system, SOLO will provide a comprehensive conformance test suite to validate interoperability and adherence to standards.

### Objectives
- Deliver a modular, high-reliability NGCS implementation.
- Ensure 100% coverage of normative STA-010 requirements (“shall” statements).
- Provide a conformance test suite that any NGCS vendor or operator can run to validate compliance.
- Support interoperability and substitution at the Functional Element (FE) level.
- Build trust in open-source NG9-1-1 through transparency, rigorous engineering, and operational discipline.

### Guiding Principles
1. **Conformance First** – development driven by STA-010 normative requirements.
2. **Separation of Concerns** – each Functional Element (FE) is independently deployable and replaceable.
3. **Operational Rigor** – security, observability, and high availability from the beginning.
4. **Open Governance** – code, tests, and specifications developed publicly under permissive licenses.
5. **Traceability** – every implementation and test maps to specific STA-010 sections or YAML interface definitions.
6. **Completeness with Prioritization** – required features first, then optional, with final completeness = all options implemented.

### Context & Motivation
- Existing NG9-1-1 vendors selectively implement features and frequently ignore mandatory (“MUST”) requirements of STA-010.
- RING’s core differentiator is **complete adherence to 100% of normative requirements**, demonstrated by the CONG conformance suite.
- This enables immediate credibility: "actually compliant, not just claiming compliance."

### Deployment Reality
- By the time RING is production-ready, most U.S. jurisdictions will already have deployed some form of NG9-1-1—not legacy E9-1-1.
- Therefore, RING must support **NG9-1-1 to NG9-1-1 migration**, not just greenfield deployments.
- Coexistence and phased replacement of existing vendor systems are first-order design goals.

### Order of Implementation
**Phase 1**: ECRF/LVF, ESRP, Logging Service, Policy Store  
**Phase 2**: BCF, MCS, LIS, ADR, SDM, GCS  
**Phase 3**: S/AL, OCIF, TCR/TCS/TCH, IDX, MDS  

---

## Part II – Architecture (Functional Elements & Platform)

*This part describes how the system is structured: the functional elements, shared platforms, and the key mechanisms that make the system resilient and interoperable. See the companion `profiles_layering.md` for a detailed view of reference vs production profiles.*

### Functional Elements (Examples)
- **ESRP (Emergency Services Routing Proxy):** SIP routing, minimal call state.
- **ECRF (Emergency Call Routing Function):** Geospatial queries, PostGIS-backed.
- **LVF (Location Validation Function):** Verifies civic address validity.
- **PRF (Policy Routing Function):** Applies XACML-based routing policies.
- **BCF (Border Control Function):** SIP firewall + overload controls.
- **Logging Service:** STA-010-compliant log store tied to incidents + internal telemetry.
- **MCS:** Conference/bridge service.
- **SDM:** Ingest + provision GIS data to ECRF/LVF/GCS.
- **GCS (Geocode Service):** Civic ↔ geospatial conversion.

### GeoCore + Query Services
RING standardizes all geospatial functions on a shared **GeoCore**:

- **GeoCore (shared platform)**: Postgres/PostGIS (single-node per subscriber) ingested **exclusively via SDPI**; multiple independent subscribers per site. Provides the provisioning endpoint (STA-010 SDPI), schema management, validation, audit logs, RBAC, quotas, and monitoring. GeoCore owns all authoritative geospatial data lifecycle (ingest, validate, publish, rollback).
- **Query Services (thin)**: ECRF, LVF, GCS, MCS, and MDS are stateless/lightweight services that execute read queries over GeoCore and return typed results per their OpenAPI contracts. MDS adds additional “layers” but consumes the same provisioned data.

**Why this split**
- One set of ingest/validation/HA/backup concerns rather than five.
- Flexible deployment: customers may run a single (replicated) GeoCore with all query services pointing to it, or multiple GeoCores by domain (e.g., ECRF-only vs “everything-else”).

**Provisioning path**
- SDM writes to GeoCore via the standardized Spatial Data Provisioning Interface (subscribe/notify CRUD, bulk/partial, planned-change effective-dated operations).
- GeoCore validates topology, civic hierarchy, ranges, and cross-layer consistency, then **publishes** a versioned dataset for readers.

**Query path**
- ECRF: georoute lookups (LoST queries over civic/geo inputs).
- LVF: civic normalization/validation queries.
- GCS: civic ↔ geocode transformations.
- MCS: map/coverage/area queries.
- MDS: layer access (same base data + extra layers).

**Interface contracts**
- Provisioning: SDPI (standardized).
- Query services: per-FE OpenAPI contracts; read-only; consistent error model; stable pagination/streaming for large results.

**Security**
- mTLS with PCA for all interfaces.
- Role-based access (role OID) and XACML policies at the interface.
- Read-only credentials for query services; publish-time WRITE-only for SDM; admin roles for schema/migration.

**SLOs (initial)**
- Publish latency (vetted → published): P95 ≤ 5 min; planned changes at scheduled epoch.
- Read latency (P99): ECRF/LVF ≤ 100 ms; GCS/MCS/MDS ≤ 250 ms per query (region-local).
- Cross-site read penalty budgeted separately (≤ 2× local P99).

### Call State Model (ESRP/BCF)
- Call state is **ephemeral only** (dialogs/transactions); no call history is kept in ESRP/BCF.
- **No durable replication is used** for call state; dialog routing is stateless via the Signed State Token (SST) described below.
- No consensus/raft/DB/Redis for call state; SIP retransmission rules cover transient loss.

### Stateless Dialog Routing via Signed State Token (SST)

**Goal**
- Make ESRP and BCF effectively stateless across failures (only transient transaction state), while ensuring any in-dialog request can be routed identically by any surviving instance.

**Primary mechanism (B2BUA mode)**
- On the upstream dialog we control, ESRP/BCF acts as a **B2BUA** and sets the **`To` tag** in the final response (e.g., 200 OK) to include a compact **Signed State Token (SST)**.
- The UAC will **echo this `To` tag** on all in-dialog requests (PRACK/UPDATE/re-INVITE/BYE/INFO). Any instance receiving the request reads the SST from the `To` tag and reconstructs routing without stored call state.

**SST contents (compact, no PII)**
- Payload fields (canonical order, minimal width):
  - `v`  (format version, 1 byte)
  - `k`  (key id for HMAC rotation, 1–2 bytes)
  - `iat` / `exp` (issue/expiry; short dialog-lifetime TTL, 4 bytes each)
  - `cid` (compact Call-ID hash, e.g., 64 bits)
  - `ftag` (From-tag hash, e.g., 32–64 bits)
  - `tgt` (routing target id, e.g., 64–128-bit stable id for downstream cluster/PSAP)
  - `ph`  (policy hash/epoch id, e.g., 32–64 bits; optional)
  - optional `site`/`az` (few bits) if needed for multi-site hints
- Encoding: **base64url(CBOR(payload) || HMAC-SHA256(payload, key[k]))**
- **Size budget:** target **≤ 160–200 bytes** total, safely under typical **~256-byte tag tolerance**.

**Validation & crypto**
- Verify HMAC using key `k` (accept both current and previous keys during rotation).
- Check `exp`, and that `cid`/`ftag` match the dialog identifiers of the received request.
- Optional: verify `tgt` is authorized by current XACML policy (we do not re-decide mid-dialog; `ph` is informational).

**Proxy-mode fallback (when we do not own the `To` tag)**
- If ESRP/BCF operates strictly as a proxy and cannot set the upstream `To` tag, attach the same SST as a **parameter on our Record-Route URI** (e.g., `;sst=`) with the same size budget.
- We prefer the To-tag method when available; the RR-param fallback must stay **short** and standards-compliant to minimize interop risks.

**Behavior notes**
- **Initial INVITE:** compute route (ECRF/PRF), build SST, then:
  - B2BUA mode: set SST-bearing `To` tag on upstream 2xx.
  - Proxy mode: add Record-Route with `sst=...`.
- **Subsequent in-dialog requests:** parse SST (from `To` tag or `Route`), validate, and forward to `tgt`. No durable per-call state is required.
- **Forking:** each early dialog leg gets its own token (if routes differ).
- **Failures:** any anycast instance with the HMAC keys can continue the dialog; loss of transient transaction state is recoverable via SIP retransmission rules.

**Operations**
- Maintain **rotating HMAC keys** (kid `k`) with overlap; export metrics for token validation failures and expirations.
- Keep tokens **PII-free** and under the size budget; use short, stable ids (not full URIs) for `tgt`.

**Standards compliance**
- Uses standard SIP dialog mechanics:
  - `To` **tag** echo by UAC (B2BUA case).
  - `Record-Route` / `Route` (proxy fallback).
- No proprietary headers required; token is opaque to peers.

### Shared Platform
- Mutual TLS with PCA PKI.
- Config, retries, backoff, circuit-breakers.
- Structured logs, metrics, distributed tracing.
- Uniform internal/external interface (OpenAPI).
- **Repository trait pattern** to prevent Postgres lock-in:
  - Core defines `Repository` trait.
  - `repo-mem` for reference, `repo-postgres` for production.
  - Business logic depends only on traits, never SQL.
- **Concurrency Model:** For expected NG911 workloads (low–moderate concurrency), default to native OS threads rather than async runtimes like Tokio. Threads offer lower overhead, simpler debugging, and adequate scalability for call volumes in NG911 systems. Async should be used selectively — only where high-volume parallel I/O (e.g., SIPREC media ingest, heavy SUBSCRIBE/NOTIFY fan-out) demonstrates clear benefit.

### Media Path Architecture

**BCF/SBC as media anchor**
- The BCF embeds an SBC that **anchors RTP** for ingress/egress calls.
- The SBC acts as a **SIPREC client** to all active Logging Service instances (media fan-out), keyed by MediaStartLogEvent.
- Normal case: **direct media** from caller ↔ PSAP; SBC stays in path only to enforce policy/interop, not to transcode by default.

**Inter-ESInet transfer (anchor release)**
- On transfer to another ESInet (e.g., misroute LA→NYC then transfer NYC→LA), the original SBC MUST **relinquish media anchoring** to avoid **tromboning**.
- Trigger: REFER/Replaces or Re-INVITE/UPDATE from ESRP/BCF indicating new remote target.
- Behavior: SBC updates SDP (ICE-less SIP/RTP) and **re-negotiate** media so that media flows directly between the new endpoints (or via the receiving ESInet’s SBC), then **tears down** its media relay for that dialog.
- Post-conditions: SIPREC recording continues from the **new** SBC(s) in the destination ESInet; the old SBC stops recording on final BYE or upon confirmed re-target.

**Bridge insertion**
- If a bridge is required (multi-party, barge-in, supervisor, transfer orchestration), we insert an **MCS Bridge**.
- The Bridge performs media mixing and policy-controlled fan-out using **GStreamer** pipelines (SIP/RTP only; no WebRTC).
- The Bridge also originates SIPREC to the Logging Service (same fan-out model).

**Conference control (XCON)**
- The Bridge implements **XCON** (RFC series) control primitives:
  - **Mute/unmute** and role changes (e.g., isolate caller while consulting).
  - **Participant visibility policy**: internal participants see true identities; external participants see **obfuscated identities** (e.g., “Call Taker”).
  - **Transfer choreography**: consultative transfer, attended/unattended, with deterministic media handoff.
- Control surface: SIP-centric (INFO/REFER) + HTTP API for NOC tools; actions are logged with participant and policy context.

**RTPEngine vs GStreamer**
- **RTPEngine**: purpose-built RTP relay/SDP mangling/NAT traversal; good for anchoring and re-INVITE choreography; lightweight; proven in SBC roles.
- **GStreamer**: powerful media graph/mixing/transcoding, needed for Bridge/XCON features.
- **Decision**: Use RTPEngine (or SBC-native media relay) for **BCF anchoring**, and GStreamer for **Bridge/MCS** media mixing. Avoid FS/Asterisk/Janus (not SIP-native conference control we need; Janus is WebRTC-centric).

**Security & observability**
- All media legs are SRTP (where peer supports) per policy; SBC enforces cipher suites; Bridge does not downgrade security.
- Export media QoS metrics (jitter, loss, RTT) and session lifecycle; correlate with Call-ID and MediaStartLogEvent.

### SIP Stack Boundary & FFI Policy (reSIProcate ↔ Rust)

**Scope & split of responsibilities**
- **reSIProcate (C++):** SIP protocol machinery — parsing/serialization, transactions, dialogs, transport (UDP/TCP/TLS), SIP timers, SUBSCRIBE/NOTIFY, and SIPREC signaling.
- **Rust (RING):** Business logic — routing decisions (ESRP/PRF), GeoCore queries (ECRF/LVF/GCS/MCS/MDS), policy evaluation (XACML), logging, metrics, and operational concerns.

**Design guidance**
- **Pragmatism over purity:** Prefer developer productivity and correctness to micro-optimizing the boundary. 911 call volumes make FFI overhead **negligible** and effectively unmeasurable.
- **Boundary emerges with code:** Do not freeze an artificial line up front. Let clarity, testability, and safety drive where logic sits as FEs mature.
- **SUBSCRIBE/NOTIFY everywhere:** Because STA-010 relies heavily on SIP events (e.g., ServiceState, replication triggers), **all FEs include reSIProcate** for uniform event handling, even when their primary API is HTTP.

**FFI safety & shape**
- Wrap all reSIProcate touchpoints in **safe Rust adapters** (`ffi_resip` crate). No raw pointers or C++ types leak into business logic.
- The adapter exposes **normalized events/requests** (e.g., `IncomingInvite`, `NotifyEvent`, `SiprecSession`) and **command intents** (e.g., `send_response`, `proxy_request`, `subscribe`).
- Unit tests target the Rust layer; protocol conformance is validated by CONG and targeted SIP tests.

**Operational notes**
- mTLS is terminated in the SIP stack; identity/role is surfaced to Rust for authorization.
- Timer tuning, transport quirks, and overload behavior remain in reSIProcate, with configuration governed via GitOps and per-FE defaults.

### HTTP Stack Standardization

All Functional Elements (FEs) expose and consume REST APIs generated from OpenAPI YAML. We standardize both the in-process HTTP implementation and, in production, a uniform data-plane to ensure consistent mTLS, keep-alives, timeouts, retries, and observability.

**Reference profile (ref):**
- Server: axum (hyper) with rustls; mutual TLS using PCA certificates; custom client-cert verifier checks PCA chain, SAN/service name, and role OID.
- Client: reqwest (hyper) with rustls via a shared client builder. Standard behaviors applied uniformly: connect/request timeouts, pool sizing, idle timeouts, TCP keepalive, HTTP/2 preference, correlation headers, metrics, tracing.
- Contracts: server stubs and client SDKs generated from the same OpenAPI YAML to prevent drift.

Production profile (prod):
- Data-plane proxy: **HAProxy** runs per host as a sidecar or local proxy.
  - Ingress: terminates mTLS, verifies PCA chain and role OID, forwards client-cert (base64 DER) to the FE on loopback.
  - Egress: FE speaks clear-HTTP to local HAProxy; HAProxy initiates mTLS to peer proxies.
  - Uniform timeouts, retries with jittered backoff, outlier detection (health checks), access logs, Prometheus metrics via haproxy_exporter.
- Envoy remains an **optional** alternative profile for environments that require native OTLP/gRPC filters or mesh-style L7 features.
- Services still use the same axum/reqwest code and OpenAPI bindings; production behavior is achieved by composition, not by changing business logic.

**Operational defaults (overridable per route/FE):**
Operational defaults (enforced in HAProxy and mirrored in client builders; overridable per route/FE):
- connect_timeout: 250 ms intra-site; 1 s cross-site
- request_timeout: 2 s typical; up to 10 s for bulk GIS routes
- idle_timeout: 30 s; max idle per host 2–4
- TCP keepalive: idle 10 s, interval 5 s, probes 3
- HTTP/2 keepalive ping: every 15 s; 3 misses ⇒ reconnect
- retries: 2 attempts with 50–200 ms jittered backoff; per-try timeout 750 ms
- hedged requests (idempotent reads only): second attempt after +200 ms to a different instance

**mTLS identity and authorization:**
- Each FE has a PCA-issued cert with role OID and SAN. Server side enforces PCA pinning, SAN/service checks, and role-to-verb authorization per the interface’s XACML policy.
- When HAProxy is in front, it verifies the PCA chain and passes the client cert to the FE (e.g., X-Client-Cert: base64 DER); the FE enforces role/OID authorization per XACML.
- Certificate rotation supports long overlap and early warnings (export `certExpiringSoon` metric); transition to ACME-like automation when the PCA offers it.

**Implementation note:**
- A shared crate (`ring-net`) supplies the standard axum server builder, reqwest client builder, mTLS verifier, correlation headers, and HTTP metrics. All FEs must consume these builders to guarantee uniform behavior.

---
## Part III – Operations & Infrastructure

*This part covers how we deploy, operate, observe, and secure the system in production.*

### 1. High Availability Model

SOLO assumes:  
- No single point of failure.  
- Multiple geographic sites per ESInet.  
- Each site contains multiple instances (active-active) of critical FEs.  
- DNS Anycast is the primary distribution and failover mechanism.  

Availability is achieved differently depending on the type of state involved:

| Component Type | HA / Replication Strategy |
|----------------|----------------------------|
| GeoCore (ECRF/LVF/GCS/MCS/MDS) | **SDPI-only replication** (snapshot + transactionId catch-up); no Patroni/etcd |
| ESRP / BCF (call routing) | **Stateless SIP** via Signed State Token (SST) in dialog; no call-state replication |
| Policy Store | **Replication is via the standardized STA-010 policy mechanism; each site stores policies locally. No database-level replication or consensus is used.** |
| Logging Service | **LogEvent replication follows STA-010 mechanisms. Media is replicated by configuring SIPREC to deliver recordings to all active loggers. No database replication or Patroni is used.** |
| Other small stores (ADR, LIS, S/AL) | Single-node Postgres or SQLite + backups unless elevated SLOs require DB HA |

Rolling upgrades are done during maintenance windows using a DNS TTL cutover pattern:
1. Lower TTL.
2. Start new instances with new software.
3. Shift DNS.
4. Drain old instances and decommission.

---

### 2. GeoCore (SDPI-Based Replication)

The GeoCore underlies ECRF, LVF, GCS, MCS, and MDS. It stores only data provisioned by SDM via the standardized **Spatial Data Provisioning Interface (SDPI)**.

**Key properties:**
- Source of truth is SDM; GeoCore never accepts direct writes.
- Each GeoCore instance runs standalone PostGIS; HA is provided by SDPI snapshot + transaction replay.
- Replication uses:
  - **Snapshot + startingTransactionId**, then
  - **Monotonic transactionId updates** over SDPI notify (HTTP webhook with GeoJSON payload).
- If a node fails, it is rebuilt from snapshot + SDPI backlog — no consensus or streaming replication is required.

**Why no Patroni/etcd:**
- SDPI is authoritative and ordered.
- Data is append-only with transaction history.
- Rebuild does not block other nodes; read availability is preserved by multiple subscribers.

**Required operational guarantees:**
- SDM must retain backlog for at least the time needed to rehydrate a GeoCore node.
- Snapshots should be published regularly (e.g., daily).
- Each GeoCore tracks metrics:
  - `lastAppliedTransactionId`
  - `sdpiLag_s`
  - `sdpiGapDetected_total`
  - `geoDatasetVersion`

---

### 3. ESRP / BCF – Stateless Call Handling

ESRP and BCF do not maintain durable or shared call state. Instead, SIP dialogs encode necessary context in SIP headers.

**Mechanism: Signed State Token (SST):**
- A compact (≤256B) signed blob included in the SIP To-tag (preferred) or a Record-Route parameter.
- Contains routing decisions and minimal call context.
- Cryptographically signed (HMAC or detached JWS) to prevent tampering.
- On failover, surviving nodes reconstruct dialog state from SIP signaling + SST.
- No database or consensus-based replication is used for call state.

**Failure handling:**
- If an ESRP dies mid-dialog, the endpoint re-INVITE/UPDATE arrives at another node (via DNS).
- The SST + Call-ID allow full state recovery or appropriate error.

---

### 4. Policy Store

- Replication is via the standardized STA-010 policy mechanism; each site stores policies locally. No database-level replication or consensus is used.

---

### 5. Logging Service (LS)

- LogEvent replication follows STA-010 mechanisms. Media is replicated by configuring SIPREC to deliver recordings to all active loggers. No database replication or Patroni is used.

---

### 6. Configuration Management (GitOps)

- All configuration (per-FE, deployment, security, routing) is stored as code in Git.
- Change control: PR + review + signed commits.
- Deployment via Ansible to bare-metal, VM, or cloud instances.
- Secrets managed with `sops + age`; never stored in plaintext.
- Environment overlays: `dev`, `qa`, `staging`, `prod`.
- Drift detection via periodic compare + alert.

---

### 7. Networking & DNS

**External DNS:**
- Cloudflare (primary) and Route 53 (backup) are examples only.
- Any provider must support:
  - DNSSEC,
  - Anycast/global routing,
  - Large-scale DDoS absorption,
  - API-driven zone/file updates.

**Internal DNS:**
- Bind with DNSSEC.
- Split-horizon optional but expected for ESInet.

**Routing:**
- **Routing:** Anycast with BGP.
- **Failure domains:** Each site is an independent failure domain.
- **Split-horizon DNS:** Optional; used to keep internal service addresses private.

---

### 8. Monitoring & Observability

**Metrics:**
- Every FE exposes `/metrics` in Prometheus text format.
- Collection can be pull (Prometheus) or push (Grafana Agent/OpenTelemetry Collector).
- Core metrics:
  - `callAttempt_total`, `callRouted_total`
  - `decisionLatency_ms`, `geocodeLatency_ms`
  - `sdpiLag_s`, `certExpiringSoon`
  - Media: `rtpJitter_ms`, `packetLoss_percent`

**Logs:**
- Structured (JSON) with call-id, incident-id, agent-id.
- PII tagging rules for redaction/export.

**Tracing:**
- W3C TraceContext for HTTP.
- SIP correlation via Call-ID + UUID in `Call-Info` or proprietary header.

**Active monitoring:**
- Automated Test Calls (STA-010.3.1, RFC 6881).
- Media quality validation via loopback at PSAP test clients.

**NOC/SOC integration:**
- Single observability pipeline for metrics/logs/traces + security alerts.
- Supports detection of DDoS, SIP fuzzing, phished endpoint behavior, etc.

---

### 9. Security Architecture

**Key Assumptions**
- No implicit perimeter trust: assume compromised credentials, malicious insiders, or supply chain breaches.
- Every Functional Element (FE) must defend itself; firewalls and private networks are not sufficient.
- All traffic between FEs uses mutual TLS (mTLS) with certificates issued by the PCA, including role/identity encoded via certificate OIDs.

**Core Mechanisms**
| Layer | Enforcement |
|-------|-------------|
| Authentication | mTLS with PCA-issued certificates; no anonymous or plaintext connections. |
| Authorization | Role/OID from certificate + XACML policies per external interface. |
| Input Defense | Strict schema validation; unknown or malformed fields cause rejection. |
| System Hardening | Minimal OS image, systemd sandboxing, SELinux/AppArmor, no SSH except via cert-based auth. |
| Monitoring | Security logs sent off-host; tamper-evident storage; alerts for anomalies. |
| Rate Limiting | Per-certificate quotas to limit data exfiltration or malicious querying. |

**Insider/Host Compromise Strategy**
- If an FE host is taken over, the attacker cannot erase or modify logs—logs stream to external immutable stores.
- Certificates remain short-lived; revocation lists propagated through PCA.
- High-risk actions (e.g., replacing GIS dataset) require dual approval (“four eyes” principle).

**DDoS/TDoS & Call Abuse**
| Threat | Mitigation |
|--------|------------|
| Volumetric DDoS | ESInet advertises BGP prefixes via scrubbing providers (Cloudflare, Akamai). |
| TDoS (SIP floods) | SIP overload control, dynamic per-source blocking, STIR/SHAKEN attestation checks. |
| SWATTING/fraud calls | Bad Actor lists from PSAPs; suspicious caller ID flagged via STIR/SHAKEN failure. |

**Supply Chain Controls**
- Reproducible builds; artifacts signed via Sigstore/cosign.
- SBOM generated for each build.
- Dependencies must be version-pinned and checksum-verified.
- CI restricted: forked PRs run in unprivileged runners; no secrets exposed.

---

## Part IV – CONG (Conformance Suite)

*This part explains how CONG drives development, resolves ambiguity, and proves compliance.*

**Purpose and posture.** CONG is first and foremost a **development and consensus-building tool**, not a vendor-policing gate. We use CONG to (a) drive RING implementation by turning STA-010 “shall” requirements into executable assertions, and (b) surface ambiguities or conflicts in the standard so they can be resolved in the NENA working group.

**Tight loop with RING.** The workflow is iterative by design:
1) Write a CONG test for a specific STA-010 behavior (with both **strict** and **intended** profiles where needed).  
2) Implement/adjust RING so the test passes.  
3) Rinse and repeat across all interfaces and edge cases.

**Standards feedback.** When a failure or ambiguity appears—whether in RING or another implementation—CONG becomes the **artifact that anchors discussion**. Findings (and any interim assumptions) are documented and brought to NENA i3 for clarification or errata. The project chairing role ensures that legitimate spec gaps can be corrected and fed back into CONG and RING coherently.

**Release policy (clarity, not policing).** RING **must pass the CONG “strict” profile** before release to ensure internal consistency with the standard as interpreted by the project. This is not about excluding others; it’s about guaranteeing that the reference implementation and the conformance suite reflect the **same, transparent interpretation**—and evolving both together when the standard is clarified.

---

## Part V – Engineering Practices

*This part describes how we ensure quality, repeatability, and safe change.*

### Code Quality & CI/CD
- CONG passing is a pre-release gate.
- All bug fixes must include regression tests.
- Merge queue: protected main, PRs rebased + tested serially.
- Flaky test budget: ≤0.5% allowed; zero tolerance in Phase 1 FEs.

### Release & Interop Position
- **RING release gate:** Passing the CONG **strict** profile is required for every release to ensure the reference implementation and its tests remain aligned.  
- **Spec assumptions log:** Any test that relies on an interpretation beyond explicit MUST/SHALL language records its assumption. If NENA clarifies or changes the text, CONG and RING are updated together; the assumption log is revised to match the ratified language.
- **Consensus-building with vendors:** We will run CONG with willing vendors/operators. A failure triggers **analysis, not blame**: is it an implementation defect, or an ambiguity in STA-010? Outcomes may include a vendor fix, a RING/CONG update, and/or a proposed standards clarification.
- **Two profiles always:**  
  - **strict** = purely normative requirements (MUST/SHALL).  
  - **intended** = behaviors widely implied or slated to become mandatory in 3.1; informative for vendors, binding for RING only when promoted to strict by the WG.

### Versioning & Compatibility Policy

**Relationship to STA-010**  
STA-010 uses a three-part version number written as **major.minor.patch** (example: `3.0.2`).  
SOLO artifacts (RING and CONG) add a **fourth number** for project iterations, forming **major.minor.patch.revision**.

**Examples**
| Component | Version Format        | Example        |
|-----------|------------------------|----------------|
| STA-010   | major.minor.patch     | 3.0.2          |
| RING911   | STA + .revision       | 3.0.2.4        |
| CONG911   | STA + .revision       | 3.0.2.2        |

Where:
- `3.0.2` is the STA-010 baseline.
- The last number is the RING/CONG revision for that baseline (counters are independent).

**When the revision increases**
- Bug fixes, security updates, performance/operational improvements.
- Implementation of optional behaviors permitted by the same STA baseline.
- These must **not** change the interpretation of STA-010.

**When the STA version changes**
- Only NENA changes **major.minor.patch**.
- `minor` must remain backward/forward compatible within a `major`.
- `patch` is used only for interoperability-affecting corrections.

**RING release rule**
- A RING release (e.g., `ring-3.0.2.4`) must pass all **CONG strict** tests for the same STA baseline.
- Ambiguities discovered are taken to the NENA i3 working group; upon clarification, both CONG and RING update in lockstep (same STA baseline, higher revision).

**CONG profiles**
- **strict**: exact MUST/SHALL language; required for RING release.
- **intended**: behaviors implied or slated for a future minor; informative and used for vendor feedback, not a release gate.

### C/C++ Interop (SIP stacks)
- RING uses reSIProcate (C++); CONG uses pjsip (C).
- All native code isolated in `ffi_*` crates with safe Rust wrappers.
- `bindgen`/`cbindgen`, reproducible builds, checksummed sources.
- Wrapper-level tests + fuzzing for parser boundaries.

### Developer & AI Agent Guidance
- `agents.md` defines prompts, coding style, test expectations.
- Core logic must not import production crates directly.
- Uniform crate layout enforced.

---

## Part VI – Governance & Commercial Model

*This part captures how the project will be guided and sustained.*

- **Governance today:** Benevolent-dictator model — led by the project initiator and STA-010 co-author.
- **Future direction:** Once stable, governance may transition to a neutral body such as NENA’s NIOC (NG9-1-1 Interoperability Coalition) or a foundation.
- **Conformance suite (CONG):** Licensed to vendors; enables sustainability without compromising open-source RING code.
- **Commercial strategy:** 
  - Paid vendor support and integration services under consideration.
  - No “freemium lock-in”; compliance and transparency are differentiators.
- **Long-term value proposition:** Vendors and PSAPs can no longer cherry-pick standards — RING/CONG defines the de facto enforcement of STA-010 behavior.

---

## Part VII – Appendices

### Appendix A – Decision Log
Source of truth for Linux distro, Postgres, DNS, tools, monitoring stack. (See also `profiles_layering.md` for profile-specific differences.)

### Appendix B – Implementation Order
Matches Phases 1–3.

### Appendix C – Glossary
| Term | Meaning |
|------|---------|
| SOLO | Overall project (reference + conformance) |
| RING911 | Reference implementation |
| CONG911 | Conformance suite |
| FE | Functional Element |
| SDPI | Spatial Data Provisioning Interface (standardized ingest/replication) |
| SST | Signed State Token (stateless SIP dialog routing) |
| PCA | Policy Certificate Authority (project root CA) |
| XCON | Conference control primitives used by Bridge/MCS |

### Appendix D – Deferred Elaborations
These topics are out of scope for this stabilized draft and will be elaborated later:
- STRIDE threat model per FE and per critical interaction.
- Security runbooks (NOC/SOC playbooks, incident response, forensics).
- Detailed SIPREC/media retention and retrieval semantics.
- Example crate wiring for `ref` vs `prod` profiles (full ESRP sample).
- End-to-end monitoring dashboards and SLO alert catalogs.