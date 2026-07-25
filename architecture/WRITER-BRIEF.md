# WRITER BRIEF — ACA Software Architecture Specification

Instructions for chapter authors. Read `00-architecture-canon.md` first; it is authoritative for every name, technology, and convention. This brief covers structure, style, and diagram rules.

## Audience & register

Senior engineers who will build the platform. Professional SAD register: precise, declarative, decision-oriented. No marketing language, no filler ("In today's fast-paced world…"), no second person. Prefer tables for enumerable facts, prose for reasoning, diagrams for structure and flows.

## Chapter skeleton (mandatory)

```markdown
# <NN> — <Chapter Title>

| | |
|---|---|
| Document ID | ACA-SAS-<NN> |
| Volume | <N> — <Volume Name> |
| Status | Draft v1.0 |
| Depends on | <links to canon + related chapters> |

## 1. Overview
<what this chapter covers, why it exists, 2–4 paragraphs>

## 2..n <Component/topic sections>

## n+1. Failure Scenarios          (chapter-level if not per-component)
## n+2. Scaling Strategy
## n+3. Security Considerations
## n+4. Observability
## n+5. Future Improvements
## n+6. Alternatives Considered & Tradeoffs
```

For each significant **component** described, cover (as subsections or a structured table): Purpose, Responsibilities, Interactions, Sequence(s), Failure Scenarios, Scaling, Security, Monitoring, Future Improvements, Alternatives Considered, Tradeoffs. Consolidate at chapter level where per-component repetition would be noise — judgment over checklist, but nothing may be silently omitted.

## Depth

- Target 350–900 lines per chapter (service chapters toward the high end; narrow topics like `45 Docker` may be shorter but must still be decision-rich, not tutorial content).
- Include concrete contracts: REST endpoint tables, event tables (topic, key, schema sketch), data-model sketches (SQL DDL excerpts where the chapter owns a schema), config/limit tables, SLO tables.
- Illustrative code/DDL/YAML only to pin down architecture — no application-logic tutorials.

## Diagram rules (Mermaid)

Every chapter: **at least 2 diagrams**; service and flow chapters: 3–5. Choose types deliberately: `flowchart` (context/container/component — C4-style via subgraphs), `sequenceDiagram`, `erDiagram`, `stateDiagram-v2`, `flowchart` for data/event flow. Deployment diagrams: `flowchart` with subgraphs per zone/cluster.

**Mermaid safety rules (violations break rendering — follow strictly):**
1. Start blocks with ```` ```mermaid ```` and a valid type keyword (`flowchart TB`, `sequenceDiagram`, `erDiagram`, `stateDiagram-v2`).
2. Flowchart node IDs: alphanumeric + underscores only (`api_gw`), never starting with a digit; put display text in quoted labels: `api_gw["API Gateway"]`.
3. **Always quote labels** containing spaces or any of `() {} [] / : , . -` → `n1["Browser Fleet (Playwright)"]`.
4. Never use the word `end` as a node ID (reserved). Avoid `class`, `style`, `click` as IDs.
5. Edge labels quoted: `a -->|"submit (async)"| b`.
6. sequenceDiagram: declare participants first: `participant AG as API Gateway` (no quotes needed after `as`); message text must not contain `:` beyond the first; use `->>` / `-->>` and `alt/else/opt/par/loop` blocks closed with `end`. `Note over A,B: text` is allowed.
7. erDiagram: entity names UPPER_SNAKE; attributes `type name` one per line, types single-word (`uuid`, `text`, `timestamptz`, `jsonb`, `int`, `numeric`, `bool`); relationships like `USER ||--o{ APPLICATION : submits`.
8. stateDiagram-v2: state names alphanumeric/underscore; transitions `A --> B : event`.
9. No markdown, HTML, backticks, or `#` inside labels. No parentheses inside erDiagram attribute lines. Keep any single diagram ≤ ~40 nodes — split rather than cram.
10. subgraph syntax: `subgraph name["Display Name"]` … `end`.

## Consistency requirements

- Service names, agent names, topic names, plan names, SLOs, model classes: **exactly** as in the canon. Do not invent synonyms ("Job Discovery Service" ❌ → `job-intelligence-service` ✅).
- Cross-reference sibling chapters with relative links, e.g. `[Chapter 18 — AI Gateway](../volume-2-core-services/18-ai-gateway.md)`.
- Use the canon's autonomy levels (L0–L3), PII classes (P0–P3), plan names, and template trust ladder verbatim.
- Currency/pricing: only the reference prices in the canon; mark as "reference".
- Dates/versions: avoid fake specific dates; use phases ("Phase 1/2") and quarters relative to GA.

## What to avoid

- Restating the entire canon — link to it and build on it.
- Generic explanations of technologies (what Kafka is) — assume expert readers; document *our* decisions and configurations.
- Unexplained magic numbers — every limit/threshold gets a rationale (one line suffices).
- Vendor lock-in as silent assumption — where a vendor is chosen, name the seam/abstraction per canon.
