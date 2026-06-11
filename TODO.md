# TODO

> Per OPS-02 (PRD v1.1): at runtime this file is rendered from the `todos` DB table. Until the system exists, it is maintained by hand.

## Active

### Decisions (Taylor)
- [ ] [DECISION] Confirm OQ-10 cross-client outreach policy (recommendation: first-come holds 30 days) — needed before Phase 3
- [ ] [DECISION] OQ-06: Tier 3 updates — human approval in V1 (default) — confirm
- [ ] [DECISION] OQ-08: Clarion role cadences (proposal: Founding AE + FDE every 4h, others 12h) — with Ryan Gallagher
- [ ] [EXTERNAL] OQ-07: unstick Pin job `0e9f6c7a` (Clarion Deployment Strategist) — Dave / Pin support, before Clarion launch

### Phase 0 — Cloud Setup (Weeks 1–2)
- [ ] [INFRA] Create GCP project: Cloud Run Jobs + trigger service, Serverless VPC connector, Secret Manager + Workload Identity
- [ ] [INFRA] Enable Vertex AI; verify one Claude call end-to-end
- [ ] [INFRA] Stand up Supabase; write full V1 DDL incl. `scores`, `changelog`, `todos`, `roles`, `clients`, `outreach_queue` (review M-1/M-5)
- [ ] [SECURITY] Implement RLS tenant-context pattern + CI client-isolation test (review B-1) — blocker for Phase 1
- [ ] [SECURITY] Document PII retention policy + deletion runbook (INF-06 / review H-4)
- [ ] [COST] Configure global daily spend cap (review H-1)
- [ ] [INFRA] Validate MCP connectivity (Pin, Wrangle, Notion, Slack) from a Cloud Run execution

### Phase 1 — Orchestrator Core (Weeks 3–5)
- [ ] [BUILD] Pipeline orchestrator with code-enforced circuit breakers (AGT-01 / review B-4)
- [ ] [BUILD] Phase-scoped tool registry — scoring stage returns JSON only, zero write tools (AGT-04 / review B-2)
- [ ] [BUILD] Three-tier prompt system; Tier 3 from `calibration_rules` table (PRM-03 / review B-3)
- [ ] [BUILD] DB-backed changelog/todos + single-writer rendered-docs job (OPS-01/02)
- [ ] [BUILD] Non-AI data pull scripts → `raw_ingest` (INF-04)
- [ ] [BUILD] Dedup with URL normalization + client-conflict SQL check (AGT-03)
- [ ] [BUILD] Slack Block Kit review card (CAL-01)
- [ ] [OPS] One backup restore test (INF-03)

## Completed
- [x] [2026-06-11] Technical review of PRD v1.0 (`docs/TECHNICAL_REVIEW.md`) — resolves OQ-01/02/03/04/09
- [x] [2026-06-11] PRD revised to v1.1 incorporating blocking findings B-1–B-4 and high findings H-1–H-4
