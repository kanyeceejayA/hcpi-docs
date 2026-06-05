# What Changed — June 2026 (Overview)

A plain-language tour of **everything** that changed in the HCPI software during
the June 2026 work. Part of that work was to **validate the system and identify
where it was inconsistent**, so this page reads in four steps:

1. **[Gaps & inconsistencies we identified](#gaps--inconsistencies-we-identified)** — what the validation and methodology audit found was wrong.
2. **[Changes implemented](#changes-implemented)** — what we fixed, biggest first, each with *what was wrong → what is better now*.
3. **[What's still pending](#whats-still-pending)** — gaps left open, mostly methodology decisions awaiting sign-off.
4. **[The way forward](#the-way-forward-product-roadmap)** — the broader roadmap, including whole areas (like the mobile app) not yet touched.

Written for both statisticians and developers.

Two companion pages go deep on the two largest areas:

- **[Dashboard & Index Overhaul](dashboard-index-overhaul.md)** — the index
  engine and the dashboards.
- **[Validation, Observations & Imputation](validation-to-observations-and-imputation.md)**
  — the price pipeline from collection to published index.

> **Where it runs.** All changes live on the shared **`18.0`** base and are
> rolled out to each country (Uganda, Kenya, Tanzania, Zanzibar) with
> `bin/hcpi propagate`. **Uganda is always tested first.**

---

## Gaps & inconsistencies we identified

Before fixing anything, we validated the engine and the price pipeline against
standard CPI practice and against the live country data. These are the concrete
problems that surfaced. **Status** shows what's been done since:
✅ fixed · ◑ partly addressed · ⬜ still open.

| # | Where | The inconsistency | Status |
|---|-------|-------------------|--------|
| G1 | Index engine | Incomplete months **collapsed toward zero** — not-yet-computed parts counted as `0` but kept their full weight, dragging the average down (National CPI reading ~`30` instead of ~`136`). | ✅ |
| G2 | Dashboard | A division figure showed **one arbitrary sub-row** (sometimes a region worth < 2%) as if it were the whole division. | ✅ |
| G3 | EA index update | The update **wiped all consumption segments up front**, so a crash/cancel mid-run left the whole index zeroed (the cause of Uganda's all-zero state). | ✅ |
| G4 | Index integrity | **Weighted nodes that resolved to no value were invisible** — no way to see which EA/class/division was missing. | ✅ |
| G5 | Validation flow | Validation **could not reach Done** — the Validate step didn't advance the record and Impute never finished. | ✅ |
| G6 | Pipeline | **Validated/imputed prices never reached the index** — nothing copied them from validation lines into Observations (the only table the index reads), so field prices and the index diverged wildly (e.g. school fees 1.35M vs 12M). | ✅ |
| G7 | Import | A spreadsheet re-import **aborted the whole file** on the first duplicate row. | ✅ |
| G8 | Imputation speed | Imputing one segment fired **tens of thousands of queries** and took minutes. | ✅ |
| G9 | Imputation rule | The donor rule required a candidate group to be **100% collected**; one missing price disqualified it, pushing items to coarser groups or to un-imputable. | ◑ |
| G10 | Imputation counts | A line could be **flagged "imputed" while still `0`** (missing prior index), inflating the imputed count and hiding real gaps. | ✅ |
| G11 | Outlier method | The **"Turkey" rule is a custom variant, not true Tukey IQR fences** — it drops exactly-unchanged prices before computing limits and uses *mean ± 2.5 × spread*. Undocumented. | ⬜ |
| G12 | Imputation base | The same-product **"previous price" is taken across *all* outlets**, not the same outlet-item; plus a latent `item_id` vs `outlet_item_id` comparison bug in the EA look-up. | ⬜ |
| G13 | Imputation guard | `geo_mean` of previous prices can hit **LN(0)** on sparse history; the zero/empty guard isn't applied consistently. | ⬜ |
| G14 | Policy | The policy for **genuinely un-imputable items** (exclude vs carry-forward) and for **discontinued-item substitution** is implicit/undecided. | ◑ |
| G15 | Reliability | A killed run left its record **stuck "processing", blocking every other computation forever**; deleting update history left a lock; and the update forms reloaded over in-flight actions, dismissing dialogs and stranding records. | ✅ |
| G16 | Mobile app | **Sync doesn't include questionnaire content** — questionnaires sync without their content, so a collector can't work offline straight after syncing; the content arrives later. | ⬜ |
| G17 | Mobile app | **Questionnaire field order** — the collection code is chosen *before* the price; it should come *after*, and selecting *temporarily / permanently missing* should clear the price automatically. | ⬜ |
| G18 | Mobile app | **No on-device period filter** — collectors can't filter to the current collection period, and a device can't be limited to show only questionnaires. | ⬜ |
| G19 | Mobile app | **GPS not captured during collection** (NBS) — location isn't recorded; needs reproduction on a device. | ⬜ |
| G20 | Mobile app | **Offline operation unconfirmed** — whether a tablet keeps working when the server is offline is untested/undocumented (NBS). | ⬜ |
| G21 | Mobile app | **No version control / distribution** — no pinned APK on the servers, not (re)published to the Play Store, and the app sits under **Kola's** Google Play account instead of an **EAC-controlled** one. | ⬜ |
| G22 | Mobile app | **XML/RPC endpoints undocumented** — the integration contract the app relies on isn't documented or maintained. | ⬜ |

> The imputation findings (G9–G14) come from the engine audit,
> `hcpi_data_collection/IMPUTATION_METHODOLOGY_AUDIT.md`. They change published
> numbers, so each is a recommendation pending statistician sign-off — see
> **[What's still pending](#whats-still-pending)**.
>
> The mobile-app findings (G16–G22) are **field-reported** (not from the June
> engine audit). The Flutter app was **not touched** in June, so all are still
> open — the full detail and roadmap is in
> **[section A](#a-mobile-data-collection-app--untouched-separate-repo)**.

---

## Changes implemented

The fixes are grouped by theme below, **biggest first**, each with *what was wrong
→ what is better now*. This table maps them to the gaps above.

| # | Theme | The headline | Closes | Detail |
|---|-------|--------------|--------|--------|
| 1 | **Correct index numbers** | Incomplete months no longer collapse to zero | G1, G2 | [Overhaul](dashboard-index-overhaul.md) |
| 2 | **Prices now reach the index** | Validated prices are promoted into Observations | G6 | [Pipeline](validation-to-observations-and-imputation.md) |
| 3 | **Validation finishes** | The Validate → Impute → Finalize → Done flow works end to end | G5 | [Pipeline](validation-to-observations-and-imputation.md) |
| 4 | **Faster, smarter imputation** | Minutes → ~16s, fills more gaps, honest counts | G8, G9, G10 | [Pipeline](validation-to-observations-and-imputation.md) |
| 5 | **Imports don't fail on duplicates** | Re-importing a month is resolved by policy, not an error | G7 | [Pipeline](validation-to-observations-and-imputation.md) |
| 6 | **Computations can't get stuck** | Dead jobs auto-release; a blocked run offers a one-click fix | G15 | below |
| 7 | **Readable dashboards** | Month selector, clearer cards, info icons, a 4-band heatmap | G2 | [Overhaul](dashboard-index-overhaul.md) |
| 8 | **Data-integrity visibility** | Missing weighted nodes are flagged level by level | G3, G4 | [Overhaul](dashboard-index-overhaul.md) |
| 9 | **Secretariat (regional) dashboard** | Month selector, pull timestamps, and a one-click "Pull latest" | — | below |

---

## 1. The index now shows the right numbers (the big one)

**What was wrong.** An index is a *weighted average* of its parts. Parts that
hadn't been computed yet still counted as `0` but kept their full weight, so they
dragged the whole index toward zero. A month that was only half collected looked
like prices had crashed — the National CPI could read `30` when it should be
about `136`, and division cards showed zeros or one tiny region in place of the
whole division.

**What is better now.**

- Every weighted average **skips parts that have no value yet** and averages only
  the data that actually exists. An incomplete month is now an honest average of
  what's been collected.
- A new stored figure, **one correct value per division per month**, is produced
  during the National Index run, so the dashboard reads a single right number
  instead of an arbitrary sub-row.
- A **provisional estimate** is available for the in-progress month: the system
  carries forward each not-yet-priced item's last known price to produce a usable
  early figure. Provisional numbers are **kept separate and never overwrite** the
  official series, and come with a **"% collected"** coverage metric.

→ Full detail in **[Dashboard & Index Overhaul](dashboard-index-overhaul.md)**.

---

## 2. Validated prices now actually reach the index

**What was wrong.** Prices flow
`Questionnaire → Validation → Observations → Index`. The engine wrote validated
prices onto the validation lines, but **nothing copied them into the Observations
table** — and Observations is the only table the index reads. So even a fully
validated month did not change the published index. (This is why field prices and
the index could diverge wildly — e.g. school fees of 1.35M in observations vs 12M
collected.)

**What is better now.** A new **"Promote to Observations"** button on a finished
validation copies its prices into Observations, one consumption segment and month
at a time. Each line is shown as **NEW** or as a **CLASH** with an existing
observation; clashes default to *the validated value winning*, and any row can be
flipped to *keep existing*. This is the missing link that lets validated prices
reach the index.

→ Full detail in **[Validation, Observations & Imputation](validation-to-observations-and-imputation.md)**.

---

## 3. Validation can now be completed

**What was wrong.** The "Validate" step didn't move the record forward, so the
next button never appeared, and "Impute" never reached a finished state — a
validation simply **could not get to Done**.

**What is better now.** The flow runs end to end:
**Validate → Validate Inliers → Impute → Finalize → Done.** A new **Finalize**
step closes the validation and warns that any line still without a price will be
excluded from the index, and a new **"Unimputed (no price)"** counter makes those
excluded lines visible instead of silently dropped.

→ Full detail in the **[Pipeline page](validation-to-observations-and-imputation.md)**.

---

## 4. Imputation is faster and smarter

**What was wrong.** Imputing one segment fired tens of thousands of database
lookups and took **minutes**. Its rule for choosing where to borrow a price
movement from was also **too strict**: a donor group was only usable if **100% of
its items were collected**, so a single missing price disqualified the whole
group and pushed the item to a broader, less-similar group — or left it
un-imputable.

**What is better now.**

- The lookups are **pre-computed once per validation** and a missing index was
  added. A large segment dropped from **minutes to about 16 seconds**, with
  identical results, and a **progress bar** now shows while it runs.
- The donor rule was relaxed to **"at least *N* collected prices"** (default
  N = 1) — the standard CPI "impute from the lowest-level cell that has data"
  approach — so more items impute from closer, more representative groups.
- A **preview before imputing** shows how many lines will be imputed, the
  segment's completeness %, and *why* each item is missing (temporarily missing,
  permanently discontinued, or simply not collected). Permanently-discontinued
  items are imputed only as a stopgap and flagged for replacement.
- **Honest counts:** a line an imputation rule touched but that still ended at `0`
  is no longer counted as imputed.

→ Method, assumptions, and the open methodology questions are in the
**[Pipeline page](validation-to-observations-and-imputation.md)**.

---

## 5. Spreadsheet imports no longer fail on duplicates

**What was wrong.** Re-importing a month that already had data **failed the whole
file** the moment it hit a duplicate row.

**What is better now.** A row that duplicates an existing observation (same
outlet-item + month) is resolved by a **policy** instead of aborting:

- **overwrite** (default) — the uploaded row wins,
- **skip** — keep the existing observation,
- **block** — the old abort behaviour.

Non-conflicting rows always import, and every overwrite/skip is reported back as a
non-blocking warning (visible in the "Test" preview too).

---

## 6. Index computations can't get permanently stuck

This group is about **reliability** — making sure a single bad run can't freeze
all index computations. It's the part not covered by the two deep-dive pages, so
here is the full picture.

**What was wrong.** Every index update refuses to start while *any* update is
still "processing". If a run was killed or crashed, its record stayed stuck in
"processing" forever — so one dead job (e.g. a 3-day-old Domestic Index update)
**blocked every other computation indefinitely**. There was no way out short of
waiting or editing the database directly. Separately, the update forms reloaded
the page about a second after you clicked **Start Update**, which could dismiss
error dialogs and even leave a record stuck in "processing".

**What is better now.**

- **Dead jobs auto-release.** Before each update checks "is another job running?",
  it automatically fails any update stuck in *processing* with no progress for
  over **2 hours**. A live run writes progress every few seconds, so only
  genuinely dead locks are cleared.
- **A blocked run offers a one-click fix.** Instead of a dead-end error, a blocked
  update now opens a small wizard listing the running job(s) with a button to
  **force-release them and retry immediately**.
- **The forms stop reloading over you.** The page only reloads when an update
  truly starts; if it was blocked or errored, the dialog stays open so you can
  act on it.

These three together mean a computation can always be unblocked from the UI,
without psql or a 2-hour wait.

---

## 7. The dashboards are easier to read

**What was wrong.** The dashboard often showed one arbitrary sub-row as a whole
division, had no way to view a past month, and a single failing section could
blank the entire page to zero.

**What is better now.**

- A **Month selector** on both the analytical and operations screens.
- **National CPI is the first card**, then division cards, each with a
  **month-over-month change pill** and a sparkline; equal-height cards with
  full names on hover.
- A **historical trend chart** beside a **by-division table**.
- The in-progress month carries a **"Provisional · X% collected"** tag until it is
  both computed and fully collected; finished official months show no tag.
- **Per-section error handling** — one failing section no longer zeroes the whole
  page.
- A new **Operations screen**: % collected (overall and per division), the
  provisional CPI, number of collections, temporarily-missing items (including a
  count missing > 2 months), and outlier / inlier / imputed totals.
- The **Division Index Heatmap** was recoloured into a clearer **4-band scale**
  (Below base / Low / Medium / High), with cut-offs picked from the real Uganda
  division spread.
- **Info (i) icons** next to every figure open a short explanation of *what it
  means and where it comes from*, clarifying the collections-vs-observations-vs-
  computed-index distinction. Card titles were reworded for clarity, and the
  validation cards now sit under a **"Price validation"** heading.

→ Full detail in **[Dashboard & Index Overhaul](dashboard-index-overhaul.md)**.

---

## 8. You can now see where data is missing

**What was wrong.** When the index resolved to no value somewhere, there was no
easy way to see *which* node was missing.

**What is better now.** The operational dashboard **flags weighted nodes that
resolve to no value** at every level (EA → Micro → … → Division), with per-level
coverage badges and a drill-down list of the actual missing nodes. It's
read-only and changes no computed value. The same release also fixed two
imputation cascade bugs (a typo that stopped sub-class imputation, and a
TZ-only model reference that crashed imputation on every other country),
added an **optional EA-level carry-forward** (off by default), and **hardened the
EA index update** to rebuild one consumption segment at a time — so a
cancel or crash can no longer leave the index globally zeroed (the original cause
of Uganda's all-zero state).

→ Full detail in **[Dashboard & Index Overhaul](dashboard-index-overhaul.md)**.

---

## 9. The regional (Secretariat) dashboard

This is the **EAC Secretariat hub** — a separate module (`hcpi_secretariat_hub`,
on its own `secretariat` branch) that pulls a snapshot from each country's
instance and shows them side by side for the region. It's the start of the work
on *"review syncing of data to the Secretariat dashboard"*.

**What was wrong.** The hub only ever showed the **latest pulled figures**, with
no way to look at an earlier month, no indication of **when** the data was last
pulled, and no way to **refresh on demand** — you waited for the scheduled sync.

**What is better now.**

- **Month selector** — the country cards, country comparison, collection figures
  and top divisions/classes all follow a chosen pulled month (default = latest);
  the historical trend stays full-history.
- **Pull timestamps** — the dashboard shows *"Last pulled … / Showing &lt;month&gt;"*
  so you know how fresh the regional picture is.
- **"Pull latest data" button** — syncs every active country on demand, isolating
  failures per country, and shows progress and the outcome.

> This is regional **read-only reporting** built on top of each country's own
> numbers — it doesn't change any country's index. Deeper review of *how* the
> country→Secretariat sync works is still on the roadmap (see
> [section E](#e-performance-infrastructure--integration)).

---

## How to roll it out (quick reference)

```bash
bin/hcpi propagate                                 # merge 18.0 into all country branches
bin/hcpi update ug hcpi_index,hcpi_dashboard       # upgrade Uganda first
# recompute in order: EA → Basket COICOP → Basket → National
# then press "Refresh Provisional" for the in-progress month
# repeat for ke / tz / zar once Uganda passes
```

The official index is built **bottom-up**, and the **Elementary-Aggregate step is
the foundation** — if a collected month still reads provisional/zero, the EA step
almost certainly hasn't been run for it yet. See the
**[Overhaul page](dashboard-index-overhaul.md)** for the full computation order
and the **[Pipeline page](validation-to-observations-and-imputation.md)** for the
validate-and-promote steps.

---

## Things to keep in mind

- **Provisional ≠ official.** Provisional figures are flagged, kept out of the
  published series, and recomputed on demand. Use the **% collected** indicator to
  judge how final a month is.
- **Outlier / inlier / imputed counts** are bucketed by the *validation's* month,
  which can differ from the month prices were observed — if a month shows `0`,
  switch to the month the validation was recorded under.
- Several **methodology choices are deliberate assumptions** awaiting statistician
  sign-off (donor minimum, import/promotion conflict winners, treatment of
  discontinued and un-imputable items). They are listed in the
  **[Pipeline page](validation-to-observations-and-imputation.md)**.

---

## What's still pending

These are the gaps the validation surfaced that are **not yet closed**. They are
mostly **methodology decisions that move published numbers**, so they were
deliberately left for statistician sign-off rather than changed unilaterally.
(The broader product roadmap — new features and untouched areas like the mobile
app — is the next section.)

**Methodology decisions awaiting sign-off** (each maps to a gap above):

- **Donor rule fallback (G9).** The donor minimum was relaxed, but when an item
  still can't be imputed from its own relative it does **not yet fall back** to
  the same item in another segment, then another EA. Decide the fallback chain and
  the lenient threshold (≈20–30% of a group priced).
- **Outlier method (G11).** Confirm whether the "Turkey" rule should become **true
  Tukey IQR fences** or stay the custom *mean ± 2.5 × spread* variant — and
  whether exactly-unchanged prices should be excluded from the bounds. Document
  the chosen method.
- **Imputation base price (G12).** Agree that the same-product previous price
  should come from the **same outlet-item**, not any outlet — and fix the latent
  `item_id` vs `outlet_item_id` look-up bug alongside it.
- **Sparse-history guard (G13).** Apply the `geo_mean` zero/empty (LN(0)) guard
  consistently across all call sites.
- **Un-imputable & discontinued policy (G14).** Decide the official rule for
  genuinely un-imputable items (exclude vs carry-forward) and add a **substitution
  workflow** for discontinued items (today they're imputed as a flagged stopgap
  only).

**Default assumptions already shipped — please confirm or correct:**

- Imputation **donor minimum = 1** collected price (`hcpi_index.impute_min_donor_quotes`).
- **Import** conflicts: the **uploaded row wins** by default (`overwrite`).
- **Promotion** conflicts: the **validated value wins** by default (per-row overridable).
- Previous-price look-backs are **bounded to before the validation month**.

Full reasoning for each is in the
**[Pipeline page](validation-to-observations-and-imputation.md)** and the engine
audit (`IMPUTATION_METHODOLOGY_AUDIT.md`).

---

## The way forward (product roadmap)

The June 2026 work cleared a big block of the known issues — the engine zeros,
the validation dead-end, the stuck-lock problem, the dashboard basics, and the
provisional index are all addressed above. But plenty is still open, and a few
whole areas — most notably the **mobile data-collection app** — were not touched
at all. This section is the roadmap of what's left, grouped and roughly
prioritised.

### Already closed by the June work

So the list below isn't misread, these reported items are **done** and described
above: month/year picker, National CPI as the hero card, show-all-divisions,
sector/national figures reading zero, validated data reaching observations,
provisional carry-forward index, per-level missing-value flagging, dead-lock
release on index updates, and the first pass on the **Secretariat dashboard**
(month selector, pull timestamps, on-demand "Pull latest").

### A. Mobile data-collection app — untouched (separate repo)

None of the June work touched the Flutter app; all of these are still open. This
is the largest unaddressed area and several items are field-blocking:

- **Offline-ready sync** — when questionnaires sync to a device they should
  arrive *with their content*, so a collector can work fully offline immediately
  after syncing (today the content arrives later).
- **Questionnaire field order** — select the collection code *after* the price is
  entered, and clear the price automatically when *temporarily* or *permanently
  missing* is chosen.
- **Period / questionnaire filtering on the device** — filter by the current
  collection period and let a device show only questionnaires.
- **GPS not working during collection** (NBS) — needs reproduction on a device.
- **Confirm true offline operation** when the server is offline (test and document).
- **Version control & distribution** — publish a pinned APK on the servers so
  collectors are always on the right build, **publish to the Play Store**, and
  **transfer the app to an EAC-controlled Google Play account** (currently Kola's).
- **Document the XML/RPC endpoints** the app uses, so the mobile surface is a
  known, maintained contract.

### B. Computation & methodology (statistical correctness)

Mostly awaiting methodology sign-off, then build:

- **No un-imputed items.** When an item can't be imputed from its own relative,
  fall back to the **same item in another segment**, then **another EA**, before
  giving up — and make the donor threshold lenient (≈20–30% of a group priced).
  June relaxed the donor rule but did **not** add the cross-segment / cross-EA
  fallback.
- **Three-tier outliers** (extreme = 24 months, outlier = 12, mild = 6) and a
  **smarter inlier check** (12 consecutive months *and* same-direction change).
- **Outliers stay flagged after imputation** — their state shouldn't be cleared,
  since it feeds later steps.
- **Centrally-administered prices** — monitor and flag changes.
- **Item substitution / replacement** workflow for discontinued items (today they
  are only imputed as a flagged stopgap).
- **Standardise item codes** across countries; clarify **standard vs non-standard
  weights**; provide an **in-built calculator / "show the math"** explainer.
- **Period and division filters on the index-update wizards**, so a run can target
  a subset instead of everything.
- A **formal review of the computation process** end to end against how it
  *should* work.

### C. Reporting & dashboards (the next layer of visibility)

- **Outlier / inlier report** and the ability to **approve or reject** flagged
  lines (singly or in bulk); rejected lines excluded from calculation but listed.
- **"Temporarily missing for 3+ months" report** (the count exists; the report
  doesn't).
- **Annual % change** and **Core / Non-core** figures on the indicator cards.
- **Data-collection analytics** — counts by collection state across outlet /
  centre / national.
- **Coverage-laggard view** — which consumption segments are furthest behind on
  outlet-item coverage.
- **Processing-history report** — how the CPI was computed per segment, per
  period, per user (logs/audit).
- **Public dashboard access** and a **link from the provisional figure to its
  validation report**; a **collections-vs-calendar-days** report.
- **Heatmap key follow-up** — the reporter asked for the blue/red bands swapped
  and **blue added with an explained key**; June instead chose a no-blue 4-band
  green→amber→red scale. Reconcile this with the reporter.
- Minor: **remove the provisional column** from "CPI by division", and make the
  validation **progress field update as status changes** (a progress bar exists
  for imputation, but the field itself still doesn't move).

### D. Workflow & access

- **Promote-to-Observations**: add an **item-code column**, and **bulk select /
  group actions** on both the promotion screen and the import screen.
- **Review permissions** for validation and promotion so only authorised users
  can run them.

### E. Performance, infrastructure & integration

- **Log/tracking audit** — large log tables are bloating the database and slowing
  queries; review what's logged and trim it. This is the most likely root cause
  of the **"slow with more users / more data"** reports (KNBS).
- **KNBS collection progress bug** — the progress bar shows 100% during collection
  then drops to 67%; reproduce and fix.
- **Admin resource view** — surface server load / what's driving it.
- **Secretariat sync** — the hub now has a month selector, pull timestamps and an
  on-demand "Pull latest" (see [section 9](#9-the-regional-secretariat-dashboard)).
  Still to do: a deeper review of *how* the country→Secretariat sync works
  (reliability, what's pulled, conflict handling).

### Suggested next sprint

The highest-value next bundle is a single shared `18.0` dashboard/reporting PR —
**outlier/inlier report, temporarily-missing report, annual % change +
core/non-core cards, and collection analytics** — followed by a stability pass on
the **log audit** and the **KNBS progress bug**. The mobile-app items are best
scoped as their own track since they live in a separate repo.
