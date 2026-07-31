# ACA — AI Career Agent

> **Mission: "Hire an AI that gets you a job."**

AI Career Agent (ACA) is an autonomous, multi-tenant AI operating system for the job search. It continuously discovers jobs across the internet, evaluates fit against a deeply understood user profile, generates tailored application materials, executes browser-based application workflows where supported and permitted, tracks outcomes, prepares users for interviews and negotiation, and learns from every signal. The web application is only the control plane; the product is the distributed agent system behind it.

Architecturally, ACA is a cloud-native microservice platform on Kubernetes with an event-driven backbone on Kafka, durable execution on Temporal, and an AI-native control plane: 32 canonical services and 23 logical agents behind a single model gateway, a Universal Portal Engine that learns hiring portals rather than hardcoding them, and a browser fleet of ephemeral sandboxed sessions. Multi-tenancy is enforced at the data, compute, and cost layers, with a cell-based architecture for enterprise isolation and data residency.

Because ACA performs consequential actions with a user's own identity and credentials, trust is engineered into the foundation rather than asserted: explicit consent scopes, user-configurable autonomy levels (L0–L3) with human-in-the-loop checkpoints, policy checks before every consequential action, encrypted credential custody that never exposes secrets to agent context, an immutable audit chain, and a kill switch at user, tenant, and platform scope.

## Architecture Specification

The full specification lives in [`architecture/`](architecture/), anchored by the [Architecture Canon](architecture/00-architecture-canon.md) — the single source of truth for names, technologies, conventions, and SLOs. **Start with the [master index](architecture/README.md)**, which includes suggested reading paths per role.

| Volume | Scope | Status |
|---|---|---|
| [1 — Foundation](architecture/volume-1-foundation/) | Mission, vision, requirements, system context, macro-architecture | Drafted (ch. 01–08) |
| [2 — Core Services](architecture/volume-2-core-services/) | Edge, identity, profile/job domains, AI-plane services, service-by-service specs | Drafted (ch. 09–20) |
| [3 — AI Architecture](architecture/volume-3-ai-architecture/) | Agent runtime, orchestration, memory, prompts, routing, cost, learning loop | In progress (ch. 21) |
| 4 — Data Platform | Ownership, schemas, events, vector/graph/search stores, retention, CQRS | Planned |
| 5 — Security | Zero trust, PII classes, session vault, prompt-injection defense, compliance | Planned |
| 6 — Operations | Kubernetes, GitOps, cells, environments, DR, observability, incident response | Planned |
| 7 — Browser Intelligence | Universal Portal Engine internals, browser fleet, portal intelligence, proxies | Planned |
| 8 — Business Platform | Billing, entitlements, notifications, analytics, recruiter/university, public API, mobile | Planned |

## Documentation Set

This repository currently contains the **software architecture specification only** — there is no application code yet. The specification is the build contract for the platform: chapter numbering is global across volumes, every chapter conforms to the [Architecture Canon](architecture/00-architecture-canon.md), and authoring rules live in the [Writer Brief](architecture/WRITER-BRIEF.md).
