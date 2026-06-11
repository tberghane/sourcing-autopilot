# Changelog

> Per OPS-01 (PRD v1.1): at runtime this file is rendered from the `changelog` DB table by a single-writer docs job. Until the system exists, it is maintained by hand.

## [2026-06-11] — TECHNICAL_REVIEW

**Run ID:** n/a (pre-build)
**Role:** n/a
**Change:** Technical review of PRD v1.0-draft completed (`docs/TECHNICAL_REVIEW.md`). P0 approved contingent on blocking findings B-1–B-4.
**Author:** agent (Claude, dev-time)

## [2026-06-11] — SPEC_REVISION

**Run ID:** n/a (pre-build)
**Role:** n/a
**Change:** PRD revised v1.0-draft → v1.1-draft incorporating review resolutions.
**Before:** Persistent VM (INF-01); RLS-as-written (INF-03); agent framework runtime (AGT-01); runtime `git commit` for changelog/Tier 3 (PRM-03/OPS-01/OPS-02); global `linkedin_url` UNIQUE; no prompt-injection requirement; no retention policy; per-run cost cap only.
**After:** Cloud Run Jobs (INF-01); tenant-scoped sessions + CI isolation test (INF-03); pipeline orchestrator with code-enforced circuit breakers (AGT-01, resolves OQ-02); DB-backed `changelog`/`todos`/`calibration_rules` with rendered docs, runtime holds no Git credentials; `UNIQUE (client_id, linkedin_url)` + append-only `scores` table; AGT-04 prompt-injection controls; INF-06 PII retention & deletion; global daily cost cap + batched scoring (MON-01). OQ-01 (GCP), OQ-02, OQ-03 (Supabase), OQ-04, OQ-09 resolved; OQ-10 (cross-client outreach policy) added.
**Author:** agent (Claude, dev-time)
