# 00 — Architecture Canon

**AI Career Agent (ACA) — Canonical Architecture Decision Record**

| | |
|---|---|
| Document ID | ACA-SAS-00 |
| Status | Authoritative — all chapters MUST conform |
| Audience | All chapter authors and reviewing engineers |
| Scope | Names, technology selections, conventions, invariants, SLOs |

This document is the **single source of truth** for the AI Career Agent Software Architecture Specification (SAS). Every chapter in Volumes 1–8 must use the names, technologies, conventions, and invariants defined here. A chapter may **extend** the canon (add detail) but must never **contradict** it. Where a chapter needs a decision the canon does not cover, the chapter records the decision explicitly in an "Alternatives Considered" section and it is promoted here on review.

---

## 1. Product Identity

- **Product name:** AI Career Agent (**ACA**)
- **Mission:** "Hire an AI that gets you a job."
- **Product definition:** ACA is an autonomous, multi-tenant AI operating system that continuously discovers jobs across the internet, evaluates fit against a deeply understood user profile, generates tailored application materials, executes browser-based application workflows where supported and permitted, tracks outcomes, prepares users for interviews and negotiation, and learns from every signal. The web application is only the control plane; the product is the distributed agent system behind it.
- **Tenant classes:** `individual`, `team`, `university`, `enterprise`, `recruiter`
- **Guardrail principle (non-negotiable):** ACA acts **on behalf of a consenting user, with the user's own credentials and identity, under explicit user-approved policies**. The platform enforces platform-compliance policies (site terms-of-service awareness, rate limiting, disclosure where required), human-approval workflows for consequential actions, and a kill switch at user, tenant, and platform scope.

## 2. Architecture Style (summary)

- Cloud-native **microservices** on Kubernetes; **event-driven** backbone on Kafka; **AI-native** control plane with a multi-agent system; **multi-tenant** with tenant isolation at data, compute, and cost layers; **zero-trust** security posture; **observability-first**; **vendor-agnostic** through abstraction seams (LLM providers, cloud provider, payment provider, vector store, graph store).
- Synchronous edge (REST/gRPC) for the control plane; asynchronous events for everything long-running. **No service calls another service's database.** Data is owned; integration is via APIs and events.
- Durable execution for long-running processes via **Temporal** (workflow engine). Sagas for cross-service consistency; the outbox pattern for atomic DB-write + event-publish.

## 3. Canonical Technology Stack

| Concern | Canonical choice | Notes / vendor-agnostic seam |
|---|---|---|
| Container platform | Kubernetes 1.30+ | Reference: AWS EKS; manifests are cloud-agnostic; GKE/AKS supported |
| IaC | Terraform + Terragrunt | Modules per environment; state in object storage w/ locking |
| Packaging | Docker (distroless / Chainguard base images), Helm charts (one umbrella + per-service charts) | |
| GitOps / CD | Argo CD + Argo Rollouts (canary, blue-green) | GitHub Actions for CI |
| Service mesh | Istio (mTLS, L7 authz, traffic policy) | SPIFFE/SPIRE workload identity |
| API gateway (edge) | Envoy Gateway (Gateway API) + custom ext-authz filter | Named **api-gateway** |
| Realtime edge | **realtime-gateway** service (WebSocket + SSE) backed by Redis pub/sub | |
| Sync inter-service | gRPC (internal), REST/JSON (external) | Protobuf registry (Buf) |
| Async backbone | Apache Kafka 3.x (KRaft) | Schema Registry (Confluent-compatible; Karapace reference) |
| Workflow engine | Temporal (self-hosted) | Named **workflow-service** wraps Temporal + worker fleets |
| OLTP database | PostgreSQL 16 (one logical DB per service; Aurora/CloudNativePG reference) | Row-Level Security for tenant isolation |
| Vector store | **pgvector** (phase 1) behind `embedding-store` interface | Scale path: Qdrant cluster (phase 2), same interface |
| Graph store | Apache AGE on PostgreSQL (phase 1) behind `graph-store` interface | Scale path: Neo4j (phase 2) |
| Cache / KV | Redis 7 (Cluster mode) | Exact cache, session state, rate limiting, semantic cache index metadata |
| Search engine | OpenSearch 2.x | Jobs index, application search, admin search |
| Object storage | S3-compatible (AWS S3 reference; MinIO for on-prem enterprise) | Resumes, artifacts, screenshots, DOM snapshots, model outputs |
| Secrets | HashiCorp Vault + External Secrets Operator; cloud KMS for envelope encryption | |
| Identity | **identity-service**: OAuth 2.1 + OIDC; reference implementation built on Ory (Kratos/Hydra); SAML + OIDC federation for enterprise SSO; WebAuthn/passkeys + TOTP for 2FA | |
| Authorization | RBAC + ABAC via OPA (Rego) sidecar/library; central **policy-service** for policy distribution | |
| Feature flags | OpenFeature SDK + Unleash (self-hosted) — **feature-flag-service** | |
| Observability | OpenTelemetry (traces/metrics/logs) → Prometheus/Mimir, Loki, Tempo; Grafana; Alertmanager → PagerDuty | |
| Frontend | Next.js 15 (App Router, RSC), TypeScript, Tailwind CSS, shadcn/ui, TanStack Query, Zustand | Deployed on Kubernetes (Node runtime), CDN in front |
| Mobile | React Native (Expo) sharing the public API | Volume 8 |
| Billing | Stripe behind **billing-service** provider adapter (Adyen/Paddle pluggable) | |
| Email/notifications | SES/SendGrid adapter, FCM/APNs push, Twilio SMS — all behind **notification-service** | |
| Browser automation | Playwright (Chromium primary) on ephemeral Kubernetes pods with gVisor isolation — **browser-fleet-service** | |
| LLM providers | Multi-provider behind **ai-gateway**: Anthropic Claude family (reference/primary), OpenAI, and self-hosted open-weight models (vLLM) | Model classes, not hardcoded vendors (§10) |
| Embeddings | Open-weight embedding models served via vLLM/TEI (bge-class) + provider embeddings as fallback, behind ai-gateway | Dimension-versioned (§10.5) |

**Languages:** TypeScript (NestJS) for product/domain microservices and frontend; Python (FastAPI) for AI-plane services (ai-gateway internals, agent runtime, learning-pipeline, portal-intelligence ML); Go permitted for high-throughput infrastructure services (proxy-manager, scheduler-service) where justified.

## 4. Canonical Service Catalog

Service names are **kebab-case, singular domain + `-service` suffix omitted in prose but implied**. The canonical identifiers below are used in diagrams, topics, and code.

### 4.1 Edge & Identity

| Service | Purpose (one line) | Owns data |
|---|---|---|
| `api-gateway` | Single external entry; routing, authn enforcement, rate limiting, WAF integration, API versioning | — (stateless) |
| `realtime-gateway` | WebSocket/SSE fan-out of events to clients (dashboard live updates, agent activity feed) | ephemeral connection state (Redis) |
| `identity-service` | Authentication, token issuance (JWT), OAuth2.1/OIDC, SSO federation, 2FA/passkeys, session management | identities, credentials, sessions, refresh tokens |
| `policy-service` | Central authz policy management + distribution (OPA bundles), consent & automation policy registry | policies, consents, approval rules |

### 4.2 Profile Domain

| Service | Purpose | Owns data |
|---|---|---|
| `user-service` | Account lifecycle, preferences, notification settings, GDPR requests, tenant membership | users, tenants, memberships |
| `profile-service` | Career profile: skills, experience, education, goals, AI profile understanding snapshots; LinkedIn/GitHub import orchestration | profiles, skills, imports |
| `resume-service` | Resume storage, parsing, versioning, tailoring artifacts, ATS-optimization scoring | resumes, resume versions, parse results |
| `portfolio-service` | Portfolio/projects management, GitHub repos analysis, portfolio site artifacts | portfolios, projects |

### 4.3 Job Domain

| Service | Purpose | Owns data |
|---|---|---|
| `job-intelligence-service` | Internet-wide job discovery, normalization, dedup, enrichment; company intelligence | jobs, companies, sources, crawl state |
| `matching-service` | Job↔profile scoring, ranking, match explanations | match scores, match features |
| `application-service` | Application lifecycle (draft→submitted→interview→offer), timeline, documents links, interview & negotiation records | applications, application events, interviews, offers |

### 4.4 Automation & AI Plane

| Service | Purpose | Owns data |
|---|---|---|
| `workflow-service` | Temporal cluster + workers; durable orchestration of long-running flows (autopilot, application submission saga) | workflow histories (Temporal store) |
| `agent-orchestrator` | Agent registry, lifecycle, scheduling, supervision, multi-agent collaboration, HITL checkpoints | agent definitions, runs, checkpoints |
| `ai-gateway` | Single LLM/embedding entry: model routing, prompt assembly, caching, safety filters, token accounting, provider failover | request logs, token ledger (hot) |
| `prompt-registry-service` | Versioned prompts, evaluation sets, rollout channels (dev/canary/stable) | prompts, prompt versions, eval results |
| `memory-service` | Agent memory tiers (session/short/long-term), conversation memory, memory consolidation jobs | memories, memory indices |
| `knowledge-graph-service` | Entity/relationship graph (users, companies, skills, outcomes) + graph queries for reasoning | graph store |
| `cost-controller` | Budgets (per tenant/user/agent/feature), spend ledger aggregation, throttle/degrade decisions | budgets, spend aggregates |
| `learning-pipeline` | Outcome ingestion, feature store, offline evaluation, model/prompt improvement loops | training sets, eval runs, feature store |
| `portal-intelligence-service` | Universal Portal Engine brain: portal fingerprints, templates, navigation graphs, semantic form models | portal fingerprints, templates, portal memory |
| `browser-fleet-service` | Browser pool scheduling/allocation, session lifecycle, screenshot service, health & recovery | browser sessions (ephemeral), fleet state |
| `session-vault-service` | Encrypted storage of user portal credentials, cookies, session artifacts (user-consented) | cookie/session vault (encrypted) |
| `proxy-manager` | Egress IP pool management, per-site rate budgets, geo routing, reputation health | proxy pool state |

### 4.5 Experience & Business

| Service | Purpose | Owns data |
|---|---|---|
| `notification-service` | Multi-channel notifications (email, push, SMS, in-app), digests, preferences enforcement | notifications, delivery receipts |
| `scheduler-service` | Cron/RRULE scheduling of platform jobs (scans, digests, re-crawls) publishing trigger events | schedules |
| `search-service` | Search API over OpenSearch indices (jobs, applications, admin) with tenant filters | search indices (projections) |
| `analytics-service` | Product + career analytics: funnels, response rates, benchmarks; CQRS read models | analytics projections (ClickHouse optional phase 2; OLAP schema in Postgres phase 1) |
| `billing-service` | Plans, subscriptions, usage metering ingestion, invoices, entitlements, dunning | subscriptions, invoices, entitlements, usage records |
| `admin-service` | Internal ops portal APIs: tenant management, support tools, abuse review, agent fleet ops | admin audit shadow, support cases |
| `recruiter-service` | Recruiter portal: candidate sourcing (consented), job posting intake, pipeline views | recruiter orgs, postings, pipelines |
| `audit-service` | Immutable audit log (hash-chained), compliance exports | audit events (append-only) |
| `public-api-service` | Public REST API + webhooks for integrations/SDK; API key management | api keys, webhook endpoints |

> Chapters may reference "the 30 services" — the catalog above is the authoritative list. New services require canon amendment.

## 5. Canonical Agent Catalog

Agents are **logical AI workers** executed by `agent-orchestrator` on the shared agent runtime; they are not separate microservices. Naming: `kebab-case` + `-agent`.

| Agent | Mission |
|---|---|
| `job-scout-agent` | Continuously discover new job postings across sources for a user's search intents |
| `job-parser-agent` | Normalize raw postings into the canonical Job schema (rules-first, LLM fallback) |
| `company-intel-agent` | Build/refresh company profiles: culture, funding, tech stack, hiring signals |
| `matching-agent` | Score and explain job↔profile fit; maintain user match calibration |
| `resume-agent` | Tailor resume versions per application; ATS optimization |
| `cover-letter-agent` | Generate tailored cover letters with evidence from profile + company intel |
| `portfolio-agent` | Optimize portfolio/project presentation for target roles |
| `browser-agent` | Execute portal workflows step-by-step via browser-fleet (perceive→plan→act→verify) |
| `portal-intel-agent` | Fingerprint portals, learn/update reusable templates, maintain navigation graphs |
| `application-tracker-agent` | Track application status via email parsing, portal checks, user signals |
| `interview-coach-agent` | Interview prep: question banks, mock interviews, feedback |
| `negotiation-agent` | Offer analysis, comp benchmarks, negotiation scripting |
| `calendar-agent` | Interview scheduling, reminders, calendar integration |
| `email-agent` | Draft/parse application-related email (user-consented mailbox integration) |
| `notification-agent` | Decide what/when/how to notify; digest composition |
| `analytics-agent` | Generate career insights narratives from analytics data |
| `learning-agent` | Mine outcomes for strategy improvements (per-user and global) |
| `policy-agent` | Pre-action compliance checks: site policy, user consent, approval requirements |
| `security-agent` | Prompt-injection screening, anomaly detection on agent behavior |
| `cost-optimization-agent` | Monitor spend patterns; recommend/enact routing + caching changes |
| `agent-supervisor` | Meta-agent: monitors agent runs, detects loops/stalls, escalates |
| `recovery-agent` | Diagnose and retry/repair failed workflows (esp. browser flows) |
| `scheduler-agent` | Plan per-user autopilot cadence within budgets |

## 6. Multi-Tenancy & Identity Conventions

- **IDs:** UUIDv7 everywhere (`id` primary keys; time-ordered). External references: prefixed opaque IDs (`usr_`, `ten_`, `job_`, `app_`, `agt_`, `run_`).
- **Tenancy model:** every row carries `tenant_id`; PostgreSQL **RLS enforced at the database** with `SET app.tenant_id`; service-layer guard as second check. Individual users are single-member tenants. Enterprise/university tenants may pin to dedicated **cells** (cell-based architecture, Volume 6) with isolated data planes.
- **Tenant tiers → plans:** `free`, `professional`, `premium`, `enterprise` (billing entitlements, §12).

## 7. API Conventions

- External REST under `https://api.aca.dev/v1/...`; versioning in path (`/v1`), additive-only within a version; breaking changes → `/v2` with 12-month deprecation window.
- Errors: RFC 9457 `application/problem+json` with `type`, `title`, `status`, `detail`, `trace_id`.
- Pagination: cursor-based (`?cursor=`, `?limit=`, default 25, max 100). Sorting explicit allowlist.
- Idempotency: all POSTs accept `Idempotency-Key` header; 24h dedup window (Redis).
- Rate limits: token-bucket per API key + per tenant at api-gateway; headers `RateLimit-Limit/Remaining/Reset`.
- AuthN: `Authorization: Bearer <JWT>` (user) or `X-API-Key` (public API service keys). JWT: EdDSA (Ed25519), 15-min access tokens, rotating refresh tokens (30-day, revocable), `kid`-based key rotation via JWKS.
- Internal gRPC: mTLS via mesh; per-method authz via OPA; deadlines mandatory (default 2s sync read, 5s sync write).

## 8. Event Architecture Conventions

- **Envelope:** CloudEvents 1.0 (JSON), `id` = UUIDv7, `source` = service name, `type` = event name, extensions: `tenantid`, `traceparent`, `schemaversion`.
- **Topic naming:** `aca.<domain>.<event>.v<major>` — e.g. `aca.job.discovered.v1`, `aca.application.submitted.v1`.
- **Canonical core topics** (chapters may add within their domain, following the convention):
  - `aca.user.registered.v1`, `aca.user.deleted.v1`
  - `aca.profile.updated.v1`, `aca.profile.import.completed.v1`
  - `aca.resume.parsed.v1`, `aca.resume.version.created.v1`
  - `aca.job.discovered.v1`, `aca.job.parsed.v1`, `aca.job.updated.v1`, `aca.job.expired.v1`
  - `aca.company.enriched.v1`
  - `aca.match.scored.v1`
  - `aca.application.drafted.v1`, `aca.application.approved.v1`, `aca.application.submitted.v1`, `aca.application.status.changed.v1`
  - `aca.interview.scheduled.v1`, `aca.offer.received.v1`
  - `aca.agent.run.started.v1`, `aca.agent.run.completed.v1`, `aca.agent.run.failed.v1`, `aca.agent.approval.requested.v1`
  - `aca.browser.session.started.v1`, `aca.browser.session.completed.v1`, `aca.browser.action.failed.v1`
  - `aca.ai.request.completed.v1` (token/cost accounting), `aca.cost.budget.threshold.v1`
  - `aca.billing.subscription.changed.v1`, `aca.billing.usage.recorded.v1`
  - `aca.notification.requested.v1`, `aca.notification.delivered.v1`
  - `aca.audit.event.v1` (mirrored to audit-service)
- **Delivery:** at-least-once; consumers idempotent (dedup on event `id` in Redis/DB). Ordering per key (`tenant_id` or entity id partition key).
- **Retries/DLQ:** consumer-side exponential backoff (3 tries) → `<topic>.retry.1m` → `.retry.10m` → `<topic>.dlq`; DLQ triaged via admin-service; replay tooling required.
- **Schema evolution:** JSON Schema in Schema Registry; BACKWARD-compatible within `v<major>`; new major = new topic.
- **Sagas:** orchestrated via Temporal (not choreography) for multi-service transactions (e.g., application submission). **Outbox pattern** (Debezium CDC) for DB→Kafka atomicity. Event sourcing only where justified: `application` timeline and `audit` log are event-sourced; others are state-stored + events.

## 9. AI Architecture Invariants

- **All model calls go through `ai-gateway`.** No service or agent calls a provider SDK directly.
- **Escalation ladder (cost §10):** deterministic rules → exact cache → semantic cache → knowledge/graph lookup → embeddings+kNN → small model → medium model → large model → frontier model. Skipping tiers requires an explicit, logged routing policy.
- **Agent loop (canonical):** Perceive → Retrieve (context builder) → Plan → Act (tool calls) → Verify → Reflect/Learn, with policy-agent checks before consequential actions and HITL checkpoints per user autonomy settings.
- **Autonomy levels (user-configurable, per action class):** `L0 suggest` (drafts only), `L1 approve-each` (HITL per action), `L2 approve-batch` (daily batch approval), `L3 autopilot` (auto within policy budget; consequential-action allowlist). Application submission defaults to L1.
- **Context builder:** assembles prompt = system (versioned, from prompt-registry) + tenant/user memory (memory-service) + task context (retrieval) + tool schemas; hard token budgets per model class; deterministic ordering for cache-friendliness (stable prefix first, volatile last).
- **Prompt management:** every prompt has `prompt_id@semver`, owned by prompt-registry-service; runtime pulls by channel (`stable`/`canary`); offline evals gate promotion; A/B experiments logged to learning-pipeline.
- **Tool calling:** JSON-Schema tool registry per agent; all tool calls policy-checked and audit-logged; tool results size-capped with overflow to object storage + reference.

## 10. Model Routing & Cost Canon

### 10.1 Model classes

| Class | Purpose | Reference models (seam: any provider) |
|---|---|---|
| `embed` | Embeddings | bge-class open-weight (self-hosted), provider fallback |
| `small` | Classification, extraction, routing, guardrails | Claude Haiku class |
| `medium` | Drafting, summarization, standard reasoning | Claude Sonnet class |
| `large` | Complex generation (tailored resumes/letters), agent planning | Claude Opus class |
| `frontier` | Hardest reasoning: novel portal workflows, negotiation strategy | Highest-capability available (e.g., Fable class); requires routing-policy flag |
| `vision` | Screenshot/DOM understanding | Vision-capable medium/large class |

### 10.2 Routing policy
- Router inputs: task type, complexity score (heuristic + small-model estimate), tenant plan, remaining budget, latency SLO, historical quality stats.
- **Budget controller integration:** cost-controller issues per-scope budgets (tenant/user/agent/feature, daily+monthly). ai-gateway enforces: soft threshold (80%) → degrade class one tier + notify; hard threshold (100%) → queue/deny non-critical, always allow user-interactive within plan floor.
- Batch lane: non-interactive work (bulk parsing, enrichment, evals) uses provider batch APIs (≈50% cost) with ≤24h SLA.
- Caching: exact cache (Redis, request-hash, TTL by task class) → semantic cache (pgvector cosine ≥ 0.97 task-scoped threshold, guarded by staleness policy) → response reuse with attribution. Provider-side prompt caching exploited via stable prefix design.

### 10.3 Token accounting
- Every ai-gateway call emits `aca.ai.request.completed.v1` with tokens in/out, cache hits, model class, provider, cost, tenant/user/agent/feature attribution. Aggregated by cost-controller (hot, Redis) and analytics-service (warm). Ledger is the billing metering source for AI usage.

### 10.4 Quality control
- Golden-set evals per prompt version (prompt-registry); online sampling with small-model judges; automatic rollback of prompt/model-routing changes on quality regression (learning-pipeline owns thresholds).

### 10.5 Embedding versioning
- Embedding space is versioned (`embed_v1`, ...). Index metadata records model+dimension. Re-embedding jobs run per-namespace with dual-read during migration.

## 11. Data Conventions

- Each service owns its schema; **no cross-service DB access**. Read models via events (CQRS) where needed.
- Migrations: per-service, forward-only (Flyway/Atlas), zero-downtime (expand → migrate → contract).
- PII classification levels: `P0` (credentials/session cookies — session-vault only, envelope-encrypted per user with per-tenant KEK), `P1` (identity PII — field-level AES-256-GCM encryption), `P2` (career data — encrypted at rest via storage encryption), `P3` (public/derived).
- Retention: raw crawl artifacts 30d; screenshots/DOM snapshots 90d (or per-tenant policy); AI request logs 90d hot → archived 13 months; audit events 7 years (WORM object storage); user deletion = hard delete + tombstone events within 30 days (GDPR).
- Backups: PITR for Postgres (5-min RPO), daily snapshots 35-day retention, quarterly restore drills; Kafka tiered storage 7-day hot retention (audit topics longer); object storage versioning + cross-region replication.
- Partitioning: time-based partitioning for high-volume tables (`job`, `application_event`, `ai_request_log`, `audit_event`); hash partitioning by `tenant_id` for very large tenants (cells).

## 12. Billing & Entitlements Canon

| Plan | Monthly price (reference) | Key entitlements |
|---|---|---|
| Free | $0 | 10 tracked applications/mo, manual apply assist, small/medium models only, 1 resume version/app |
| Professional | $29 | 100 applications/mo, autopilot L2, interview prep, large-model tailoring, analytics |
| Premium | $79 | 400 applications/mo, autopilot L3, negotiation agent, priority queue, frontier-class access (budgeted) |
| Enterprise / University / Recruiter | Custom | Seats, SSO/SAML, dedicated cell option, admin & API access, SLAs, DPA |

- Entitlement checks: billing-service issues signed entitlement snapshots; enforced at api-gateway (coarse) + services (fine) + cost-controller (AI budgets). Usage metering: `aca.billing.usage.recorded.v1` from cost-controller & application-service.

## 13. Security Canon

- Zero trust: no implicit network trust; every hop authenticated (mTLS/SPIFFE) and authorized (OPA). Secrets only from Vault; no secrets in env-committed files.
- Human identities: OIDC SSO + passkeys; admin access via short-lived, audited, JIT elevation (admin-service + policy-service).
- Prompt-injection defense (all LLM inputs from external content — job posts, portal DOM, emails): content sanitization, source tagging (`untrusted_content` wrappers), instruction/data separation, output validators, tool allowlists per agent, security-agent screening, no raw credentials ever in prompts.
- Browser isolation: each browser session in gVisor-sandboxed pod, per-session network policy (egress allowlist to target portal + proxy), no cross-user browser reuse, credentials injected via session-vault sidecar at egress, never into agent context.
- Compliance targets: SOC 2 Type II (year 1), GDPR/CCPA (day 1), ISO 27001 (year 2); WCAG 2.2 AA frontend.

## 14. Observability & SLO Canon

- OTel everywhere; `trace_id` propagated through events (`traceparent` extension) and workflows.
- **Golden SLOs:**

| SLI | SLO |
|---|---|
| Control-plane API availability | 99.95% monthly |
| API latency (read p99 / write p99) | < 300 ms / < 800 ms |
| Realtime event delivery p95 | < 2 s |
| Job discovered → scored (autopilot) p95 | < 15 min |
| Application draft generation p95 | < 60 s |
| Browser workflow success rate (known portals) | ≥ 92% weekly |
| AI answer quality (eval pass rate on stable prompts) | ≥ 95% |
| Notification delivery p95 | < 60 s |

- Error budgets gate releases (burn-rate alerts 2%/1h, 5%/6h). Business + AI + browser + cost metrics are first-class (chapters define per-domain dashboards).

## 15. Deployment & Environments Canon

- Environments: `dev` → `staging` → `prod`; prod is multi-region **active-passive** phase 1 (primary `us-east-1`-class, warm standby `eu-west-1`-class; EU data-resident tenants pinned to EU cell), active-active for stateless edge phase 2.
- DR: RPO ≤ 5 min (Postgres streaming + Kafka replication), RTO ≤ 30 min (runbooked, quarterly game-days).
- Progressive delivery: canary (Argo Rollouts, 5%→25%→100% with metric analysis) default; blue-green for stateful/riskier services; DB migrations decoupled from deploys.
- Cells: a **cell** = complete data-plane stack (services + Postgres + Redis + Kafka consumers) serving a tenant shard; shared control plane (identity, billing, admin). Enterprise tenants may get dedicated cells.

## 16. Universal Portal Engine Canon

- **Never hardcode per-site automation.** The engine generalizes: Portal Detection → Fingerprinting → Perception (DOM + accessibility tree + screenshot vision) → Semantic Form/Workflow Model → Action Planning → Validated Execution → Verification → Template Learning (reusable `PortalTemplate@version` stored in portal-intelligence-service).
- Template trust ladder: `discovered` → `candidate` (shadow-verified) → `trusted` (auto-usable) → `stale` (drift detected) — drift detection via fingerprint diffs and failure-rate monitors; auto re-learning.
- Known ATS families (Greenhouse, Lever, Ashby, Workday, Taleo, iCIMS, SuccessFactors) are just **high-confidence fingerprint families with mature templates**, not hardcoded adapters. Adapter SDK allows first/third parties to contribute template packs via plugin framework.
- Every consequential submission requires: policy-agent clearance + user autonomy level check + dry-run validation + post-submit verification (confirmation signal capture).

## 17. Cross-Cutting Documentation Rules (for authors)

- Chapter numbering/file layout fixed (see `architecture/README.md`).
- Required per-component sections: Purpose, Responsibilities, Interactions, Sequences, Failure Scenarios, Scaling Strategy, Security, Monitoring, Future Improvements, Alternatives Considered & Tradeoffs.
- Diagrams: Mermaid, following WRITER-BRIEF rules. Cross-references: relative links to sibling chapter files.
- Terminology: use canonical names exactly (`ai-gateway`, not "AI Gateway service" in identifiers; prose may use Title Case with identifiers in backticks).
