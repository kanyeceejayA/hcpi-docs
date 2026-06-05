# Validation → Observations, Imputation & Import Conflicts (June 2026)

Plain-language summary of a set of changes to the price **validation**,
**imputation** and **import** pipeline: **what was broken, what changed and
where, the imputation method and the assumptions taken, how to operate it, and
what to keep in mind.** Written for both statisticians and developers, and
intended for review/sign-off before the methodology assumptions are treated as
official.

---

## The problem we set out to fix

Collected prices flow:

```
Questionnaire (collection)  →  Validation (outliers, inliers, imputation)  →  Observations  →  Index
```

Three things were broken in the middle of that chain:

1. **Validation never finished.** The "Validate" step didn't advance the record,
   so the next button never appeared; and "Impute" never moved the record to a
   completed state. A validation could not reach **Done**.
2. **Validated/imputed prices never reached the index.** The engine wrote prices
   onto the validation lines but **nothing copied them into the Observations
   table** — which is the only table the index actually reads. So even a fully
   validated month did not change the published index. (This is why field-
   collected prices and the index could diverge wildly — e.g. school fees of
   1.35M in observations vs 12M collected.)
3. **Spreadsheet import aborted on any duplicate.** Re-importing a month that
   already had data failed the whole file with an error, instead of letting the
   user decide.

Alongside these, imputation was **very slow** (minutes for a large segment), and
its rule for choosing where to impute from was **too strict** (explained below).

---

## What changed (and where)

All changes live on the shared **`18.0`** base branch and are fanned out to each
country with `bin/hcpi propagate`. **Uganda is tested first**, then Kenya,
Tanzania, Zanzibar.

### 1. Validation now completes
`hcpi_data_collection/models/hcpi_data_validation.py`

The state machine was repaired: **Validate → Validate Inliers → Impute →
Finalize → Done**. A new **Finalize** step closes the validation; it warns that
any line still without a price will be excluded from the index. A new
**"Unimputed (no price)"** counter and stat button make those excluded lines
visible instead of silently dropped.

### 2. Promote validated prices into Observations (with conflict review)
New wizard `hcpi.promote.observation.wizard`.

A **"Promote to Observations"** button on a finalized validation copies its
prices into the Observations table — **batched by collection month** for one
consumption segment. Each line is shown as **NEW** (no observation yet that
month) or **CLASH** (one already exists). Clashes default to **the validated
value winning**, and any row can be switched to **keep existing**. This is the
missing link that lets validated prices reach the index. Writes go straight to
the observation records, so the dedup guard used by the importer is never in the
way.

### 3. Import no longer aborts on duplicates — the uploaded row wins
`base_import_inherit/models/base_import.py`

A spreadsheet row that duplicates an existing observation (same outlet-item +
month) is now resolved by a **policy** instead of failing the import:

- **overwrite** (default): the **uploaded row wins** — the existing observation's
  price is replaced.
- **skip**: keep the existing observation, ignore the upload.
- **block**: the old behaviour (abort).

Non-conflicting rows always import. Every overwrite/skip is reported back as a
non-blocking warning, and the "Test" preview shows it too. The policy is the
system parameter `hcpi_import.observation_conflict_policy` (default `overwrite`).

### 4. Imputation made fast
Imputing one segment used to fire tens of thousands of database lookups (a
previous-price query for every collected line, repeated for every missing line,
plus a completeness query per candidate group). These are now **pre-computed
once per validation** in two queries, and a missing database index was added.
Result on a large segment (876 lines, 175 missing): **from minutes to ~16
seconds**, with identical results. A **progress bar** now shows while imputing.

### 5. See a particular collection period at a glance
Observations, Questionnaires and Validations list views gained **This Month /
Last Month** quick filters and a monthly **group-by**. Questionnaires now group
by month instead of by individual day.

### 6. Operational dashboard: promotion + collections-vs-validations
`hcpi_dashboard` operational dashboard gained two sections:

- **Promotion status**: how many validations are done, how many of those have
  been promoted vs still pending, and the total unimputed lines for the month.
- **Collections vs validations**: for the month, how many validated prices match
  an observation, how many exist only in validation, how many **clash**, and the
  **mean absolute % difference** — the live version of the old ad-hoc comparison
  spreadsheet.

### 7. A preview/alert before imputing
Clicking **Impute** now opens a short preview showing how many lines will be
imputed, the segment's **collection completeness %**, and a **breakdown of why**
each missing item is missing (see the imputation section). It nudges users to
collect more before imputing when completeness is low. **Proceed** runs the work;
**Cancel** aborts.

---

## How imputation works (and the rule we relaxed)

When an item has **no price** this month, the engine does not invent a number. It
takes the item's **own previous price** and multiplies it by the **price movement
of a group of similar items** that *were* collected:

```
imputed_price(item)  =  item's previous price  ×  (price movement of a donor group)
```

The "price movement" of a group is its current average ÷ its previous average
(a geometric mean). The engine looks for a **donor group** from **closest to
broadest** and uses the first usable one:

```
the item's Elementary Aggregate → micro-class → sub-class → class → group → division
```

Closer groups contain more similar products, so they give better imputations.

### The rule that was too strict
A donor group used to be usable **only if 100% of its items were collected** this
month. One missing price disqualified the whole group, forcing the item up to a
broader, less-similar group — or leaving it un-imputable.

**Example.** Item X (rice at outlet A) is missing. X's "rice" Elementary
Aggregate has 5 items; 4 were collected, 1 was not. Under the old rule the whole
group was rejected (not 100% complete), so the 4 good rice prices were thrown
away and X was pushed to a broader group or dropped.

### What it is now (aligned with standard CPI practice)
A donor group is usable once it has **at least *N* collected prices** (default
**N = 1**), and the movement is computed from whatever **is** present. This is the
standard "impute from the lowest-level cell that has data" approach. In the
example, the 4 collected rice prices are now used to impute X. The minimum is the
system parameter `hcpi_index.impute_min_donor_quotes` (default `1`) so it can be
raised per country.

> **Effect observed.** Relaxing the rule lets more items impute from closer, more
> representative groups. On the Uganda test month it did **not** by itself empty
> the "could not impute" list, because the remaining gaps are caused by a
> different thing: **missing previous-period indices** for the donor groups (you
> cannot compute a movement without a prior value). That is a data prerequisite
> (run the historical index first), not a rule problem.

### Why a missing item is missing — the breakdown
Each priceless line carries (from collection) a reason, so the preview classifies
them into three buckets:

- **Temporarily missing** — collected with a reason code (e.g. *Temporarily
  Missing*, out of season). Imputed normally.
- **Permanently missing** — code **P** (*Permanently Missing* / discontinued).
  **Imputed only as a stopgap** and tagged
  `[PERMANENTLY MISSING - review for replacement]`, because a discontinued item
  should be **substituted**, not imputed forever.
- **Not collected at all** — no price and no reason recorded. Imputed, but it
  signals a **collection gap** worth following up.

### Honest counts
A line that an imputation rule touched but that still ended at **0** (e.g. its
donor group had no prior index) is **not** counted as imputed. So
`imputed_count` = real successes and `unimputed_count` = real gaps. On the
Uganda test month: of 175 missing, **129 imputed, 46 genuinely un-imputable**
(all 46 blocked by missing prior indices).

---

## Assumptions taken (for statistician / dev review)

These are deliberate choices that affect published numbers. Please confirm or
correct them:

1. **Donor minimum = 1 collected price** (configurable). We impute from the
   lowest-level group that has any data, per standard practice. Raise
   `hcpi_index.impute_min_donor_quotes` if a larger minimum is preferred.
2. **Import conflicts: the uploaded row wins by default** (`overwrite`). The
   assumption is that a fresh upload is the corrected/canonical figure.
3. **Promotion conflicts: the validated value wins by default** (per row
   overridable). The assumption is that the validated number supersedes an
   earlier observation for the same month.
4. **Permanently-missing items are imputed as a stopgap and flagged**, not
   excluded. They should be replaced via substitution (no substitution workflow
   exists yet — flagged for follow-up).
5. **Un-imputable items stay excluded from the index** (left at 0, surfaced via
   `unimputed_count`) rather than carried forward. A separate opt-in EA
   carry-forward exists for the provisional path only.
6. **Previous-price look-backs are bounded to before the validation month**, so
   promoting the current month cannot make it act as its own "previous" period.

## Methodology points still open (recommended, not yet changed)

From the engine audit (`hcpi_data_collection/IMPUTATION_METHODOLOGY_AUDIT.md`):

- The **outlier ("Turkey") rule** is a custom *trimmed mean ± 2.5 × spread*, not
  classic Tukey IQR fences, and it excludes exactly-unchanged prices before
  computing limits. Confirm the intended method or document this variant.
- The same-product previous price is fetched **by item across all outlets**; it
  should arguably use the **same outlet-item's** own previous price.
- **Inlier detection** (flagging prices unchanged for 12 months) is informational
  only; administered/regulated prices will always flag — a whitelist would cut
  noise.
- Decide the official policy for genuinely un-imputable items (exclude vs carry
  forward) and for discontinued-item substitution.

---

## How to operate it

- **Validate a month**: open the Validation, click **Validate → Validate Inliers
  → Impute** (review the preview, then Proceed) **→ Finalize**.
- **Publish to the index**: on the finalized validation click **Promote to
  Observations**, resolve any clashes, **Promote**. Then run the index
  computation as usual.
- **Re-import a spreadsheet**: just import; duplicates are resolved by the policy
  and reported. Change `hcpi_import.observation_conflict_policy` if you want
  `skip` or `block` instead of `overwrite`.
- **Watch progress**: the operational dashboard's *Promotion status* and
  *Collections vs validations* cards track how much of a month has been promoted
  and how validated prices compare to what's in observations.
