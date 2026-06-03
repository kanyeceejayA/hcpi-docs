# Reported Issues — Triage

A reordering of [`reported-issues.md`](reported-issues.md) by ease of implementation, with what each issue actually means, where the fix lives in code, and which branch it should ship on.

## How to read this

Each issue lists:

- **What it is** — the problem in plain language.
- **Where it lives** — the module / file / model the fix touches.
- **Branch** — where the change is committed.
- **Effort** — rough sizing.

### Branch model (recap)

The HCPI codebase is split between **shared modules** that are identical across every country, and **country-overlay modules** that only exist on one country's branch. The split is enforced by a per-country branch model:

| Layer | Branch | Modules |
|---|---|---|
| Shared base (computation, dashboard, data-collection workflow, index math) | `18.0` | `hcpi_computation`, `hcpi_coicop`, `hcpi_item`, `hcpi_outlet`, `hcpi_data_collection`, `hcpi_index`, `hcpi_dashboard` |
| Uganda overlay | `ug_18` | `ug_location`, `ug_outlet`, `ug_base_import` |
| Kenya overlay | `ke_18_mtnce` | `ke_location`, `ke_outlet`, `ke_data_collection`, `ke_base_import` |
| Tanzania overlay | `tz_18_mtnce` | `tz_location`, `tz_outlet`, `tz_data_collection`, `tz_base_import` |
| Zanzibar overlay | `zar_18` | `zar_location`, `zar_outlet`, `zar_data_collection` |
| Regional aggregator | `secretariat` | `hcpi_secretariat_hub` |
| Mobile (Flutter) | separate repo | not in this codebase |

**Rule of thumb:** if the fix touches calculation, validation, the dashboard, or workflow that all countries share, it lands on `18.0` and gets merged forward into each country branch. If it touches geography or country-specific workflow, it lands on that country's branch only.

---

## Tier 1 — Quick wins (1–3 days each)

Well-scoped, low risk, mostly additive. Good candidates to bundle into a single shared PR.

### T1.1 — Document XML/RPC endpoints

**What it is:** The Flutter mobile app talks to the server entirely through Odoo's standard `/xmlrpc/2/object` endpoint, calling methods on `hcpi.data.collection`, `hcpi.data.observation.line`, etc. There is no central reference of *which* methods are mobile-facing, what they accept, and what they return.

**Where it lives:** [`hcpi-docs`](.) — a new page under [`docs/modules/`](docs/modules/) listing the mobile-callable methods (search `hcpi_data_collection_mobile_app/models/` for `_inherit` overrides — those are the public surface).

**Branch:** `main` of this docs repo.
**Effort:** Documentation only.

---

### T1.2 — Transfer Mobile App to EAC-Controlled Google Play

**What it is:** The Flutter app is published under Kola's Google Play account. EAC wants ownership.

**Where it lives:** Google Play Console — admin task, not code.

**Branch:** N/A.
**Effort:** Coordination with Kola + Google Play account transfer flow.

---

### T1.3 — Report of outliers and inliers

**What it is:** The Tukey outlier algorithm and inlier check already flag suspicious price lines, but there's no easy way for a supervisor to *see the list* of flags in a given month.

**Where it lives:** `hcpi_data_collection`. The flags `is_outlier` and `is_inlier` already exist on `hcpi.data.collection.line` (set by `run_turkey()` and `validate_inliers()` in `hcpi_data_validation.py`). The work is purely a new list view + filter + export action.

**Branch:** `18.0` → merge to all four country branches.
**Effort:** Small — list view, search view, exporter.

---

### T1.4 — "Temporarily missing for 3+ months" indicator and report

**What it is:** When an outlet stops providing a price, the system imputes it; but if it stays missing for many months the imputation becomes unreliable. There's no surface telling a supervisor *"this item has been missing for N months."*

**Where it lives:** `hcpi_outlet` — add a `consecutive_missing_months` computed field on `hcpi.outlet.item` (read from `hcpi.outlet.item.observation` history). Then a list view filtering for ≥ 3 months.

**Branch:** `18.0` → merge to all four country branches.
**Effort:** Small — one compute + one view.

---

### T1.5 — Show all Divisions on the Dashboard

**What it is:** The dashboard shows only the COICOP classes whose `dashboard_display` flag is set. By default several Divisions are hidden, so the dashboard looks incomplete.

**Where it lives:** `hcpi_index` (toggle the data records — set `dashboard_display=True` on every division) and `hcpi_dashboard` (CSS in `static/src/css/custom-13.css` may need to fit all 12 divisions on one screen).

**Branch:** `18.0` → merge to all four country branches.
**Effort:** Small. Mostly data, with light CSS work.

---

### T1.6 — Confirm offline data collection works when server is offline

**What it is:** NBS asked whether collectors can keep working when the server is unreachable. This is a "test and document" task — the offline path already exists in the mobile app; we just need to verify it end-to-end and write it up.

**Where it lives:** Test against a sandbox, then add a section to [`docs/modules/data-collection.md`](docs/modules/data-collection.md).

**Branch:** `main` of docs repo.
**Effort:** Small — one test session and a short writeup.

---

### T1.7 — Annual % change, Core and Non-core on indicator cards

**What it is:** The dashboard headlines show point-in-time values; statisticians want **year-on-year change** and the **Core / Non-core split** as headline indicators (the standard inflation-report numbers).

**Where it lives:** `hcpi_dashboard` (extend `dashboard_hooks()` to compute Y-on-Y from the existing time series; add card components). "Core" is a special-index category — confirm it exists as an `hcpi.special.index.category` record; if not, add the data record in `hcpi_index/data/`.

**Branch:** `18.0` → merge to all four country branches.
**Effort:** Small-to-medium — one server method, one or two UI cards.

---

## Tier 2 — Medium effort, well-defined (3–10 days each)

Real engineering work, but the scope is clear and the design isn't in doubt.

### T2.1 — Month/year dropdown on the dashboard

**What it is:** Right now the dashboard hard-codes a rolling 12-month / 24-month window. Users want to pick a specific month and year to view.

**Where it lives:** `hcpi_dashboard`. `dashboard_hooks()` currently uses fixed `relativedelta(months=...)`. Extend it to accept `start_month` / `end_month` parameters; add `<select>` inputs in the OWL component.

**Branch:** `18.0` → all four country branches.
**Effort:** Backend method change + UI controls + re-rendering on change.

---

### T2.2 — National CPI value at the top, historical trend above the fold

**What it is:** Cosmetic but important: the headline national CPI value should be the first thing visible, and the historical trend chart should be lifted higher on the page. Currently buried below other cards.

**Where it lives:** `hcpi_dashboard` — `static/src/xml/hcpi_dashboard.xml` (layout), `static/src/css/custom-13.css` (visual hierarchy), and possibly one extra series in `dashboard_hooks()` for the headline value.

**Branch:** `18.0` → all four country branches.
**Effort:** Layout work + one server-side addition.

---

### T2.3 — Data Collection Analytics (counts by state × outlet / centre / national)

**What it is:** A status report: for a given period, how many questionnaires are in `draft`, `survey`, `standardization`, `validation`, `done`, broken down by collection-centre / region / national totals.

**Where it lives:** `hcpi_data_collection` — read-only aggregation over `hcpi.data.collection.state`. Each country's overlay already defines the right leaf-location (parish / zone / district / collection-centre), so the group-by uses the same field across countries.

**Branch:** `18.0` → all four country branches.
**Effort:** New report model + pivot view.

---

### T2.4 — Smarter inlier check (12 consecutive months + same-direction change)

**What it is:** Today an "inlier" is flagged if the price is *identical* for 12 months — but a price that moves by the same percentage every month is equally suspicious (copy-paste error, formulaic update). Extend the check to flag both.

**Where it lives:** `hcpi_data_collection` — `validate_inliers()` in `hcpi_data_validation.py`. The 12-month lookback is already there; extend the predicate.

**Branch:** `18.0` → all four country branches.
**Effort:** Algorithm extension + tests.

---

### T2.5 — Flag changes in Centrally Administered Prices

**What it is:** Government-administered prices (fuel, utilities, certain staples) move on a different cadence and need to be watched specifically. Add an `is_administered` flag on items and a "what moved this month?" report.

**Where it lives:** `hcpi_item` (new boolean field on `hcpi.item`) + a small `hcpi_dashboard` surface or `hcpi_index` report listing month-over-month changes restricted to administered items.

**Branch:** `18.0` → all four country branches.
**Effort:** Schema add + populated view.

---

### T2.6 — Progress bar shows 100% during collection then drops to 67% (KNBS)

**What it is:** Collectors complete what they think is a full questionnaire (100%), but after validation runs the displayed `progress` % drops. Almost certainly a denominator that grows during the validation stage (e.g. extra lines materialised) but isn't accounted for in the computed % during collection.

**Where it lives:** `hcpi_data_collection` — the `progress` compute on `hcpi.data.collection`. Identify what changes between `survey` state and `validation` state in the line count.

**Branch:** start on `18.0` (likely shared). If reproducible only with Kenya's extra `review` state, move to `ke_18_mtnce`.
**Effort:** Diagnose first, then a single-method fix.

---

### T2.7 — In-built calculator / "show the math" view (KNBS)

**What it is:** KNBS statisticians want to see *how* an index value was produced — which observations went in, which were dropped as outliers, the geometric mean, the weight applied. The EA-index compute already records rich diagnostics for this purpose (zero-price reasons, missing geo-mean, etc.) but they aren't surfaced in the UI.

**Where it lives:** `hcpi_index` — a new "Inspect" view on `hcpi.basket.elementary.aggregate.index` that pulls observation rows and diagnostics for the chosen (EA, segment, month).

**Branch:** `18.0` → all four country branches.
**Effort:** Read-only UI over data that already exists.

---

### T2.8 — Three-tier outliers (extreme 24mo / outlier 12mo / mild 6mo)

**What it is:** Today the Tukey algorithm marks a price either an outlier or not. Statisticians want graded severity tied to how many months back the comparison reaches, so they can prioritise review.

**Where it lives:** `hcpi_data_collection` — `run_turkey()` in `hcpi_data_validation.py`. Add an `outlier_severity` Selection on `hcpi.data.collection.line`; run the comparison at three windows and pick the most severe match.

**Branch:** `18.0` → all four country branches.
**Effort:** Algorithm extension + new field + UI surface + tests. Methodology should be signed off by a statistician before merging.

---

### T2.9 — Period and Division filters on index update wizards

**What it is:** Today running an index update wizard recomputes *everything*. If you only need to re-run May 2026 Food after fixing one outlet, you have to recompute every month and every division. Add filters so a wizard can target a slice.

**Where it lives:** `hcpi_index` — each `*.index.update` wizard (`ea.index.update`, `basket.coicop.index.update`, `basket.index.update`, `national.index.update`). Add `period_id` and `division_ids` fields and propagate them into the queue-job query.

**Branch:** `18.0` → all four country branches.
**Effort:** Same change applied to 4–5 wizards. Test that downstream re-aggregation still produces consistent totals on a subset.

---

### T2.10 — Collected data must reliably reach permanent observations

**What it is:** Reports that collected data sometimes gets "stuck" — it passes some validation steps but never materialises as `hcpi.outlet.item.observation` rows, so the index never sees it. Workflow bug in the `validation → done` transition.

**Where it lives:** `hcpi_data_collection` — `action_approve()` on `hcpi.data.collection`. Audit the path that writes observations; identify which edge case skips creation (cancelled mid-flow? items rejected as outliers? imputed-only lines?).

**Branch:** `18.0` → all four country branches.
**Effort:** Reproduction will be the slow part. The fix itself is usually one missing branch in the workflow method.

---

### T2.11 — Public dashboard access

**What it is:** External / public users (press, citizens, partner orgs) should be able to view headline CPI without logging in. Today the dashboard is internal-only.

**Where it lives:** New `hcpi_dashboard_public` module (or a controller in `hcpi_dashboard`) — exposes `/cpi` as a public website route with a sanitised `dashboard_hooks_public()` that only returns published, aggregated series.

**Branch:** `18.0` → optionally enabled per country.
**Effort:** Medium — must be designed so it cannot leak internal data, draft indices, or per-outlet observations. Security review required.

---

### T2.12 — Formal Review of Computations Process

**What it is:** EAC asked for an audit pass over how the index is actually computed.

**Where it lives:** [`HCPI_COMPUTATION_FLOW.md`](HCPI_COMPUTATION_FLOW.md) in this repo *is* the deliverable — it walks every stage from price quote to national figure with code line references. Needs publishing into the docs site and a guided walkthrough session.

**Branch:** `main` of docs repo.
**Effort:** Publication and presentation, no further investigation.

---

## Tier 3 — Hard, open-ended, or blocked

### T3.1 — Sector / national figures showing 0 on dashboard (UBOS + KNBS + Uganda-specific)

**What it is:** Affected dashboards show 0 for every sector or no national value. Three distinct possible root causes:

1. **Zero-price safety gate** stopped recomputation from a contaminated month onward. The shared computation refuses to compute index values past the first month containing a zero `observed_price`. Earlier months stay safe; everything after is blank until the bad price is corrected.
2. **All `dashboard_display` flags are False** — the dashboard reads only flagged classes and silently renders empty if none qualify.
3. **National index wizard never ran** for the months the dashboard is trying to display (the wizards do not cascade automatically).

**Where it lives:** Investigation first against each country's DB snapshot. Fix likely lands on the country branch (`ug_18`, `ke_18_mtnce`) as data / configuration, not as code.

**Branch:** Diagnosis on country branches; if a class of bug is found in `dashboard_hooks()`, fix lands on `18.0`.
**Effort:** Diagnostic-first. Could be a 1-hour data fix or a multi-day root-cause investigation depending on what's found.

---

### T3.2 — GPS not working during data collection (NBS)

**What it is:** Tanzania collectors report GPS location not captured when filing prices.

**Where it lives:** The Flutter mobile app — the server has no GPS code. The server-side `hcpi_data_collection_mobile_app` module would only be touched if a new field needs to be added to receive the GPS payload.

**Branch:** **Mobile app repo (separate from this codebase).** Server-side change, if any, on `18.0`.
**Effort:** Unknown — Android/iOS permissions and Flutter location-plugin debugging.

---

### T3.3 — Review syncing to Secretariat Dashboard (EAC)

**What it is:** The EAC Secretariat runs a separate aggregator app that pulls CPI snapshots from each country instance via XML-RPC every N hours. Reports of sync gaps or stale data need investigation.

**Where it lives:** `secretariat` branch — `hcpi_secretariat_hub/models/` (`remote_instance.py`, `dashboard.py`, `data/ir_cron.xml`). Cross-checks needed:

- Are integration users on each country instance still active?
- Are API keys still valid?
- Are the cron schedules firing?
- Does the `sync_month_window` (default 12) cover all the months the dashboard needs?

**Branch:** `secretariat` for code; each country's branch for the integration-user audit.
**Effort:** Audit + likely small code changes. Communication across the four country teams is the slow part.

---

### T3.4 — System slow with more users / more data (KNBS)

**What it is:** Performance degrades when concurrent users grow or as historical data accumulates.

**Where it lives:** No single file — needs profiling first. Likely contributors:

- `mail.message` and `mail.tracking.value` table growth (`tracking=True` is enabled on many fields).
- Missing indexes on `hcpi.outlet.item.observation` for the lookup queries used during validation.
- Default `queue_job` concurrency in `hcpi.conf` is conservative.
- The `ir_logging` table can grow unbounded if log level is verbose.

**Branch:** `18.0` for code/config changes; per-instance `hcpi.conf` tuning for queue concurrency.
**Effort:** Open-ended. Treat as a recurring "stability pass," not a single ticket.

---

### T3.5 — Large log tables → slow queries, large DB

**What it is:** Closely related to T3.4. `mail.message` and tracking tables grow forever; nothing prunes them.

**Where it lives:** Audit `tracking=True` across all `hcpi_*` modules — remove it from low-value fields. Add a retention cron for `mail.message` older than N months. Tune Odoo `ir_logging` retention.

**Branch:** `18.0` for the audit and retention cron.
**Effort:** Medium audit + a small cron module. Almost certainly the root cause behind a chunk of T3.4.

---

### T3.6 — Provisional Index (copy-forward previous month values)

**What it is:** When a month's data isn't fully collected yet but a headline number is needed (e.g. for media), publish a "provisional" index that fills gaps by carrying forward last month's values, with the methodology clearly labelled.

**Where it lives:** Code is small — a new wizard in `hcpi_index/models/computations/` that mirrors the EA index update but uses last-month observations for missing items. The **hard part is methodology agreement** between EAC and the four national statistics offices: is this a separate published series, or an imputation method on the existing one? How is it labelled?

**Branch:** `18.0` for code; methodology decision is EAC-wide.
**Effort:** Small code, slow governance.

---

### T3.7 — Deleting update history leaves an advisory lock and prevents restart

**What it is:** EA index computation acquires a PostgreSQL advisory lock per segment to prevent two workers running the same job. If the queue_job and history rows are deleted manually, the lock isn't released, the job isn't marked failed, and the next attempt to restart on the same segment silently blocks.

**Where it lives:** `hcpi_index` — `_job_compute_indices_for_segment` (advisory lock lives on the transaction). The deletion path needs to either (a) refuse to delete an in-flight job, or (b) mark the job failed and release/expire the lock.

**Branch:** `18.0` → all four country branches.
**Effort:** Medium — concurrency-sensitive change, easy to introduce a race if done carelessly. Needs concurrent-load tests.

---

### T3.8 — Standardise item codes across countries

**What it is:** Each country's items use different codes for what is logically the same EAC item. Harmonisation would let the Secretariat aggregator do real cross-country comparisons at the item level (today it only compares EAs and above).

**Where it lives:** `hcpi_item` (possibly add an `eac_standard_code` field that's separate from the country-local code) + a mapping data file + a one-off migration on each country instance.

**Branch:** `18.0` for the schema; mapping data on each country branch; migration scripts per instance.
**Effort:** Largest non-engineering cost in this list — needs sign-off from all four NSOs on a canonical item list. Engineering is small once the list exists.

---

### T3.9 — "Standard and non-standard of weights" (KNBS)

**What it is:** Unclear — could mean "show standard vs deviation of weights," "separate standard EA weights from custom ones," or "validate weights sum to 100%."

**Where it lives:** Cannot scope until clarified with the reporter.

**Branch:** TBD.
**Effort:** TBD — get clarification first.

---

## Suggested first sprint

If picking up work now, the highest-value bundle is a single shared PR on `18.0`:

- T1.3 (outlier / inlier report)
- T1.4 (temporarily-missing report)
- T1.5 (show all divisions)
- T1.7 (Y-on-Y + Core / Non-core indicator cards)
- T2.1 (month/year picker)
- T2.2 (national CPI hero card)
- T2.3 (collection analytics)

All live in `hcpi_dashboard`, `hcpi_data_collection`, and `hcpi_outlet`, all are additive, and together they answer the bulk of the "Dashboard issues for all" section plus three Suggestions.

A second pass on stability:

- T2.6 (KNBS progress bug — reproduce and fix)
- T3.5 (log / tracking audit — almost certainly the source of T3.4 slowness)

Hold T3.1 (sector zeros) until a database snapshot is available — it is almost always a data state issue, not a code bug.
