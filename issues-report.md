# HCPI — Reported Issues Report

A grouped, prioritised view of all reported issues. Effort sized as **S** (1–3 days), **M** (3–10 days), **L** (open-ended). Branch column shows where the change ships; "shared" = `18.0` and merged into `ug_18`, `ke_18_mtnce`, `tz_18_mtnce`, `zar_18`.

Source: [reported-issues.md](reported-issues.md). Detailed breakdown: [issue-triage.md](issue-triage.md).

---

## 1. Dashboard & Reporting

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 1.1 | Month / year dropdown to filter displayed figures | `hcpi_dashboard` | M | shared |
| 1.2 | Show National CPI value at top; raise historical trend above the fold | `hcpi_dashboard` | M | shared |
| 1.3 | Show all 12 Divisions on screen | `hcpi_dashboard`, `hcpi_index` data | S | shared |
| 1.4 | Annual % change + Core / Non-core indicator cards | `hcpi_dashboard` | S | shared |
| 1.5 | Public dashboard access (no login) | new `hcpi_dashboard_public` route | M | shared |
| 1.6 | **Uganda / UBOS:** national figures missing on dashboard | data investigation first | M | `ug_18` |
| 1.7 | **Kenya / KNBS:** sector figures showing 0 | data investigation first | M | `ke_18_mtnce` |

---

## 2. Data Collection & Validation

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 2.1 | List/report of flagged outliers and inliers | `hcpi_data_collection` | S | shared |
| 2.2 | Indicator + report for items temporarily missing 3+ months | `hcpi_outlet` | S | shared |
| 2.3 | Data Collection Analytics — counts by state, by outlet / centre / national | `hcpi_data_collection` | M | shared |
| 2.4 | **KNBS:** progress bar shows 100% during collection but drops to 67% after | `hcpi_data_collection` (progress compute) | M | shared (possibly `ke_18_mtnce`) |
| 2.5 | Three-tier outliers — extreme (24mo) / outlier (12mo) / mild (6mo) | `hcpi_data_collection` (`run_turkey`) | M | shared |
| 2.6 | Smarter inliers — check 12 consecutive months and identical change pattern | `hcpi_data_collection` (`validate_inliers`) | M | shared |
| 2.7 | Flag changes in Centrally Administered Prices | `hcpi_item` + small dashboard | M | shared |
| 2.8 | Collected data must reliably reach permanent observations | `hcpi_data_collection` (`action_approve`) | M | shared |
| 2.9 | **NBS:** confirm offline tablet behaviour when server unreachable | test + docs | S | docs |

---

## 3. Computations & Index Logic

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 3.1 | Formal review of computations process | already drafted in [HCPI_COMPUTATION_FLOW.md](HCPI_COMPUTATION_FLOW.md) | S | docs |
| 3.2 | "Show the math" / in-built calculator view for an index value (KNBS) | `hcpi_index` (read-only diagnostics view) | M | shared |
| 3.3 | Period + Division filters on index update wizards | `hcpi_index` wizards | M | shared |
| 3.4 | Provisional Index — carry forward previous month values | `hcpi_index` (new wizard) + methodology sign-off | L | shared |
| 3.5 | Deleting update history leaves advisory lock; cannot restart | `hcpi_index` queue-job lock cleanup | M | shared |
| 3.6 | **KNBS:** "standard and non-standard of weights" — meaning unclear | TBD | TBD | TBD |

---

## 4. Performance & Infrastructure

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 4.1 | **KNBS:** system slow as users / data grow | profiling first | L | shared + per-instance config |
| 4.2 | Large log tables (`mail.message`, `ir_logging`) inflating DB | audit `tracking=True`; add retention cron | M | shared |

---

## 5. Cross-Country & Governance

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 5.1 | **EAC:** review syncing of data to Secretariat Dashboard | `hcpi_secretariat_hub` + per-country integration users | M | `secretariat` |
| 5.2 | Standardise item codes across countries | `hcpi_item` schema + EAC-wide mapping | L | shared + data per country |
| 5.3 | Document XML/RPC endpoints | docs page | S | docs |
| 5.4 | Transfer mobile app from Kola to EAC Google Play account | admin / account transfer | S | N/A |

---

## 6. Mobile App (separate Flutter repo)

| # | Issue | Module / Area | Effort | Branch |
|---|---|---|---|---|
| 6.1 | **NBS:** GPS not captured during data collection | Flutter app code | L | mobile app repo |

---

## 7. Already Resolved

| # | Issue | Notes |
|---|---|---|
| 7.1 | **OCGS:** unable to update EA Indices | Resolved per source report |

---

## Recommended sequencing

**Sprint 1 — Dashboard refresh (one shared PR):**
1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3

**Sprint 2 — Stability pass:**
2.4 (progress bug), 4.1 + 4.2 (slowness + log retention together), 3.5 (lock cleanup)

**Sprint 3 — Validation enhancements (methodology sign-off needed):**
2.5, 2.6, 2.7, 3.2, 3.3

**Investigation backlog (do before estimating further):**
1.6, 1.7 (need DB snapshots), 5.1 (audit sync), 3.6 (clarify with KNBS)

**Governance / non-engineering:**
3.1 (publish review), 3.4 (methodology), 5.2 (cross-country code agreement), 5.4 (Google Play transfer)
