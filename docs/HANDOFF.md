# Handoff: MC_Volha1 team project (Spark Lakehouse course)

Context for an AI coding assistant helping a team member continue this project. Last updated 2026-11-26.

## Project in one paragraph
University course project. The team builds a Spark + Delta Lake "lakehouse" for a fictional consulting company (CorpTech). Raw daily files go through Bronze (raw copy + `_batch_date`, `_ingested_at`, `_source_file`, idempotent per day) → Silver (clean, dedup, quarantine bad rows with `dq_reasons`, SCD2 employees, FX forward-fill, CRM skew handling) → Gold (5 marts) → Streamlit/DuckDB dashboard (4 views), orchestrated by an Airflow DAG with DQ gates. The full spec is in `docs/STUDENT_ASSIGNMENT.md` and the expected counts are in `docs/SANITY_CHECKS.md`. Grading also runs an unseen 70-day holdout (2026-07-20 → 2026-09-27), so every job must be incremental and idempotent. Each member's contributions must be visible in the GitHub history: own branches, own commits, PRs, reviews.

## Repos
- **Team repo (work here):** https://github.com/K1NGR4T04/MC_Volha1. It's a fork of `smoothgrinder/spark-lakehouse-starter` and the owner is K1NGR4T04. Clone it with `gh repo clone K1NGR4T04/MC_Volha1`, which also adds the prof's repo as `upstream`. Work on feature branches (e.g. `feat/silver-hr`) and open PRs into `main`.
- **Prof's plugin/skills repo:** https://github.com/smoothgrinder/spark-lakehouse-marketplace. It has AI-assistant plugins: `lakehouse-dev-ops` (working: `stack-doctor` health check plus guardrail rules) and 3 empty TODO plugins (`spark-medallion-coach`, `dq-and-airflow-ops`, `lakehouse-reporting-ml`). The team is meant to fill those in, which counts toward grading.
- **"Phase 0"** is not defined anywhere in either repo (searched for phase/milestone/sprint/week). It's assumed to mean the setup step. Ask the prof to confirm.

## Status
- micmikemic's machine: everything below is done (team repo cloned, `.env` filled, data downloaded, Docker stack running, tests passing). Each teammate must repeat the setup on their own laptop.
- **The team decided to skip Bronze for now.** No pipeline code has been written yet, nothing is committed, and the repo matches the starter.

## Setup per teammate (with fixes for the problems we hit)
1. Install: Git, Docker Desktop, `uv` (`uv python install 3.11`), `gh` (and `gh auth login`).
2. `cp .env.example .env` and fill in the 5 `R2_*` values from the prof. **Fix: set `R2_PREFIX=course-2026/archives`.** The documented `course-2026` fails with `NoSuchKey`, because the manifest lives in `archives/` but the script resolves files relative to the prefix. `.env` and `data/` are git-ignored, so never commit or paste the keys.
3. `./scripts/download_data.sh` puts ~179 MB into `data/landing/` (10 datasets). Re-check later with `--verify-only`.
4. Docker: the course asks for ≥8 GB RAM in Docker settings. On an 8 GB Mac that's impossible, so it ran on ~4 GB, which is fine for the 14-day slice and tight for the 200-day runs. Start with `docker compose up -d --build` (first build 10–25 min).
5. Tests: `docker compose run --rm spark-master pytest -v` should give 9 passed.
6. **Fix: `spark-submit` isn't on the container PATH.** The README command fails, so use the full path:
   `docker compose exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/workspace/spark_jobs/bronze/ingest_clients.py --batch-date 2026-01-01` should print 300 rows.
7. UIs: Airflow http://localhost:8080 (admin/admin), Spark http://localhost:8081, History http://localhost:18080, Streamlit http://localhost:8501.
8. Claude Code plugin (optional). **Fix: use the full URL**, because the short `owner/repo` form errors:
   `claude plugin marketplace add https://github.com/smoothgrinder/spark-lakehouse-marketplace.git`
   `claude plugin install lakehouse-dev-ops@spark-lakehouse-marketplace`
   (Re-running install when it's already installed also shows "unknown error". That's harmless.)

## Data profile (200 days, 2026-01-01 → 2026-07-19)
1,200 employees, 300 clients, 800 projects. Rows: employees 240k (full snapshot every day), timesheets 507k, invoices 62k, expenses 101k, fx_rates 142 weekday files (6 pairs each), crm_events 601k, employee_surveys 2,059 (only 2 waves: Jan 9–16 and Feb 20–27).

**SANITY_CHECKS.md is partly wrong.** Totals and distinct-ID counts match (clients 300, projects 800, employees 16,800, fx 60, invoices 4,445, orphans 43, open invoices 1,352, CRM 42,507 with VIP 19,151 = 45.05%, distinct ts_id 33,051, exp_id 6,552, survey_id 746). But most anomaly counts are about **2× the documented value** in the real data (14-day slice):
| metric | doc | actual |
|---|---|---|
| timesheets rows | 34,675 | 36,368 |
| ts null project / hours≤0 / hours>24 | 1,742 / 876 / 908 | 3,674 / 1,770 / 1,906 |
| invoices EU-format issued_at | 211 | 441 |
| credit notes | 232 | 455 |
| expenses rows | 6,894 | 7,191 |
| surveys rows / out-of-range / blank written_answer | 760 / 18 / 40 | 787 / 40 / 63 |
The dataset was probably regenerated without updating the doc. Tell the prof, and don't hard-code the doc numbers in tests.

**Known issues (as the spec says):**
- Timesheets: dups (some IDs ×3), and near-dups differ only in `hours` (≤1.5 h). Hours range −8 to 49.25. 4,035 post-termination ("zombie") rows over 200 days, but 0 in the first 14 days.
- Expenses: near-dups differ only in `amount`.
- Invoices: ~10% `DD.MM.YYYY` dates, ~10% credit notes (negative amounts are valid), ~1% `PRJ_ORPHAN_*`.

**Not mentioned in the spec:**
- 5,180 invoices have `paid_at` after 2026-07-19, i.e. future payment dates that are already in the issue-day file. This is a leakage / point-in-time concern for open-invoice KPIs and ML.
- `projects.status` is a static snapshot. "Planned" projects have 109k timesheet rows and 10k invoices.
- 156 active employees report to a terminated manager.
- 2,707 EU dates are ambiguous (day ≤12). Checked against the partition date: **DD.MM is correct**.
- CRM timestamps are all UTC (`Z`). Local-timezone casting shifts 39k events to the wrong day, so set `spark.sql.session.timeZone=UTC`.
- eNPS is very negative: 75% detractors, about −69. That's real, not a bug. `enps_score` always equals the Q01 answer, and 11 submissions have out-of-range values (−1/11).

**Clean, no extra work:** no unknown FKs (clients, employees, managers, non-orphan projects), FX has only weekend gaps and consistent inverse rates, CRM payload keys are cleanly separated per event_type, there are 113 departures with consistent termination data (100 voluntary / 13 involuntary, starting in February), and 86 employees changed department/role/rate (for SCD2). No reactivations, no hire-date changes.

## Rules for the assistant
- Never print, commit or paste `.env` / `R2_*` values. Report only SET/MISSING. Never `git add -f`.
- Run Spark and pytest inside Docker. Run `uv run --no-project --with ruff ruff check .` and `ruff format --check .` before commits.
- Work on branches and open PRs so each member's work shows up in GitHub.

## Open next steps
- Confirm with the prof what "Phase 0" means and report the SANITY_CHECKS mismatch, plus the two setup gotchas (`R2_PREFIX`, `spark-submit` path).
- Agree on the role split (HR / Finance / CRM+surveys / Gold+dashboard / Airflow+DQ+CI+skills).
- Data profile is in `docs/DATA_PROFILE.md`.
- Start Silver/other work per the team's decision (Bronze skipped for now).
