# Landing Data Profile & Inconsistencies

This profile covers the 200-day development dataset (`2026-01-01` – `2026-07-19`) in `data/landing/`. It was produced with DuckDB queries over the raw files on 2026-11-26.

> The data itself is **not** in Git. Each teammate downloads it with `./scripts/download_data.sh`. Set `R2_PREFIX=course-2026/archives` in `.env`, because the documented `course-2026` fails with `NoSuchKey`.

## 1. What the data is

The fictional company CorpTech has **1,200 employees, 300 clients and 800 projects**.

| Dataset | Format | Load type | Rows (200 d) | Content | Used for |
|---|---|---|---:|---|---|
| `clients` | CSV | full load | 300 | `client_id`, name, tier (Standard/Silver/Gold/Enterprise) | dimension |
| `projects` | CSV | full load | 800 | project → client, `budget_eur`, status | dimension, profitability |
| `survey_questions` | JSON array | full load | 10 | Q01 eNPS (0–10), Q02–Q10 Likert drivers (1–5) | survey metadata |
| `employees` | CSV | **daily full snapshot** (1,200/day) | 240,000 | dept, role, seniority, hourly rate, hire/termination, churn type | SCD Type 2 |
| `timesheets` | **Parquet** | daily | 507,382 | `emp_id`, `project_id`, `work_date`, `hours` | labor cost |
| `fx_rates` | JSON array | **weekdays only** (142 files) | 852 | 6 pairs EUR/USD/CZK | FX calendar spine + LOCF |
| `invoices` | JSONL | daily | 62,480 | amount, currency, `issued_at`, `paid_at` (raw strings) | revenue |
| `expenses` | CSV | daily | 101,491 | dept, category, amount, currency | department finance |
| `crm_events` | JSONL, nested `payload` | daily | 601,497 | 5 event types, payload schema varies by type | client engagement, skew |
| `employee_surveys` | JSONL, `answers` array | 16 days (2 waves) | 2,059 | eNPS + 10 answers per submission | eNPS analytics |

CRM `payload` keys per `event_type` (clean, no overlap):

| event_type | payload keys |
|---|---|
| `page_view` | `device`, `duration_sec`, `path`, `referrer` |
| `api_call` | `endpoint`, `latency_ms`, `region`, `status_code` (200 or 429) |
| `support_ticket` | `module`, `severity` (low/medium/high/critical), `sla_hours`, `ticket_id` |
| `meeting_logged` | `attendees_count`, `meeting_type`, `next_step_days`, `outcome` |
| `contract_renewal` | `arr_delta_eur`, `opportunity_id`, `probability`, `stage` |

The survey waves are **Wave 1: 2026-01-09 → 01-16** (995 respondents, ~83%) and **Wave 2: 2026-02-20 → 02-27** (959, ~80%). There are no further waves in the dev window.

## 2. `SANITY_CHECKS.md` does not match the data ⚠️

Totals and distinct-ID counts match the doc exactly. These all match: clients 300, projects 800, employees 16,800, fx 60 rows / 10 partitions, invoices 4,445, orphans 43, open invoices 1,352, CRM 42,507, VIP 19,151 (45.05%), distinct `ts_id` 33,051, `exp_id` 6,552, `survey_id` 746.

Most **anomaly counts are ~2× the documented values** (first 14 days):

| Metric (Days 1–14) | `SANITY_CHECKS.md` | Actual |
|---|---:|---:|
| timesheets raw rows | 34,675 | **36,368** |
| timesheets extra duplicate rows | 1,624 | **3,317** |
| timesheets `project_id IS NULL` | 1,742 | **3,674** |
| timesheets `hours <= 0` | 876 | **1,770** |
| timesheets `hours > 24` | 908 | **1,906** |
| invoices `DD.MM.YYYY` in `issued_at` | 211 | **441** |
| invoices credit notes (`amount < 0`) | 232 | **455** |
| expenses raw rows | 6,894 | **7,191** |
| surveys raw rows | 760 | **787** |
| survey answers out of range (−1 / 11) | 18 | **40** |
| survey answers blank/NULL `written_answer` | 40 | **63** |

The likely cause is that the dataset was regenerated with higher anomaly rates and the doc wasn't updated. **Don't hard-code these doc numbers in tests.** This has been reported to the instructor (TODO: confirm).

## 3. Known upstream issues (confirmed, as described in the assignment)

- **Timesheets:** 44,874 duplicated `ts_id`s (43,672 ×2, 1,202 ×3). Near-duplicates differ **only in `hours`** (avg Δ 0.88 h, max 1.5 h). `hours` ranges from −8 to 49.25. There are **4,035 post-termination ("zombie") rows**, none of them in Days 1–14, so a 14-day test won't catch them. There are 6,948 weekend rows, which are valid.
- **Expenses:** 9,003 duplicated `exp_id`s. Near-duplicates differ **only in `amount`**.
- **Surveys:** duplicate submissions (identical content), 11 out-of-range `enps_score` values, and out-of-range answers spread across all of Q01–Q10. `enps_score` always equals the Q01 answer. `overall_comment` is never blank.
- **Invoices:** ~10% have `DD.MM.YYYY` dates in `issued_at` / `paid_at`. Credit notes make up 10% (6,250), which is valid and must be kept. 1.04% are orphans (`PRJ_ORPHAN_*`, 648). 18,810 open invoices (`paid_at` NULL) are valid.
- **FX:** files exist only for weekdays. There are no holiday gaps, no weekend files, and inverse pairs are consistent (rate × inverse ≈ 1).

## 4. Inconsistencies NOT mentioned in the assignment

1. **Invoices contain future payment dates.** 5,180 invoices issued in the dev window have `paid_at` **after 2026-07-19**, and the payment date is already present in the issue-day file (pay lag is 3–45 days, median 24). This is a point-in-time / leakage risk for "open invoice" KPIs and any ML features built from payments.
2. **`projects.status` is a static snapshot that contradicts activity.** Projects marked `Planned` have 108,929 timesheet rows and 10,085 invoices, and `Completed` projects still get some. Don't use `status` to filter activity.
3. **156 active employees report to a terminated manager** (as of 2026-07-19).
4. **Ambiguous European dates.** 2,707 `DD.MM.YYYY` values have day ≤ 12 (e.g. `05.03.2026`). We validated against the partition date: **`DD.MM` is correct** for all 6,436 EU `issued_at` values (`MM.DD` matches only 215 by coincidence).
5. **Timezone trap.** All CRM timestamps are UTC (`Z`). Casting with a local timezone (e.g. Europe/Prague) shifts ~39k events to the wrong day, so set `spark.sql.session.timeZone=UTC`.
6. **Very negative eNPS.** 75% of all submissions are Detractors (1,546), 18% Passives, 6% Promoters, so eNPS ≈ −69. That's real, not a bug, and worth highlighting in the HR dashboard.
7. **Only 2 survey waves** in 200 days, so `gold_hr_enps_analytics` will have just 2 `survey_wave` values in dev.

## 5. Clean, no handling needed

- No unknown FKs: every `client_id` / `emp_id` / `manager_id` / non-orphan `project_id` resolves. Invoice `client_id` always matches the project's client.
- `employees`: always 1,200 rows/day, no duplicate `emp_id` per snapshot, `snapshot_date` = partition, no reactivations, `hire_date` never changes.
- **Departures:** 113 (100 voluntary / 13 involuntary), starting in February (Feb 2, Mar 24, Apr 23, May 29, Jun 23, Jul 12). `termination_date` always equals the first inactive snapshot.
- **SCD2 input:** 86 employees changed attributes (46 department, 46 role, 52 hourly rate). Seniority, work mode, manager and promotion date never change.
- No duplicate `inv_id` or `event_id`. Partition date = record date for every daily dataset.
- CRM: `VIP_CORP_001` share is 45.03% over all 200 days. Latency is 15–864 ms. There are no negative `arr_delta_eur` values.
