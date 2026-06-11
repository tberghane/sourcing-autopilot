# Calibratr Sourcing Autopilot — Product Requirements Document

**Version:** 1.0-draft  
**Owner:** Taylor Berghane, Calibratr  
**Status:** Awaiting technical review  
**Last Updated:** June 2026

---

## Table of Contents

1. [Problem Statement](#problem-statement)  
2. [Goals](#goals)  
3. [Non-Goals](#non-goals)  
4. [User Stories](#user-stories)  
5. [System Architecture Overview](#system-architecture-overview)  
6. [Requirements](#requirements)  
   - P0 — Infrastructure & Data Layer  
   - P0 — Sourcing Agent Core  
   - P0 — Prompt Architecture  
   - P0 — Eval Layer  
   - P0 — Changelog & Task Tracking  
   - P1 — Calibration & Feedback Loop  
   - P1 — ATS \+ Notion Sync  
   - P1 — Monitoring & Cost Controls  
   - P2 — Future Considerations  
7. [Data Schema](#data-schema)  
8. [Success Metrics](#success-metrics)  
9. [Open Questions](#open-questions)  
10. [Timeline & Phasing](#timeline--phasing)  
11. [Technical Reviewer Brief](#technical-reviewer-brief)

---

## Problem Statement

Calibratr's recruiters currently execute sourcing manually: pulling candidates from Wrangle and Pin, reviewing profiles, calibrating against role DNA, and managing outreach sequencing. This process is high-quality but not scalable — capacity is capped by human hours, calibration knowledge lives in recruiter heads rather than code, and there is no mechanism for continuous improvement from signal (accept/reject) data. As Calibratr moves from a copilot model toward an autonomous sourcing product, the gap between what recruiters do manually and what a well-architected agent system can do autonomously is the core business problem to solve. Failing to build this now means Calibratr cannot add clients without adding headcount, and the "Sourcing Autopilot" product line cannot exist.

---

## Goals

1. **Autonomous sourcing pipeline**: A cloud-deployed agent system sources, scores, and queues candidates for human review with zero manual triggering — running on configurable cron schedules (e.g., every 4 hours per role).  
2. **Calibration that compounds**: Every accept/reject signal from the recruiter updates the feedback rules layer so the next sourcing run is measurably more accurate than the prior one.  
3. **Cost-optimized architecture**: Non-reasoning tasks (data pulls, schema validation, ATS writes) run as plain scripts without LLM calls; LLMs are called only where reasoning is required.  
4. **Self-validating output**: An eval layer runs on every significant LLM output — candidate shortlist, score rationale, outreach draft — so the system can flag its own errors before a human sees them.  
5. **Auditable and recoverable**: A changelog and active TODO list are maintained by the codebase itself, so any engineer (or agent session) can understand what has changed, what is in progress, and how to roll back.

---

## Non-Goals

**V1 scope exclusions:**

1. **Full remove-human-from-loop autonomy** — V1 still has a human review gate before outreach is sent. The architecture should be designed agentic-first (so human-in-loop can be removed later), but we are not shipping zero-human-review in V1.  
2. **Multi-tenant client isolation at the infrastructure layer** — V1 runs all clients on a shared compute instance with logical isolation (client\_id partitioning in the DB). True cloud-tenant isolation is a V2 concern after we validate the model.  
3. **Custom fine-tuned models** — Calibration happens through the feedback rules layer (prompt-tier 3), not model fine-tuning. Fine-tuning is architecturally incompatible with the speed of feedback iteration we need.  
4. **Outreach deliverability infrastructure** — V1 leverages existing outreach tools (existing sequences, Pin outreach). Building our own SMTP/deliverability stack is out of scope.  
5. **Real-time streaming / sub-minute sourcing** — Cron-scheduled runs (hourly minimum) are sufficient for V1. Real-time event-driven sourcing (e.g., trigger on a new LinkedIn job posting) is a V2 signal integration.

---

## User Stories

### Calibratr Recruiter (Primary Operator)

- As a recruiter, I want the system to run sourcing for all active roles every 4 hours so that I wake up to a populated queue of pre-scored candidates instead of starting each day from zero.  
- As a recruiter, I want to accept or reject a candidate with a single click and have that signal immediately update the calibration layer so that the next run reflects my feedback.  
- As a recruiter, I want to see a plain-language score rationale for every candidate so that I can trust or challenge the system's reasoning rather than treat it as a black box.  
- As a recruiter, I want any outreach draft to pass a voice check before it enters my review queue so that I never send something that breaks Calibratr's brand rules.  
- As a recruiter, I want a full changelog of every change to the system — prompts, scripts, schema, calibration weights — so that I can roll back when something breaks.

### Taylor / Calibratr Principal (Product Owner)

- As the product owner, I want the system's codebase to be agentic-first from day one so that adding new capabilities (new signal sources, new MCP tools, new clients) does not require architectural rewrites.  
- As the product owner, I want cost per sourcing run to be visible and bounded so that I can price the Autopilot product with confidence and prevent runaway LLM spend.  
- As the product owner, I want a technical reviewer to sign off on the architecture before we build more than 20% of it so that security issues, cost traps, and scaling blockers are caught early.

### Client (e.g., Clarion Health)

- As a client, I want my candidates to reflect my specific hiring bar and company context (Company DNA \+ Role DNA) so that the system behaves like a recruiter who has been deeply briefed on my company, not a generic job-board scraper.  
- As a client, I want to see a weekly pipeline report showing candidates in queue, in review, and in outreach — without having to log into multiple tools.

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLOUD (GCP or AWS)                          │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Scheduler   │───▶│ Agent Runner │───▶│   Data Lake      │  │
│  │  (Cron Jobs) │    │ (Hermes /    │    │  (PostgreSQL /   │  │
│  │              │    │  OpenClaw /  │    │   Supabase /     │  │
│  │  Per-role    │    │  Claude API) │    │   BigQuery)      │  │
│  │  schedules   │    │              │    │                  │  │
│  └──────────────┘    └──────┬───────┘    └────────┬─────────┘  │
│                             │                     │            │
│                    ┌────────▼──────────┐           │            │
│                    │   MCP Tool Layer  │           │            │
│                    │  ┌─────────────┐  │           │            │
│                    │  │  Pin MCP    │  │           │            │
│                    │  │  Wrangle MCP│  │           │            │
│                    │  │  Exa MCP    │  │           │            │
│                    │  │  Notion MCP │  │           │            │
│                    │  │  Slack MCP  │  │           │            │
│                    │  └─────────────┘  │           │            │
│                    └────────┬──────────┘           │            │
│                             │                      │            │
│                    ┌────────▼──────────┐            │            │
│                    │  Three-Tier       │◀───────────┘            │
│                    │  Prompt System    │                         │
│                    │  ┌─────────────┐  │                         │
│                    │  │ 1. Core     │  │                         │
│                    │  │ 2. Working  │  │                         │
│                    │  │ 3. Feedback │  │                         │
│                    │  └─────────────┘  │                         │
│                    └────────┬──────────┘                         │
│                             │                                    │
│                    ┌────────▼──────────┐                         │
│                    │   Eval Layer      │                         │
│                    │  (LLM reviewer)   │                         │
│                    └────────┬──────────┘                         │
│                             │                                    │
│                    ┌────────▼──────────┐                         │
│                    │  Output Queue     │                         │
│                    │  (Slack cards /   │                         │
│                    │   Notion rows /   │                         │
│                    │   ATS push)       │                         │
│                    └───────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
         ▲                              │
         │  accepts/rejects             │  deploys / updates
         │                             ▼
┌────────┴──────────┐         ┌─────────────────────┐
│  Recruiter UI     │         │  Claude Code /       │
│  (Slack Block Kit │         │  Claude Desktop      │
│   or Notion view) │         │  (dev only, not      │
└───────────────────┘         │   runtime)           │
                              └─────────────────────┘
```

**Key architectural principle:** Claude Code / Claude Desktop is the development and update mechanism only. The runtime system is fully cloud-deployed and does not depend on a local machine or an active desktop session.

---

## Requirements

### P0 — Infrastructure & Data Layer

#### INF-01: Cloud Compute Instance

**Description:** A persistent cloud compute instance (GCP Compute Engine or AWS EC2) hosts the agent runner, cron scheduler, and MCP server integrations.

**Acceptance Criteria:**

- [ ] Instance runs continuously without local machine dependency  
- [ ] Instance is accessible for deployment from Claude Code via SSH or CI/CD  
- [ ] Instance has IAM roles scoped to minimum required permissions (no wildcard IAM)  
- [ ] Instance is in a private VPC; external access only via load balancer or bastion  
- [ ] Startup script documented and version-controlled

**Technical Notes:**

- Start with a single `n2-standard-4` (GCP) or `t3.xlarge` (AWS) — 4 vCPU, 16GB RAM  
- Use a managed instance group for restart-on-failure behavior  
- Tag all resources with `project=calibratr-autopilot` for cost tracking

---

#### INF-02: Vertex AI / Model Garden Access (GCP path)

**Description:** GCP Vertex AI Model Garden provides access to Claude (Anthropic), OpenAI models, and open-source models (Mistral, Llama) under a single billing account with GCP-level security controls.

**Acceptance Criteria:**

- [ ] Vertex AI API enabled and tested end-to-end with at least one Claude call  
- [ ] All LLM calls route through Vertex AI (not direct Anthropic API) when on GCP  
- [ ] Model selection is configurable per task type (not hardcoded)  
- [ ] API keys and credentials stored in Secret Manager, not in code or environment variables

**Technical Notes:**

- Vertex AI provides model-level cost controls and audit logging not available in direct API  
- For AWS path: use Bedrock \+ IAM role assumption instead of Vertex AI

---

#### INF-03: PostgreSQL / Supabase Data Lake

**Description:** A structured relational database stores all intake data, candidate profiles, scores, calibration rules, run logs, and feedback signals.

**Acceptance Criteria:**

- [ ] Database schema is version-controlled and applied via migration files (Alembic or Flyway)  
- [ ] Row-Level Security (RLS) policies enforce client\_id isolation — no query returns data across clients without explicit join  
- [ ] All tables have `created_at`, `updated_at`, `client_id`, and `run_id` columns  
- [ ] Connection pooling configured (PgBouncer or Supabase built-in pooler)  
- [ ] Automated daily backups enabled with 30-day retention  
- [ ] Database credentials stored in Secret Manager

**Tables (minimum V1 schema — see Data Schema section):** `candidates`, `roles`, `clients`, `sourcing_runs`, `scores`, `calibration_rules`, `eval_results`, `outreach_queue`, `changelog`

---

#### INF-04: Non-AI Data Pull Scripts

**Description:** Initial data ingestion from Pin, Wrangle, and other sources is executed by plain Python/Node scripts — no LLM calls. LLMs are reserved for reasoning tasks only.

**Acceptance Criteria:**

- [ ] Data pull scripts are runnable independently of the agent (testable in isolation)  
- [ ] Scripts log every API call with timestamp, response code, and row count  
- [ ] Scripts handle rate limiting with exponential backoff  
- [ ] Scripts write to a `raw_ingest` staging table before any transformation  
- [ ] Total cost for a single full sourcing run (all roles, all clients) does not require LLM calls for the data pull phase

**Technical Notes:**

- Pin API → pulls candidate profiles matching search criteria, writes to `raw_ingest`  
- Wrangle API → pulls enriched profile data, writes to `raw_ingest`  
- Exa → pulls company and domain intelligence for Company DNA enrichment

---

#### INF-05: Cron Scheduler

**Description:** Cloud Scheduler (GCP) or EventBridge (AWS) triggers sourcing runs on per-role configurable schedules.

**Acceptance Criteria:**

- [ ] Each role has an independent cron schedule configurable in the DB (`roles.cron_schedule`)  
- [ ] Missed runs are retried once with exponential backoff; second failure triggers Slack alert  
- [ ] Overlapping runs for the same role are prevented (idempotency lock on `sourcing_runs`)  
- [ ] Run history (start time, end time, status, candidate count, LLM token cost) is written to `sourcing_runs` table

---

### P0 — Sourcing Agent Core

#### AGT-01: Agent Runtime

**Description:** The agent runner executes sourcing workflows using an agentic framework (Hermes, OpenClaw, or equivalent) that supports tool use, multi-step reasoning, and MCP integration.

**Acceptance Criteria:**

- [ ] Agent can be invoked via CLI and via HTTP endpoint (for cron trigger)  
- [ ] Agent maintains state across multi-step workflows without losing context  
- [ ] Agent has an `AGENT.md` file at the root of the codebase defining its purpose, tools, and behavioral rules  
- [ ] Agent session logs are persisted to `sourcing_runs.session_log` (truncated at 100K tokens)  
- [ ] Agent can call any registered MCP tool as a first-class action

---

#### AGT-02: MCP Tool Registry

**Description:** The agent has access to a registered set of MCP tools, each with a defined schema, error handling, and cost budget.

**Registered tools (V1):** | Tool | MCP Server | Purpose | |------|-----------|---------| | Pin search | `mcp.pin.com/mcp` | Source candidate profiles | | Pin accept/reject | `mcp.pin.com/mcp` | Push calibration signal | | Wrangle people search | `app.usewrangle.com/api/mcp/remote` | Enrich candidate data | | Exa web search | `mcp.exa.ai/mcp` | Company/domain intelligence | | Notion write | `mcp.notion.com/mcp` | Write candidate rows to Recruiting OS | | Slack post | `mcp.slack.com/mcp` | Post candidate review cards |

**Acceptance Criteria:**

- [ ] All MCP tool credentials stored in Secret Manager; agent loads at runtime  
- [ ] Each tool call is logged with input params, response, latency, and token cost  
- [ ] Tool failures return structured errors (not uncaught exceptions) and trigger retry logic  
- [ ] Each tool has a per-run budget cap configurable in `roles.tool_budgets` JSON column

---

#### AGT-03: Candidate Deduplication

**Description:** Before scoring, candidates are checked against the existing `candidates` table to prevent duplicate processing or outreach.

**Acceptance Criteria:**

- [ ] Dedup check runs on `linkedin_url` (primary) and `email` (secondary)  
- [ ] Candidates already in `outreach_queue` or `outreach_sent` are skipped  
- [ ] **Client conflict check**: candidates currently employed at active Calibratr clients (Clarasight, Tandem/Forus, Rebuild, Craniometrix, Clarion Health) are flagged with `status = HOLD_CLIENT_CONFLICT` — never proceed to outreach  
- [ ] Dedup check is a SQL query, not an LLM call

---

### P0 — Three-Tier Prompt Architecture

#### PRM-01: Tier 1 — Core Directive (Static)

**Description:** The agent's foundational system prompt. Defines identity, non-negotiable rules, output format contracts, and safety constraints. This prompt never changes.

**Contents (non-negotiable rules the agent must always follow):**

- Identity: "You are the Calibratr Sourcing Agent. Your purpose is to find qualified candidates for Calibratr's clients."  
- Safety: Never source or outreach to candidates at active client companies  
- Output contracts: All candidate evaluations must include score (0-100), rationale (max 200 words), signal\_tier (A/B/C), and confidence (low/medium/high)  
- Tool discipline: Log every tool call before executing it  
- Cost awareness: Prefer SQL and scripts over LLM calls for any task that does not require reasoning  
- Codebase rules: Always update `CHANGELOG.md` before completing any run; always update `TODO.md` after any run that creates new tasks

**Acceptance Criteria:**

- [ ] Tier 1 prompt is stored as a versioned file (`prompts/tier1_core.md`), not hardcoded in code  
- [ ] Tier 1 prompt changes require a PR review (not editable at runtime)  
- [ ] Tier 1 prompt version hash is written to every `sourcing_runs` record

---

#### PRM-02: Tier 2 — Working Prompt (Role-Scoped)

**Description:** The context-specific prompt that is assembled at the start of each sourcing run. Composed from the role's DNA, the client's Company DNA, intake form responses, and current search criteria.

**Contents (assembled per run):**

- Role DNA: title, level, required skills, experience bands, must-have vs. nice-to-have  
- Company DNA: stage, industry, culture signals, anti-patterns, recent news  
- Intake form: hiring manager priorities, specific asks from most recent briefing  
- Search configuration: geographies, target company tiers, seniority bounds, exclusion list

**Acceptance Criteria:**

- [ ] Tier 2 prompt is assembled from DB records, not written manually each time  
- [ ] Assembled prompt is stored in `sourcing_runs.working_prompt` for auditability  
- [ ] Token count of assembled Tier 2 prompt is logged; alert if \> 8,000 tokens  
- [ ] Role DNA and Company DNA are pulled from Notion Recruiting OS via Notion MCP at run time

---

#### PRM-03: Tier 3 — Feedback Rules Layer (Living Document)

**Description:** A structured rules file that is continuously updated based on recruiter accept/reject signals. This is where calibration lives. It is the only prompt tier that the agent can modify autonomously.

**Contents (structured format):**

```
[STRONG POSITIVE SIGNALS - weight: 1.5x]
- Candidate has deployed a clinical NLP model to production
- Former employee of: Epic, Veeva, Health Catalyst, Arcadia

[STRONG NEGATIVE SIGNALS - weight: 0.0x / disqualify]
- No demonstrated healthcare domain experience
- Only academic ML, no production deployment evidence

[SOFT NEGATIVE SIGNALS - weight: 0.7x]
- Current company > 5,000 employees (unless specific role at startup scale)

[PENDING CALIBRATION - needs more signal]
- Candidates from consulting backgrounds (mixed results so far)
```

**Acceptance Criteria:**

- [ ] Tier 3 file is stored at `prompts/clients/{client_id}/roles/{role_id}/feedback_rules.md`  
- [ ] Agent reads Tier 3 at the start of every scoring pass  
- [ ] After each recruiter accept/reject action, the agent proposes a Tier 3 update and writes it after human confirmation (V1) or autonomously (V2)  
- [ ] Every Tier 3 write is logged to `CHANGELOG.md` with timestamp, role, and diff  
- [ ] Tier 3 file is version-controlled in Git; agent commits changes via `git commit`

---

### P0 — Eval Layer

#### EVL-01: Output Validator

**Description:** Every significant LLM output is reviewed by a second LLM call that evaluates whether the output matches its stated intent. This is the mechanism for self-improvement and error prevention.

**What gets eval'd (V1):** | Output Type | Eval Prompt Focus | |------------|-------------------| | Candidate shortlist | "Do these candidates match the role DNA and Tier 3 rules? Flag any that shouldn't be here." | | Score rationale | "Is this rationale consistent with the signal tier assigned? Is any claim unsubstantiated?" | | Outreach draft | "Does this message comply with Calibratr's voice rules? Does it contain any prohibited phrases or fabricated claims?" | | Tier 3 rule update | "Is this proposed rule update consistent with the accept/reject signals that triggered it, or does it overgeneralize?" |

**Acceptance Criteria:**

- [ ] Eval runs on every LLM output before it is written to the output queue  
- [ ] Eval result (pass/flag/block) is written to `eval_results` table with reasoning  
- [ ] `block` result prevents the output from reaching the recruiter queue; triggers Slack alert to Taylor  
- [ ] `flag` result passes output to recruiter queue but marks it with a warning badge  
- [ ] Eval model is a different model than the generating model when feasible (e.g., generator \= Claude Sonnet, evaluator \= Gemini Flash or Claude Haiku) to reduce correlated errors  
- [ ] Eval token cost is tracked separately from generation token cost

---

#### EVL-02: Self-Improvement Loop

**Description:** Aggregate eval results feed a weekly summary that identifies patterns in flagged/blocked outputs, enabling systematic prompt improvements.

**Acceptance Criteria:**

- [ ] Weekly job runs every Monday morning, reads `eval_results` from the prior 7 days  
- [ ] Summary is posted to Calibratr Slack `#autopilot-evals` channel  
- [ ] Summary identifies the top 3 recurring failure modes with frequency counts  
- [ ] Agent proposes Tier 1 or Tier 3 changes to address top failure modes (requires human approval to apply)

---

### P0 — Changelog & Task Tracking

#### OPS-01: CHANGELOG.md

**Description:** The agent maintains a structured changelog file in the codebase root. Every significant state change is logged here.

**Format:**

```
## [YYYY-MM-DD HH:MM UTC] — {event_type}
**Run ID:** {run_id}
**Role:** {role_name} @ {client_name}
**Change:** {description}
**Before:** {previous state if applicable}
**After:** {new state}
**Author:** agent | human
```

**What gets logged:**

- Every Tier 3 rule write  
- Every schema migration  
- Every cron schedule change  
- Every model version change  
- Every failed run with error reason  
- Every manual override by a human

**Acceptance Criteria:**

- [ ] Agent writes to `CHANGELOG.md` before completing any run that changes system state  
- [ ] `CHANGELOG.md` is committed to Git after every write  
- [ ] `CHANGELOG.md` is included in the weekly eval summary  
- [ ] Nothing is considered "done" until it is in the changelog

---

#### OPS-02: TODO.md

**Description:** The agent maintains a living TODO list. Nothing is considered complete until it is checked off the TODO list.

**Format:**

```
## Active
- [ ] [ROLE: Clarion/Founding AE] Calibrate tier 3 after first 10 accepts — due: {date}
- [ ] [INFRA] Migrate candidate dedup to use email as primary key — blocker: schema migration

## Completed
- [x] [2026-06-08] Initial Pin MCP integration deployed and tested
```

**Acceptance Criteria:**

- [ ] Agent adds to TODO after any run that surfaces a new task  
- [ ] Agent marks items complete only after the task is verified (not just attempted)  
- [ ] TODO is reviewed in every weekly eval summary  
- [ ] Items with no progress for 14 days trigger a Slack alert

---

### P1 — Calibration & Feedback Loop

#### CAL-01: Recruiter Feedback Interface

**Description:** Recruiters review candidate cards in Slack or Notion and provide accept/reject/hold signals that feed back into Tier 3\.

**Slack Block Kit Card (per candidate):**

```
[Candidate Name] — [Signal Tier: A/B/C] — Score: {score}/100
[Company] | [Title] | [Location]
[Score Rationale — max 3 bullets]
[Proof of Work — 1-2 links]
─────────────────────────────────
[✅ Accept]  [❌ Reject]  [⏸ Hold]  [👁 View Full Profile]
[Optional: Rejection reason dropdown]
```

**Acceptance Criteria:**

- [ ] Card renders in Slack with all required fields  
- [ ] Accept/reject/hold action is captured and written to `candidates.recruiter_signal` within 5 seconds  
- [ ] Rejection reason (if provided) is written to `candidates.rejection_reason`  
- [ ] After 10 accepts or rejects for a role, the system proposes a Tier 3 update based on pattern analysis  
- [ ] Accept/reject actions are attributed to the recruiter's Slack user\_id

---

#### CAL-02: Calibration Run Trigger

**Description:** After sufficient feedback signal accumulates, a calibration job re-evaluates recently sourced candidates against the updated Tier 3 rules.

**Acceptance Criteria:**

- [ ] Calibration job triggers automatically after: 5 new accepts, 10 new rejects, or 7-day interval (whichever comes first)  
- [ ] Calibration run re-scores the 50 most recent `PENDING` candidates with updated Tier 3  
- [ ] Calibration results are written as a separate run with `run_type = CALIBRATION`  
- [ ] Score delta (before vs. after calibration) is logged and surfaced in the weekly eval summary

---

### P1 — ATS \+ Notion Sync

#### ATS-01: Notion Recruiting OS Write

**Description:** Accepted candidates are automatically written to the correct role page in Calibratr's Notion Recruiting OS.

**Acceptance Criteria:**

- [ ] Candidate row is created in the Candidates database with all standard fields populated  
- [ ] Candidate is linked to the correct Role and Client via Notion relations  
- [ ] Status is set to `Sourced - In Review` on creation  
- [ ] Duplicate prevention: check for existing Notion row before creating  
- [ ] Failed writes are logged to `outreach_queue.notion_sync_error` and retried once

---

#### ATS-02: ATS Push (Ashby / Greenhouse)

**Description:** For clients with active ATS integrations, accepted candidates are pushed to the ATS as a new prospect.

**Acceptance Criteria:**

- [ ] ATS push is configured per client (`clients.ats_integration` JSON column)  
- [ ] Push includes: name, LinkedIn URL, email (if available), source \= "Calibratr Autopilot", role\_id  
- [ ] ATS push is idempotent: duplicate check before write  
- [ ] Failed pushes are retried 3x with backoff; persistent failures alert via Slack

---

### P1 — Monitoring & Cost Controls

#### MON-01: Cost Dashboard

**Description:** Real-time and per-run cost tracking for all LLM calls, API calls, and compute.

**Acceptance Criteria:**

- [ ] Per-run token costs (input \+ output) logged by model and task type  
- [ ] Daily cost summary posted to `#autopilot-costs` Slack channel  
- [ ] Hard spend cap per run configurable in `roles.max_run_cost_usd` (default: $2.00)  
- [ ] If run exceeds 80% of cap, non-critical tasks (e.g., Company DNA enrichment) are skipped  
- [ ] Monthly cost forecast available via `/cost forecast` Slack command

---

#### MON-02: Health Alerts

**Description:** Proactive alerting for system failures, degraded performance, and anomalous behavior.

**Alert conditions:** | Condition | Severity | Channel | |-----------|----------|---------| | Sourcing run failed | P1 | `#autopilot-alerts` | | Eval layer blocked \> 3 outputs in one run | P1 | `#autopilot-alerts` | | DB connection error | P0 | `#autopilot-alerts` \+ PagerDuty | | Zero candidates sourced for a role (3 runs) | P2 | `#autopilot-evals` | | Cron job missed | P1 | `#autopilot-alerts` | | Cost cap exceeded | P1 | `#autopilot-alerts` |

---

### P2 — Future Considerations

These are architecturally informed decisions — design V1 to not block them, but do not build them now.

- **Zero-human-review mode**: Remove the human gate for outreach on roles with high-confidence calibration scores (Tier 3 maturity signal TBD). Architecture already supports this; it's a config flag change.  
- **Multi-tenant cloud isolation**: Per-client VPC or namespace isolation for enterprise security requirements.  
- **Real-time signal triggers**: Webhook from LinkedIn/Crunchbase on funding events, headcount changes, or job postings that trigger sourcing runs outside the cron schedule.  
- **Agent-initiated Tier 3 updates**: Today the agent proposes; human approves. V2: agent writes directly for low-risk rule updates (e.g., adding a company to a positive-signal list).  
- **Fine-tuned eval model**: Train a Calibratr-specific eval model on historical accept/reject data to replace the general-purpose LLM eval.  
- **Cross-role calibration transfer**: When a new role is created for an existing client, automatically seed its Tier 3 from the client's existing accepted-candidate patterns.

---

## Data Schema

### `candidates`

```sql
CREATE TABLE candidates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id UUID REFERENCES clients(id),
  role_id UUID REFERENCES roles(id),
  run_id UUID REFERENCES sourcing_runs(id),
  linkedin_url TEXT UNIQUE,
  email TEXT,
  full_name TEXT,
  current_title TEXT,
  current_company TEXT,
  location TEXT,
  raw_profile JSONB,              -- full API response, never modified
  enriched_profile JSONB,         -- post-Wrangle enrichment
  score INTEGER CHECK (score BETWEEN 0 AND 100),
  signal_tier CHAR(1) CHECK (signal_tier IN ('A','B','C')),
  score_rationale TEXT,
  confidence VARCHAR(10) CHECK (confidence IN ('low','medium','high')),
  recruiter_signal VARCHAR(20),   -- ACCEPT | REJECT | HOLD | NULL
  rejection_reason TEXT,
  status VARCHAR(30),             -- PENDING | IN_REVIEW | ACCEPTED | REJECTED | HOLD_CLIENT_CONFLICT | OUTREACH_QUEUED | OUTREACH_SENT
  notion_row_id TEXT,
  ats_prospect_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `sourcing_runs`

```sql
CREATE TABLE sourcing_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  role_id UUID REFERENCES roles(id),
  client_id UUID REFERENCES clients(id),
  run_type VARCHAR(20) DEFAULT 'SCHEDULED', -- SCHEDULED | MANUAL | CALIBRATION
  status VARCHAR(20),             -- RUNNING | COMPLETE | FAILED | PARTIAL
  candidates_sourced INTEGER DEFAULT 0,
  candidates_scored INTEGER DEFAULT 0,
  candidates_queued INTEGER DEFAULT 0,
  tier1_prompt_hash TEXT,
  working_prompt TEXT,
  session_log TEXT,
  total_llm_tokens INTEGER,
  total_cost_usd DECIMAL(10,4),
  error_message TEXT,
  started_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
```

### `calibration_rules` (mirrors Tier 3 file in structured form)

```sql
CREATE TABLE calibration_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  role_id UUID REFERENCES roles(id),
  rule_text TEXT NOT NULL,
  rule_type VARCHAR(20),          -- POSITIVE | NEGATIVE | SOFT_NEGATIVE | PENDING
  weight DECIMAL(3,2) DEFAULT 1.0,
  source VARCHAR(20),             -- AGENT_PROPOSED | HUMAN_ADDED
  trigger_signal_ids UUID[],      -- which accept/reject IDs triggered this rule
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `eval_results`

```sql
CREATE TABLE eval_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id UUID REFERENCES sourcing_runs(id),
  candidate_id UUID REFERENCES candidates(id),
  output_type VARCHAR(30),        -- SHORTLIST | SCORE_RATIONALE | OUTREACH_DRAFT | TIER3_UPDATE
  eval_model TEXT,
  result VARCHAR(10),             -- PASS | FLAG | BLOCK
  eval_reasoning TEXT,
  input_tokens INTEGER,
  output_tokens INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Success Metrics

### Leading Indicators (measure weekly from day 1\)

| Metric | Target | Measurement |
| :---- | :---- | :---- |
| Sourcing runs completed without failure | \> 95% success rate | `sourcing_runs.status` |
| Candidates per run per role | ≥ 15 per run | `sourcing_runs.candidates_sourced` |
| Eval pass rate (no block/flag) | \> 80% of outputs pass clean | `eval_results.result` |
| Recruiter signal latency | \< 24h from queue to accept/reject | `candidates.updated_at - created_at` |
| Cost per run | \< $2.00 per role per run | `sourcing_runs.total_cost_usd` |

### Lagging Indicators (measure at 30/60/90 days)

| Metric | Target | Measurement |
| :---- | :---- | :---- |
| Recruiter accept rate (% of queued candidates accepted) | \> 15% at day 30 → \> 25% at day 90 (calibration improving) | `candidates` accept/total ratio |
| Time-to-pipeline (days from role creation to first 10 candidates in review) | \< 3 days | `roles.created_at` vs. 10th `candidate.created_at` |
| Outreach conversion rate (accept → response) | \> 8% | `outreach_queue` response tracking |
| Recruiter hours saved per client per month | \> 20 hours | Recruiter self-report |
| Client retention impact | All Autopilot clients renew at 6 months | CRM |

---

## Open Questions

| \# | Question | Owner | Blocking? |
| :---- | :---- | :---- | :---- |
| OQ-01 | Which cloud provider — GCP or AWS? GCP recommended for Vertex AI access; AWS if team has existing AWS familiarity. | Taylor \+ Technical Reviewer | **Yes — before infra setup** |
| OQ-02 | Which agent runtime — Hermes, OpenClaw, or custom Claude API loop? Answer affects architecture significantly. | Technical Reviewer | **Yes — before agent build** |
| OQ-03 | Supabase vs. self-managed Postgres vs. BigQuery? Supabase simplest for V1; BigQuery needed if query volume \> 10M rows/month. | Technical Reviewer | **Yes — before schema migration** |
| OQ-04 | Who is the technical reviewer? This person needs to understand both cloud infrastructure AND agentic LLM systems. | Taylor | **Yes — before any build** |
| OQ-05 | What is the Autopilot product's pricing model? This affects the cost cap per run (OQ-05 affects MON-01 thresholds). | Taylor | No — can finalize post-V1 |
| OQ-06 | Should Tier 3 updates in V1 require human approval before writing, or can the agent write autonomously? Default recommendation: human approval in V1. | Taylor | No — can start with human-approval as default |
| OQ-07 | Pin MCP job ID `0e9f6c7a` (Clarion Deployment Strategist) is stuck. Is this a Pin API issue or a data issue? | Dave / Pin support | No — but should be resolved before Clarion Autopilot launch |
| OQ-08 | What is the target cadence for each Clarion role? Default proposal: Founding AE \+ Forward Deployed Engineer every 4h; others every 12h. | Taylor \+ Ryan Gallagher | No |
| OQ-09 | Security posture for MCP credentials stored in cloud Secret Manager — who has access, and is there an audit log requirement? | Technical Reviewer | No — but must be addressed before client data in system |

---

## Timeline & Phasing

### Phase 0 — Technical Review & Cloud Setup (Weeks 1–2)

**Exit criteria:** Technical reviewer signed off; cloud account created; Vertex AI / Bedrock tested; Postgres schema migrated; CI/CD pipeline for code deploy from Claude Code to cloud instance.

- Hire / engage technical reviewer (must understand infra \+ agentic AI)  
- Select cloud provider (GCP or AWS)  
- Set up VPC, compute instance, Secret Manager  
- Enable Vertex AI (GCP) or Bedrock (AWS)  
- Stand up Postgres with base schema  
- Validate MCP connectivity from cloud instance (Pin, Wrangle, Notion, Slack)

### Phase 1 — Agent Core (Weeks 3–5)

**Exit criteria:** Agent can complete a single sourcing run end-to-end for one role (Clarion Founding AE) without human intervention, and writes results to Notion \+ Slack.

- Build agent runtime \+ MCP tool registry  
- Implement three-tier prompt system  
- Build non-AI data pull scripts  
- Implement candidate dedup (including client-conflict check)  
- Build Slack Block Kit review card  
- Implement `CHANGELOG.md` and `TODO.md` discipline in agent  
- Deploy cron scheduler for single role

### Phase 2 — Eval Layer \+ Calibration (Weeks 6–8)

**Exit criteria:** Eval layer is running on all outputs; Tier 3 is updating based on recruiter signals; cost per run is tracked and bounded.

- Build eval layer (EVL-01, EVL-02)  
- Build recruiter feedback capture (CAL-01)  
- Build calibration job (CAL-02)  
- Build cost tracking (MON-01)  
- Build health alerts (MON-02)  
- Expand to all 5 Clarion roles

### Phase 3 — ATS Sync \+ Multi-Client Rollout (Weeks 9–12)

**Exit criteria:** Clarion fully on Autopilot; at least one additional client onboarded to Autopilot; ATS sync working.

- Build Notion Recruiting OS sync (ATS-01)  
- Build ATS push (ATS-02)  
- Onboard second client to Autopilot  
- Weekly eval summary automation  
- Retrospective \+ V2 scoping

---

## Technical Reviewer Brief

**This section is for the person conducting the technical review.**

Calibratr is building a cloud-deployed agentic sourcing system. The architecture described in this spec is a starting recommendation based on product requirements. The technical reviewer's job is to challenge and improve it before significant build begins.

**Key areas for reviewer scrutiny:**

1. **Security**: Are the IAM roles, secret management, VPC configuration, and RLS policies sufficient for handling candidate PII (names, emails, LinkedIn URLs)? What compliance requirements apply (SOC 2, GDPR)?  
     
2. **Cost architecture**: Is the separation of LLM vs. non-LLM tasks sufficient to keep per-run costs under $2.00? What are the real token costs at scale (100 roles × 4 runs/day)?  
     
3. **Agent runtime choice**: Is Hermes / OpenClaw the right choice for a production cloud deployment, or is a custom Claude API loop more maintainable? What are the failure modes of each?  
     
4. **Database choice**: Is Supabase the right call for V1 given the RLS requirements and expected query patterns? What are the migration pain points if we need to move to BigQuery later?  
     
5. **MCP reliability**: MCP servers (Pin, Wrangle, Notion) are third-party. What happens when they go down? Is the retry/fallback logic sufficient, or do we need a local cache layer?  
     
6. **Agentic failure modes**: What happens if the agent enters a reasoning loop and burns $50 in tokens on a single run? What circuit breakers are needed beyond the per-run cost cap?  
     
7. **Changelog / Git discipline**: Is the pattern of the agent committing to Git via `git commit` during runtime viable and safe? What are the risks of agent-authored commits in a production repo?

**Deliverable expected from reviewer:** A written review document flagging any of the above with severity (blocking / high / medium / low) and a recommended resolution for each blocking item.

---

*This spec is a living document. All changes must be logged in `CHANGELOG.md`. No section is finalized until the technical reviewer has signed off on P0 requirements.*  
