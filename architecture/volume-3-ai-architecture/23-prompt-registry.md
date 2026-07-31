# 23 — Prompt Registry

| | |
|---|---|
| Document ID | ACA-SAS-23 |
| Volume | 3 — AI Architecture |
| Status | Draft v1.0 |
| Depends on | [00 — Architecture Canon](../00-architecture-canon.md) §9–§10, §14; [18 — AI Gateway](../volume-2-core-services/18-ai-gateway.md) §5.3; [19 — Agent Orchestrator](../volume-2-core-services/19-agent-orchestrator.md) §10; [21 — AI Architecture](21-ai-architecture.md) §4, §7; [22 — Multi-Agent Framework](22-multi-agent-framework.md) §7, §10; [26 — Semantic Cache](26-semantic-cache.md); [28 — Model Routing](28-model-routing.md) |

## 1. Overview

Prompts are production code for the AI plane. Every behavior regression in [Chapter 21 §8](21-ai-architecture.md)'s failure taxonomy — hallucination, tool misuse, over-refusal, injection susceptibility — is reachable by editing a string, and no compiler catches it. `prompt-registry-service` exists so that no model-visible string ships without the discipline applied to any other deployable artifact: immutable versioning, review, staged rollout, automated regression gates, and one-command rollback. The canon makes this binding: every prompt has `prompt_id@semver`, runtime resolves by channel, offline evals gate promotion ([canon §9](../00-architecture-canon.md)).

This chapter specifies the service end to end: the artifact model (semver semantics, immutability, yanking), the template language and its typed variable contracts, the `dev`/`canary`/`stable` channel machinery and progressive rollout, the evaluation framework (golden sets, `small`-class LLM judges, human review queues), promotion gates and automatic rollback, the prompt↔model-class compatibility matrix, storage DDL, the admin and runtime APIs, integration sequences with `ai-gateway` and `learning-pipeline`, and the prompt security review pipeline (§12) whose adversarial library is shared with the agent replay harness ([Chapter 22 §10](22-multi-agent-framework.md)).

Ownership boundaries, stated once: the registry owns prompt *content, versions, channels, eval data, and compatibility facts*. It does not render prompts at request time (the context builder library renders from cached bundles — [Chapter 21 §4](21-ai-architecture.md)), does not enforce envelope integrity (that is `ai-gateway`'s hash-honesty rule R4, [Chapter 18 §5.3](../volume-2-core-services/18-ai-gateway.md)), and does not own online regression thresholds or the rollback *decision* (that authority is `learning-pipeline`'s, [canon §10.4](../00-architecture-canon.md) — the registry executes the flip). This separation keeps the registry off every request hot path: at steady state it serves bundle fetches and admin traffic, nothing else.

## 2. Component Architecture

```mermaid
flowchart TB
  subgraph pr["prompt-registry-service"]
    api["Admin API (REST via api-gateway)"]
    rpc["Runtime API (gRPC ResolveBundle)"]
    comp["Template compiler and linter"]
    store["Artifact store (Postgres)"]
    chan["Channel manager (pointers, weights, history)"]
    evalo["Eval orchestrator (suites, judges, review queue)"]
    dist["Bundle distributor (signing, invalidation)"]
    sec["Security review pipeline (injection lint, adversarial replay)"]
  end
  admin["admin-service ops portal"]
  rt["Agent runtime bundle caches"]
  gw["ai-gateway (batch lane for eval and judge calls)"]
  lp["learning-pipeline (online SLIs, rollback authority)"]
  rd["Redis (bundle cache, pub-sub invalidation)"]
  k["Kafka (aca.prompt domain events)"]
  admin --> api
  api --> comp
  comp --> store
  api --> chan
  chan --> store
  chan --> dist
  dist --> rd
  rt --> rpc
  rpc --> rd
  rpc --> store
  evalo --> gw
  evalo --> store
  sec --> evalo
  lp -->|"rollback and promotion signals"| api
  chan --> k
```

| Component | Purpose | Failure posture |
|---|---|---|
| Admin API | authoring, publish, channel moves, yank, suite management — all human/`learning-pipeline` mutations | fail-closed; every mutation audited |
| Runtime API | `ResolveBundle` for runtime caches; read-only | fail-static — consumers serve cached bundles (§10.2) |
| Template compiler + linter | parse to AST, validate variable contract, token estimation, static lint (incl. security lint §12.1) | fail-closed — uncompilable versions cannot be created |
| Artifact store | Postgres system of record for prompts, versions, pointers, eval data | PITR per canon §11 |
| Channel manager | pointer moves with gate enforcement; canary weights; pointer history | fail-closed; pointer flips are single-row transactions |
| Eval orchestrator | executes suites via `ai-gateway` batch lane; scores; feeds review queue | eval outage blocks *promotions*, never runtime |
| Bundle distributor | builds signed bundles, publishes invalidations on pointer change | delayed invalidation degrades to 60 s poll staleness |
| Security review pipeline | injection-resistance lint + adversarial replay + human review routing | fail-closed for publish (G0 gate, §7) |

**Interactions.** Writers: prompt owners (via `admin-service` UI and CI), `learning-pipeline` (rollback/promotion API calls). Readers: agent runtime pods and domain services through per-pod bundle caches; `ai-gateway` verifies envelope prompt-segment hashes against registry manifests ([Chapter 18 §5.3](../volume-2-core-services/18-ai-gateway.md) R4). All eval model calls go through `ai-gateway` on the batch lane ([canon §9](../00-architecture-canon.md) — no direct provider SDK use, even for evals).

## 3. Prompt as a Versioned Artifact

### 3.1 Artifact model

A **prompt** is a stable identity (`prompt_key`) owned by a team; a **prompt version** is an immutable artifact: template body + variable contract + metadata, addressed as `prompt_key@MAJOR.MINOR.PATCH`.

| Field | Content | Rationale |
|---|---|---|
| `prompt_key` | dot-path: `<agent-or-service>.<task>.<role>` — e.g. `browser-agent.planner.system`, `resume-agent.tailor.system`, `ai-gateway.injection-screen.system` | keys mirror the agent/task structure so ownership and eval-family mapping are mechanical |
| `task_family` | one of the eval families in [Chapter 21 §7](21-ai-architecture.md) (planning, generation, extraction, browser, coaching, guardrail) | selects default suites and judges |
| `body` | PromptTemplate v1 source (§4) | |
| `variable_contract` | JSON Schema with trust/PII/size extensions (§4.2) | the caller-facing API of the prompt |
| `content_hash` | SHA-256 over canonical body + contract | the integrity anchor verified by `ai-gateway` |
| `consequential` | bool — prompt drives consequential-action agents (browser submission, email send, negotiation) | triggers stricter gates (§7) and mandatory human security review (§12.3) |
| `changelog` | required, human-written | reviewers and incident responders read diffs *and* intent |

### 3.2 Semver semantics

Because any content change can change behavior, ACA's semver maps to *contract* impact, not "bug fix vs feature":

| Bump | Meaning | Examples | Consumer obligation |
|---|---|---|---|
| MAJOR | variable contract change (add/remove/retype required variable) or declared intent change | new required `company_intel` variable; extraction prompt becomes classification prompt | callers re-integrate; agent versions pinning `>=` ranges fail registration until updated ([Chapter 22 §7](22-multi-agent-framework.md)) |
| MINOR | behavior-affecting content change, contract-compatible | rewording instructions, adding few-shot examples, tightening output constraints | none; flows through channels with full eval gating |
| PATCH | no model-visible change: metadata, description, tags, changelog fixes | typo in description | none; exempt from eval gates (the only exemption, because the rendered bytes are unchanged — enforced by hash equality with the parent version) |

### 3.3 Immutability and lifecycle

Published versions are frozen: a database trigger rejects updates to `body`, `compiled_ast`, `variable_contract`, and `content_hash` once `status='published'` — only status transitions remain legal. There is no "edit in place"; every change is a new version. This is what makes the compatibility matrix (§8), eval results, and incident forensics trustworthy: an eval result refers to bytes that cannot have drifted.

```mermaid
stateDiagram-v2
  [*] --> draft
  draft --> published : lint_and_g0_security_pass
  draft --> discarded : abandoned
  published --> deprecated : superseded_on_stable
  published --> yanked : incident_or_security_finding
  deprecated --> yanked : incident_or_security_finding
  yanked --> [*]
```

- `draft` — editable, resolvable only by explicit version pin in `dev`-channel contexts.
- `published` — immutable, channel-eligible per §5.
- `deprecated` — no longer pointed to by `stable`; still resolvable by pin (running agent versions may reference it); excluded from new agent-version registration.
- `yanked` — never resolvable; agent versions referencing it fail registration ([Chapter 22 §7](22-multi-agent-framework.md) makes dangling references impossible); channel pointers referencing it at yank time are atomically flipped to the last non-yanked stable ancestor. Yanking requires an incident or security-review reference — it is an emergency tool, not cleanup.

## 4. Template Language & Variable Contracts

### 4.1 PromptTemplate v1 — deliberately logic-less

The template language is a restricted, logic-less dialect (Mustache-family). Anything resembling a programming language in prompt templates ends with business logic hiding in strings where no test framework can see it; the language is small enough to lint exhaustively.

| Construct | Syntax | Semantics | Bound |
|---|---|---|---|
| Interpolation | `{{var}}` | insert declared variable, rendered per its contract type | value ≤ variable `max_tokens` |
| Presence section | `{{#var}}...{{/var}}` | include block iff variable present/truthy | nesting ≤ 3 |
| Bounded iteration | `{{#each var limit=N}}...{{/each}}` | iterate list variable; `limit` mandatory | N ≤ 50 — unbounded loops are unbounded token spend |
| Partial include | `{{> partial_key@semver}}` | include a shared partial **pinned to an exact version** | depth ≤ 2; partials are prompt versions themselves |
| Comment | `{{! text }}` | stripped at compile | — |

**Forbidden (compiler-rejected):** expressions, arithmetic, filters/pipes, unpinned partials, recursion, environment access, and any interpolation inside `untrusted`-variable wrappers other than the variable itself. Formatting and derivation belong to the caller's code, where they are unit-testable; the template only arranges reviewed text and typed slots.

Compilation happens at version creation: the compiler produces a canonical AST (shipped in bundles so runtimes never re-parse), a static token estimate (template constant + Σ variable `max_tokens`), and the lint report. A version whose worst-case estimate exceeds the smallest context budget of its target classes ([Chapter 21 §4.2](21-ai-architecture.md)) fails compilation — oversized prompts are caught at authoring time, not at request time.

### 4.2 Variable contracts

The variable contract is a JSON Schema with four ACA extensions per variable:

| Extension | Values | Enforcement |
|---|---|---|
| `trust` | `trusted` \| `trusted_derived` \| `untrusted` | `untrusted` variables are auto-wrapped by the renderer in `untrusted_content` markers with a mandatory `source` attribute ([canon §13](../00-architecture-canon.md)); the envelope pre-filter rejects unwrapped external content, so the wrap must be structural, not authorial |
| `pii_max` | `P1` \| `P2` \| `P3` | highest PII class the variable may carry; cross-checked against the request's `pii_profile` at `ai-gateway` ([Chapter 18 §7.3](../volume-2-core-services/18-ai-gateway.md)); `P0` is unrepresentable — credentials never enter prompts |
| `max_tokens` | int | renderer truncates **never** — over-budget values are a render error returned to the context builder, which owns semantic truncation ([Chapter 21 §4.2](21-ai-architecture.md)) |
| `stability` | `stable` \| `semi` \| `volatile` | maps the variable to its envelope segment for prompt-cache alignment ([Chapter 18 §5.3](../volume-2-core-services/18-ai-gateway.md)); the linter forbids `volatile` variables in the system-prompt segment (rule R3 upstream) |

Example (abbreviated) — `cover-letter-agent.generate.system@2.3.0`:

```yaml
variables:
  user_voice_profile:   { type: object, required: true,  trust: trusted_derived, pii_max: P2, max_tokens: 400,  stability: semi }
  job_posting_digest:   { type: string, required: true,  trust: untrusted,       pii_max: P3, max_tokens: 1200, stability: volatile }
  company_evidence:     { type: array,  required: false, trust: trusted_derived, pii_max: P3, max_tokens: 800,  stability: volatile }
  grounded_claims_only: { type: boolean, required: true, trust: trusted,         pii_max: P3, max_tokens: 8,    stability: stable }
```

The contract is the integration surface agent versions pin against ([Chapter 22 §7](22-multi-agent-framework.md)): registration statically checks that the agent's context builder configuration can supply every required variable. A MAJOR bump is precisely "this static check may now fail."

## 5. Channels & Progressive Rollout

### 5.1 Channels

| Channel | Points at | Who resolves it | Eligibility |
|---|---|---|---|
| `dev` | any `draft` or `published` version | dev environment; shadow-channel agent runs ([Chapter 19 §10](../volume-2-core-services/19-agent-orchestrator.md)); replay harness | no gates beyond compilation |
| `canary` | a `published` version under progressive rollout | production, weighted subset of eligible traffic | gates G0–G3 passed (§7) |
| `stable` | the current production version | all remaining production traffic | full gate ladder G0–G5 |

Channel pointers are per-`prompt_key`, atomic, and historied. Resolution is **sticky**: a request resolves `canary` iff `hash(tenant_id, prompt_key) mod 100 < weight` — sticky per tenant so one tenant sees consistent behavior across a session (the same scheme, deliberately, as agent-version canarying in [Chapter 19 §10](../volume-2-core-services/19-agent-orchestrator.md); prompts hash on `prompt_key` rather than `agent_id` so agent and prompt canary populations decorrelate).

### 5.2 Progressive rollout ladder

| Step | Weight | Soak | Advance condition |
|---|---|---|---|
| 1 | canary 5 % | 24 h | online SLIs healthy for a continuous 24 h ([Chapter 21 §7](21-ai-architecture.md)); judge sample delta ≥ −1 point vs stable; cost/latency delta within declared envelope |
| 2 (consequential families only) | canary 25 % | 24 h | as above, plus zero consequential-action incidents attributed to the version |
| 3 | stable 100 % | — | G5 human approval where required (§7) |

5 % matches the platform's Argo canary convention and gives ~10⁴ sampled generations/day for busy families — enough for judge-based detection of a 3-point quality drop within one soak window; the 25 % intermediate step exists only for `consequential=true` families, where the blast radius of a subtle behavioral regression is user-visible actions, not just text quality.

**Composition with agent rollouts** (normative here, stated in [Chapter 22 §7](22-multi-agent-framework.md)): an agent canary may resolve a prompt canary only when both were promoted through a **joint eval run** (the eval record links both artifact ids); otherwise canary agents resolve `stable` prompts. Two independent 5 % canaries multiplying into an untested combination is the failure this rule prevents.

## 6. Evaluation Framework

### 6.1 Suite kinds

| Kind | Content | Minimum size | Gate role |
|---|---|---|---|
| `golden` | curated input→expectation cases per task family, from sanitized production traces + authored cases | 50/family gate minimum; 200 for generative families (pairwise judging needs volume to resolve 2–3-point deltas) | G1 |
| `adversarial` | injection and jailbreak library, shared verbatim with the [Chapter 22 §10](22-multi-agent-framework.md) `injection-suite` | ≥ 300 cases, growing per incident (§12.4) | G2 |
| `regression` | one case per production incident ([Chapter 21 §8](21-ai-architecture.md): "the taxonomy funds the eval suite") | monotonically growing | G1 (merged into golden scoring, tracked separately) |
| `calibration` | weekly human-labeled sample for judge honesty | n = 50/family/week ([Chapter 21 §7](21-ai-architecture.md)) | judge health, not promotion |

### 6.2 Case anatomy and scorers

An eval case stores an **input bundle** (rendered-variable values plus a frozen context-manifest snapshot, so the case replays byte-identically regardless of live data drift), an **expectation**, and a **scorer** binding:

| Scorer | Mechanism | Used for |
|---|---|---|
| `exact` | normalized string/JSON equality | extraction, classification, routing verdicts |
| `schema` | JSON Schema validation + field-level assertions | structured outputs, tool-call shapes |
| `rubric_judge` | `small`-class judge, ≤ 7 binary criteria, temperature 0, verdict + per-criterion booleans | constraint adherence, grounding, tone |
| `pairwise_judge` | `small`-class A/B comparison vs incumbent stable output, position-swapped and averaged (position bias is the dominant judge artifact) | generative quality deltas |
| `human` | review-queue task | calibration; sampled canary disputes |

Judges are deliberately `small`-class ([canon §10.4](../00-architecture-canon.md)): rubric decomposition into binary checks is what makes a small model a reliable judge, and it keeps a full golden run for a generative family under ~$2 on the batch lane. Judge prompts are themselves registry artifacts (`prompt-registry.judge.<family>.system`) — evaluated against the calibration set, versioned, and rolled out through the same channels; a judge change invalidates comparability, so eval runs record the judge version and gate comparisons only within it.

### 6.3 Human review queues

The registry owns the queue; `admin-service` renders the UI. Tasks enter from: (a) weekly calibration sampling, (b) `rubric_judge` verdicts in the uncertain band (2–3 of 7 criteria failed — clear passes and clear fails do not burn human time), (c) canary disputes where online judges disagree with offline golden results, and (d) all G5 approvals. SLA: 48 h to verdict (promotions for consequential families wait on it; the SLA is what keeps G5 from becoming a pocket veto). Reviewer verdicts feed judge recalibration; sustained judge–human agreement < 90 % flags the family's SLIs low-confidence ([Chapter 21 §9](21-ai-architecture.md)).

## 7. Promotion Gates & Auto-Rollback

### 7.1 Gate ladder

| Gate | Check | Threshold | Automated |
|---|---|---|---|
| G0 | compile + lint + security lint (§12.1) + adversarial *smoke* (30-case subset) | zero lint errors; smoke block rate 100 % | yes — blocks `publish` |
| G1 | golden + regression suites on all target model classes | pass rate ≥ 95 % (canon §14 SLO); no per-case-class regression > 2 points vs incumbent; no regression-suite case newly failing | yes |
| G2 | full adversarial suite | block rate ≥ 99.5 % ([Chapter 21 §9](21-ai-architecture.md)) | yes |
| G3 | compatibility verification for every class the routing policy may select (§8) | all reachable classes `verified` | yes |
| G4 | canary soak per §5.2 | online SLIs healthy 24 h; judge delta ≥ −1; cost delta ≤ +10 % unless declared in changelog | yes — `learning-pipeline` |
| G5 | human approval | required for `consequential=true` families and for any MAJOR bump | no — review queue, 48 h SLA |

Cost appears in G4 deliberately: a prompt that doubles output length passes every quality gate while silently doubling spend; the +10 % envelope forces the tradeoff into the changelog where a human sees it.

### 7.2 Auto-rollback

`learning-pipeline` owns thresholds and the rollback decision ([canon §10.4](../00-architecture-canon.md)); the registry executes pointer flips. Canary SLIs are evaluated on a 15-minute cadence over 1 h and 6 h windows ([Chapter 21 §7](21-ai-architecture.md)); a confirmed breach flips the pointer with no human in the loop.

```mermaid
sequenceDiagram
  participant J as Online judges on canary sample
  participant LP as learning-pipeline
  participant PR as prompt-registry-service
  participant RD as Redis pub-sub
  participant RT as Runtime bundle caches
  participant K as Kafka
  J->>LP: judge scores and SLI samples
  LP->>LP: evaluate canary SLIs each 15 min window
  alt regression confirmed
    LP->>PR: rollback request for prompt canary channel
    PR->>PR: flip canary pointer to incumbent stable version
    PR->>RD: publish bundle invalidation
    RD-->>RT: caches refetch within seconds
    PR->>K: emit aca.prompt.rollback.executed.v1
    K-->>LP: rollback recorded on the experiment
  else healthy through 24 h soak
    LP->>PR: promotion-eligible signal for G4
  end
```

End-to-end budget: detection ≤ 15 min (one evaluation window) + decision-to-flip ≤ 60 s + cache convergence ≤ 60 s (pub/sub push, 60 s poll as backstop) — a regressed canary stops receiving traffic well inside 20 minutes, bounded further by the 5 % weight. Rollbacks annotate the version with the triggering SLI; a version rolled back twice is barred from `canary` until a MAJOR/MINOR successor supersedes it (re-trying an identical artifact against the same signal is noise, not diligence).

A family-level quality-SLO breach freezes all canary→stable promotions in that family ([Chapter 21 §9](21-ai-architecture.md)); the freeze is a registry-enforced flag on the gate ladder, not a convention.

## 8. Model-Class Compatibility Matrix

A prompt is written against a model class's behavior envelope; routing failovers and degradation ladders ([Chapter 28 §8](28-model-routing.md)) can execute it on a class it was never tested against. The matrix makes that visible and gateable.

Per (`prompt_version`, `model_class`) the registry records:

| Status | Meaning | Set by |
|---|---|---|
| `untested` | no eval run on this class | default |
| `provisional` | golden pass 90–95 % on this class | eval orchestrator |
| `verified` | golden pass ≥ 95 % + adversarial ≥ 99.5 % on this class | eval orchestrator |
| `incompatible` | pass < 90 %, or structural failure (context overflow, tool-call format) | eval orchestrator; sticky until a new eval run |

Enforcement points:

1. **Publish (G3):** every class reachable for the prompt's task types under the current routing policy bundle ([Chapter 28 §4](28-model-routing.md)) — including degradation-ladder rungs and failover chain entries — must be `verified`. The registry queries the policy bundle's reachable-class set at gate time; a routing policy change that makes a new class reachable re-opens G3 for affected stable prompts (checked by the [Chapter 28 §13](28-model-routing.md) routing-change evaluation).
2. **Runtime:** the bundle ships the matrix row; if the router degrades to a class marked `incompatible`, the context builder substitutes the **compatibility fallback** — the newest ancestor version `verified` for that class — and tags the request `degraded_reason=prompt_compat_fallback`. Serving an older prompt on the right class beats serving the right prompt on a class it demonstrably fails on.
3. **`dev`/replay:** `provisional` is resolvable, so model-swap replay experiments ([Chapter 22 §10](22-multi-agent-framework.md)) can generate exactly the evidence that promotes a pair to `verified`.

## 9. Storage Model

```mermaid
erDiagram
  PROMPT ||--o{ PROMPT_VERSION : has
  PROMPT ||--o{ CHANNEL_POINTER : routed_by
  PROMPT_VERSION ||--o{ EVAL_RUN : evaluated_by
  PROMPT_VERSION ||--o{ CLASS_COMPATIBILITY : rated_for
  PROMPT_VERSION ||--o| SECURITY_REVIEW : gated_by
  PROMPT_VERSION ||--o{ PROMOTION : moved_by
  EVAL_SUITE ||--o{ EVAL_CASE : contains
  EVAL_SUITE ||--o{ EVAL_RUN : drives
  EVAL_RUN ||--o{ HUMAN_REVIEW_TASK : samples
  PROMPT {
    uuid id
    text prompt_key
    text task_family
    text owner_team
    bool consequential
  }
  PROMPT_VERSION {
    uuid id
    uuid prompt_id
    int major
    int minor
    int patch
    text body
    jsonb compiled_ast
    jsonb variable_contract
    bytea content_hash
    int token_estimate
    text status
    text changelog
  }
  CHANNEL_POINTER {
    uuid prompt_id
    text channel
    uuid version_id
    int weight
    text reason
  }
  EVAL_RUN {
    uuid id
    uuid version_id
    uuid suite_id
    text model_class
    text judge_version
    jsonb scores
    text status
  }
  CLASS_COMPATIBILITY {
    uuid version_id
    text model_class
    text status
    uuid eval_run_id
  }
```

DDL excerpt (Postgres 16; forward-only migrations per [canon §11](../00-architecture-canon.md)):

```sql
CREATE TABLE prompt (
  id            uuid PRIMARY KEY,                 -- UUIDv7
  prompt_key    text NOT NULL UNIQUE,
  task_family   text NOT NULL,
  owner_team    text NOT NULL,
  consequential boolean NOT NULL DEFAULT false,
  description   text,
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE prompt_version (
  id                 uuid PRIMARY KEY,
  prompt_id          uuid NOT NULL REFERENCES prompt(id),
  major              int NOT NULL, minor int NOT NULL, patch int NOT NULL,
  body               text NOT NULL,
  compiled_ast       jsonb NOT NULL,
  variable_contract  jsonb NOT NULL,
  content_hash       bytea NOT NULL,
  token_estimate     int NOT NULL,
  status             text NOT NULL DEFAULT 'draft'
                     CHECK (status IN ('draft','published','deprecated','yanked','discarded')),
  changelog          text NOT NULL,
  security_review_id uuid,
  created_by         text NOT NULL,
  published_at       timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (prompt_id, major, minor, patch)
);
-- Immutability: BEFORE UPDATE trigger rejects changes to body/compiled_ast/
-- variable_contract/content_hash when OLD.status <> 'draft'.

CREATE TABLE channel_pointer (
  prompt_id  uuid NOT NULL REFERENCES prompt(id),
  channel    text NOT NULL CHECK (channel IN ('dev','canary','stable')),
  version_id uuid NOT NULL REFERENCES prompt_version(id),
  weight     int  NOT NULL DEFAULT 100 CHECK (weight BETWEEN 0 AND 100),
  reason     text NOT NULL,
  updated_by text NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (prompt_id, channel)
);
CREATE TABLE channel_pointer_history (LIKE channel_pointer INCLUDING ALL, valid_to timestamptz);

CREATE TABLE class_compatibility (
  version_id  uuid NOT NULL REFERENCES prompt_version(id),
  model_class text NOT NULL,
  status      text NOT NULL DEFAULT 'untested'
              CHECK (status IN ('untested','provisional','verified','incompatible')),
  eval_run_id uuid,
  updated_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (version_id, model_class)
);
```

Registry data is **platform control-plane data, not tenant data**: no `tenant_id`, no RLS — a deliberate exception to the canon §6 every-row rule, recorded here because prompts are shared platform artifacts (tenant-custom prompts are a Phase 2 feature that will add a tenant-scoped table *beside* this one, §17). Eval case input bundles may embed sanitized production traces; sanitization to ≤ `P3` happens at curation time ([Chapter 21 §7](21-ai-architecture.md)), so the registry never stores `P1` content. Scale is small by design: O(10³) prompts, O(10⁴) versions/year, O(10⁵) eval results/year — a single Postgres instance with a read replica; the hot path is Redis-cached bundles, not the database.

## 10. API

### 10.1 Admin REST (via `api-gateway`, admin/CI scopes; problem+json errors, Idempotency-Key on POSTs per canon §7)

| Method | Path | Purpose |
|---|---|---|
| POST | `/v1/prompts` | create prompt identity |
| POST | `/v1/prompts/{key}/versions` | create draft (body + contract); returns compile + lint report |
| POST | `/v1/prompts/{key}/versions/{semver}/publish` | run G0; publish (immutability begins) |
| POST | `/v1/prompts/{key}/versions/{semver}/evals` | trigger suites × class matrix (G1–G3 evidence) |
| POST | `/v1/prompts/{key}/channels/{channel}` | move pointer / set weight; gate-checked; `reason` mandatory |
| POST | `/v1/prompts/{key}/rollback` | flip `canary` (or `stable`, break-glass) to incumbent; used by `learning-pipeline` |
| POST | `/v1/prompts/{key}/versions/{semver}/yank` | yank with incident/security reference |
| GET | `/v1/prompts/{key}/versions/{semver}` | artifact + eval history + compatibility row |
| GET | `/v1/eval-suites/{id}/cases` / POST same | suite curation |
| GET | `/v1/review-queue?status=pending` | human review tasks (G5, calibration, disputes) |

### 10.2 Runtime gRPC (mesh mTLS; read-only)

| RPC | Semantics |
|---|---|
| `ResolveBundle(prompt_keys[], channel_context) → SignedBundle` | compiled ASTs + contracts + channel pointers + compatibility rows for the requested keys; `channel_context` carries tenant hash inputs for sticky canary resolution |
| `GetVersion(prompt_key, semver) → Artifact` | pinned fetch (replay, forensics) |
| `GetManifest(prompt_keys[]) → HashManifest` | `content_hash` per current pointer — `ai-gateway`'s R4 sampling verifies envelope prompt segments against this |

Bundles are Ed25519-signed (platform signing key from Vault) so a compromised cache cannot forge prompt content; runtime caches poll every 60 s and subscribe to Redis pub/sub invalidation (pointer changes converge in seconds, poll is the backstop). **Fail-static rule:** on registry outage, caches serve their last-validated bundle for up to 24 h — a stale stable prompt is strictly safer than no prompt; after 24 h, agents degrade to L0-suggest behavior rather than run on unverifiable prompt state (staleness beyond a day means rollbacks may not have propagated).

### 10.3 Events

| Topic | Key | Payload sketch |
|---|---|---|
| `aca.prompt.version.published.v1` | `prompt_id` | `{prompt_key, semver, content_hash, classes_verified[]}` |
| `aca.prompt.channel.updated.v1` | `prompt_id` | `{prompt_key, channel, from_semver, to_semver, weight, reason, actor, auto}` |
| `aca.prompt.rollback.executed.v1` | `prompt_id` | `{prompt_key, channel, from_semver, to_semver, trigger_sli, eval_window}` |
| `aca.prompt.eval.completed.v1` | `prompt_id` | `{semver, suite_kind, model_class, scores, pass}` |

`learning-pipeline` consumes all four to correlate quality time series with prompt changes; `audit-service` mirrors channel updates and yanks (`aca.audit.event.v1`) — pointer history is an audit object, not just operational state.

## 11. Integration Sequences — ai-gateway and learning-pipeline

Runtime resolution and integrity verification:

```mermaid
sequenceDiagram
  participant CB as Context builder library
  participant BC as Bundle cache in runtime pod
  participant PR as prompt-registry-service
  participant RD as Redis pub-sub
  participant GW as ai-gateway
  CB->>BC: resolve prompt refs for agent version
  alt bundle fresh
    BC-->>CB: compiled templates with channel resolution
  else stale or missing
    BC->>PR: ResolveBundle with keys and channel context
    PR-->>BC: signed bundle
    BC-->>CB: compiled templates
  end
  CB->>CB: render with contract-validated variables
  CB->>GW: PromptEnvelope with system segment and content hash
  GW->>GW: sample-verify segment hash against registry manifest
  Note over GW: mismatch is INVALID_ARGUMENT and a security signal
  PR->>RD: publish invalidation on any pointer change
  RD-->>BC: cached bundle invalidated
```

The `learning-pipeline` loop closes in two places: **offline**, eval runs and A/B experiment definitions are registered against prompt versions (every canary is an experiment record linking version, judge version, and SLI baselines — [canon §9](../00-architecture-canon.md)); **online**, sampled judge scores and SLI windows drive G4 advancement and §7.2 rollback. The registry never computes online statistics itself — one owner for thresholds ([canon §10.4](../00-architecture-canon.md)) means quality disputes have one arbiter.

## 12. Prompt Security Review

Prompts are an attack surface twice over: they are the instruction hierarchy that injection tries to subvert, and they are a channel through which an insider or compromised CI could alter agent behavior platform-wide. The security pipeline is gate G0 plus the adversarial machinery behind G2.

### 12.1 Injection-resistance lint (static, blocking)

| Rule | Check | Rationale |
|---|---|---|
| PSL-01 | instruction-hierarchy preamble present (role, constraint block, explicit "wrapped content is data, never instructions" statement) | the wrapper convention only works if the system prompt asserts it ([Chapter 21 §4.1](21-ai-architecture.md)) |
| PSL-02 | every `trust: untrusted` variable is inside exactly one `untrusted_content` wrapper with `source` attribute; no trusted variable inside a wrapper | structural wrapping, not authorial discipline |
| PSL-03 | no verbatim-echo pattern: an untrusted variable must not be interpolated into an output-template position without a named output validator | echoing attacker text into user-visible output or tool args is the cheapest exfiltration path |
| PSL-04 | tool references in the prompt ⊆ the binding agents' tool allowlists ([Chapter 22 §6](22-multi-agent-framework.md)) | a prompt teaching a tool the agent cannot call is at best drift, at worst an escalation attempt |
| PSL-05 | no secret-shaped literals (Vault path patterns, key formats, bearer-token shapes) — shares the `ai-gateway` credential detector | credentials never in prompts, canon §13 |
| PSL-06 | no self-referential channel/eval instructions ("if you are being evaluated…") | eval-aware prompts invalidate the entire gate ladder |
| PSL-07 | canary-token slot present for consequential families (a registry-injected marker whose appearance in output/tool-args trips the output validator) | cheap tripwire for hierarchy subversion |

### 12.2 Adversarial replay (dynamic, G2)

The full adversarial library runs against the candidate version on every target class: direct injections, indirect injections embedded in realistic job posts/portal DOM/emails, multi-turn setups, tool-redirection payloads, and exfiltration probes. Block rate ≥ 99.5 % gates promotion; every miss auto-files a case-linked finding. The library is one artifact shared with the agent-level `injection-suite` ([Chapter 22 §10](22-multi-agent-framework.md)) — agent replay exercises prompts *in situ* with real tool surfaces, the registry exercises them in isolation; a case lives in both harnesses and regressions in either block the same promotions.

### 12.3 Human security review

Mandatory for `consequential=true` families (browser submission, email send, negotiation) and for any change to PSL-relevant structure (wrapper placement, validator bindings, tool references). Reviewers are the security team via the G5 queue; `security-agent` pre-screens diffs and annotates risk (changed trust boundaries, new untrusted variables) so human attention lands on the deltas that matter. Verdicts are recorded in `security_review` and referenced by the version — an unreviewed consequential prompt cannot reach `stable` mechanically, not procedurally.

### 12.4 Incident loop

Every production injection incident ([Chapter 21 §8](21-ai-architecture.md)) contributes: (a) a minimized adversarial case within 48 h, (b) a lint-rule proposal if the vector was structural, (c) a yank decision for the exposed version if the vector is exploitable at `stable`. The ≥ 99.5 % block-rate SLI is continuously measured by adversarial replay against the *stable* channel, not only at promotion — drift in provider model behavior can silently reopen closed holes ([Chapter 21 §8](21-ai-architecture.md), silent quality drift).

## 13. Failure Scenarios

| Scenario | Impact | Response |
|---|---|---|
| Registry Postgres down | no publishes/promotions; runtime unaffected (fail-static caches, §10.2) | PITR restore; promotions queue; error budget burns on admin SLO only |
| Redis pub/sub loss | pointer changes propagate at 60 s poll cadence instead of seconds | acceptable degradation; alert at > 5 min convergence |
| Bundle signature verification fails fleet-wide | runtimes refuse new bundles, serve last-validated | page — indicates key rotation error or tampering; Vault key check |
| Eval orchestrator backlog (batch lane congestion) | promotions delayed | promotions are never latency-critical; priority bump only for security-fix versions |
| Judge model regression (provider drift) | gate decisions unreliable | calibration-set breach flags family SLIs low-confidence; freeze promotions in affected families; pin judge provider version |
| Erroneous auto-rollback (SLI false positive) | canary reverted needlessly | rollback is cheap and safe by design; re-promotion requires re-passing G4 — the asymmetry is intentional |
| Registry compromise attempt (unauthorized pointer flip) | potential platform-wide behavior change | pointer mutations require admin JIT elevation (canon §13), are audit-mirrored, and anomalous flips (no linked eval evidence) trip `security-agent` monitoring |

## 14. Scaling Strategy

The registry is deliberately a low-QPS control-plane service: runtime load is absorbed by per-pod bundle caches and Redis (fleet-wide `ResolveBundle` misses only on pod cold-start and invalidation). Sizing: 3 replicas per region for availability, not throughput; Postgres single-writer + read replica; eval throughput scales via the `ai-gateway` batch lane, which is elastic by design ([Chapter 18 §9](../volume-2-core-services/18-ai-gateway.md)). The only growth dimension needing attention is eval-case storage with embedded input bundles (~50 KB/case) — object-storage offload for bundles > 64 KiB keeps the OLTP store lean.

## 15. Security Considerations

- Mutations require admin JIT elevation ([canon §13](../00-architecture-canon.md)); CI publishes via a scoped service identity limited to `versions` + `evals` (never channel moves — humans or `learning-pipeline` only).
- Bundles signed (Ed25519, Vault-held key, `kid` rotation like JWKS); runtimes pin the verification key set.
- Registry content is `P3` by policy; curation-time sanitization enforced with a `P1`-detector scan on case ingestion (defense in depth against a leaky sanitizer upstream).
- Full §12 pipeline; channel history and yanks mirrored to `audit-service` (hash-chained, 7-year retention per canon §11).
- The `stable` channel pointer set is part of the platform kill-switch surface: the platform-scope kill switch can freeze all channels (no flips, including automated ones) during a security incident.

## 16. Observability

| Signal | Type | Alert |
|---|---|---|
| `prompt_bundle_staleness_seconds` (max over fleet) | gauge | > 300 s — invalidation path degraded |
| `prompt_resolution_channel_ratio` (canary/stable per key) | gauge | canary ratio drift from configured weight > 2 pp — sticky-hash bug |
| `prompt_eval_pass_rate{family,class,suite}` | gauge | family < 95 % — promotion freeze (auto) |
| `prompt_adversarial_block_rate{channel=stable}` | gauge | < 99.5 % — security incident process |
| `prompt_rollbacks_total{auto}` | counter | > 2/week/family — authoring quality review |
| `prompt_compat_fallback_served_total` | counter | any sustained rate — routing policy and G3 coverage mismatch |
| `judge_human_agreement{family}` | gauge | < 90 % — judge recalibration |
| Gate latency (publish→stable lead time) | histogram | p50 > 5 days — process, not system, alarm |

Traces: pointer flips and bundle publishes carry `trace_id` through to cache-refresh spans, so "when did the fleet actually converge" is answerable per rollout.

## 17. Future Improvements

- **Tenant-custom prompts** (enterprise): tenant-scoped overlay table with RLS, constrained to variable-contract-compatible overrides of platform prompts; runs the same gate ladder with tenant-funded eval budgets.
- **Joint prompt+routing optimization:** feed §8 compatibility economics into [Chapter 28 §12](28-model-routing.md) adaptive routing so class selection can weigh "cheaper class, older verified prompt" against "target class, current prompt."
- **Prompt diff intelligence:** `small`-class semantic diff summaries on version creation, flagging behavior-relevant deltas for reviewers (today reviewers read raw diffs + changelog).
- **Auto-generated adversarial cases:** red-team agent generating candidate injection cases against draft prompts in `dev`, human-triaged into the shared library.
- **Cross-version output drift maps:** systematic pairwise judging between stable versions across quarters to quantify cumulative drift invisible in single-step comparisons.

## 18. Alternatives Considered & Tradeoffs

| Decision | Alternative | Why rejected / tradeoff accepted |
|---|---|---|
| Dedicated registry service | Prompts in git only, shipped with service images | Git alone couples prompt rollout to deploys, cannot express per-tenant sticky canaries, and gives no runtime pointer flip for sub-minute rollback. Cost accepted: one more control-plane service (mitigated by fail-static caches removing it from hot paths). Git remains the *authoring* interface via CI. |
| Logic-less PromptTemplate v1 | Jinja2/full templating | Full templating hides untestable logic in strings and makes the security lint undecidable. Accepted cost: some duplication across similar prompts, mitigated by pinned partials. |
| Semver with contract-based MAJOR | Content-hash-only addressing | Hashes give no compatibility signal; agent registration needs "will my variables still bind" statically. Accepted cost: authors must judge MINOR vs MAJOR; the compiler auto-forces MAJOR on contract diffs, so misjudgment is only possible in the lenient direction for intent changes. |
| `small`-class rubric judges | `large`-class judges for higher fidelity | 10–20× eval cost would ration evaluation exactly where volume matters (canary sampling); binary-rubric decomposition recovers most fidelity. Weekly human calibration bounds residual judge error. Accepted: judges miss subtle quality dimensions between calibrations. |
| Rollback authority in `learning-pipeline` | Registry self-monitors and rolls back | Two arbiters of quality would disagree during incidents; canon §10.4 already places thresholds in `learning-pipeline`. Registry keeps a break-glass manual rollback for the case where `learning-pipeline` itself is down. |
| Fail-static 24 h bundle serving | Fail-closed on registry outage | Fail-closed turns a control-plane outage into a platform-wide AI outage. 24 h bound limits exposure to un-propagated rollbacks; beyond it, degrading agents to L0 caps consequence risk. |
| Compatibility fallback (older verified version on degraded class) | Reject request when class unverified | Rejection turns a budget degradation into a user-visible failure; a previously-verified ancestor is a known-good artifact. Accepted: temporarily serving older prompt behavior, flagged via `degraded_reason` for observability. |
