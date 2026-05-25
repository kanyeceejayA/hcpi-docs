# CPI Concepts

This page is for developers who know Odoo (or are learning it) but don't have a statistics background. It explains what the HCPI system is actually *computing* — enough domain context that the model names, the data flow, and the validation rules make sense.

If you've done statistics work before, skim. If terms like *Laspeyres index*, *basket*, *elementary aggregate*, or *price relative* are new — start here.

## What is a CPI?

A **Consumer Price Index (CPI)** is a single number that says how expensive a typical household's consumption is *this period* compared to *some reference period*. Standard convention sets the reference period to **100**.

If this month's CPI is **117.3**, it means a household's typical basket of goods costs 17.3% more than in the reference period.

Computing one CPI number requires three things:

1. **A list of items** to price (a kilogram of sugar, a loaf of bread, a litre of petrol, a haircut, ...)
2. **Prices for those items**, sampled from real outlets at regular intervals (monthly is the standard)
3. **Weights** that say how important each item is in the typical household budget

These three are precisely what the HCPI codebase models. Items live in `hcpi.item`. Prices live in `hcpi.outlet.item.observation`. Weights live on the COICOP hierarchy and in the basket structure.

## The COICOP hierarchy

You can't average the price of bread and the price of haircuts directly — you need to group items into related categories first, then aggregate up. The international standard for this is **COICOP** (Classification of Individual Consumption by Purpose), published by the UN Statistics Division.

COICOP is a six-level hierarchy. The top is 12-14 broad categories of household spending. The leaves are small enough to contain only items that substitute for each other.

| Level | Name | Example |
|---|---|---|
| 1 | **Division** | "01 — Food and non-alcoholic beverages" |
| 2 | **Group** | "01.1 — Food" |
| 3 | **Class** | "01.1.1 — Bread and cereals" |
| 4 | **Sub-class** | "01.1.1.1 — Rice" |
| 5 | **Micro-class** | "01.1.1.1.1 — Rice (white)" |
| 6 | **Elementary aggregate (EA)** | "01.1.1.1.1.01 — Rice, white, 1 kg, retail" |

That six-level structure is **exactly** what `hcpi_coicop` defines — one model per level. The `full_code` field on `hcpi.elementary.aggregate` concatenates the chain into a string like `01.01.01.01.01.01`.

### Why the elementary aggregate matters

The **elementary aggregate (EA)** is the deepest level — fine-grained enough that all items inside it are essentially substitutes for each other. "Rice, white, 1 kg, retail" is an EA; the items inside it are different brands or origins of the same product.

This matters for two reasons:

1. **Below EA, you don't weight.** Items inside an EA are treated as equivalent — you average their prices using a simple geometric mean (no weighting).
2. **Above EA, you weight.** EAs get composite weights based on household expenditure surveys. Those weights flow up: each class's weight is the sum of its sub-classes' weights, and so on.

So the CPI is computed in two halves: a *non-weighted* averaging below the EA, then a *weighted* aggregation up the hierarchy.

## Items, outlets, observations

Every month, enumerators visit outlets and record prices for predefined items. The data model:

```
hcpi.item              hcpi.outlet                hcpi.outlet.item
─────────              ───────────                ─────────────────
"Rice, white, 1 kg"    "Kikuubo Market, stall 4"  Junction record —
"Bread, 600 g loaf"    "Nakumatt Supermarket"     this outlet sells
"Cooking oil, 1 L"     "Owino Market, stall 12"   this item
                                                  Has a base_price
                                                  Has observations
```

Each `hcpi.outlet.item` is a *plan* — "we observe rice prices at this market every month". Each `hcpi.outlet.item.observation` is one actual recorded price: "on 2025-09-15, white rice 1 kg cost UGX 4,200 at Kikuubo Market stall 4."

Many outlets × many items × many months = millions of observations over the life of the system. That's why `hcpi.outlet.item.observation` has explicit PostgreSQL indexes on `(outlet_item_id, observation_date DESC)` — queries by item and time window are constant.

## The basket — consumption segments

Households don't all consume the same way. Rural and urban households have different spending patterns; high-income and low-income households even more so. A single national CPI hides those differences.

HCPI's solution: **consumption segments** (called "baskets" in the UI). Each segment is a grouping of similar consumers — e.g. "urban high-income", "rural", "Kampala metropolitan". Outlets are tagged with the consumption segment they serve.

```
                ┌─ Urban high-income basket ◀── observations from supermarkets
hcpi.outlet ───┤
                └─ Rural basket             ◀── observations from rural markets
```

You compute one CPI per basket per month (`hcpi.basket.index`), then roll them up nationally weighted by population (`hcpi.national.index`). That's the whole top half of the `hcpi_index` module.

## Price relatives — the unit of computation

You don't add prices directly when computing a CPI. You compute **price relatives** — ratios of the current price to the base-period price — and aggregate those.

If rice was UGX 4,000 in the base period and is UGX 4,400 now, the price relative is **4400 / 4000 = 1.10** (rice has gone up 10%).

The CPI is then the weighted geometric mean of all price relatives in the basket, multiplied by 100 to put it on the same scale as the reference period (`100` in the base).

This means **the base price matters**. If the base-period price is wrong or missing, every subsequent month's CPI is wrong. That's why `hcpi.outlet.item` has a `base_price` field, and why the data-collection workflow flags zero-price observations aggressively (see [the zero-price validation](../understanding-the-codebase/module-reference.md#hcpi_computation)).

### Long-term vs. short-term

You'll see two kinds of price relative in the code:

- **Long-term price relative** — current price ÷ base-period price. Used in the CPI itself.
- **Short-term price relative** — current price ÷ previous-month price. Used for month-over-month change detection and outlier flagging.

Both are stored on `hcpi.outlet.item.observation`.

## Outlier detection — the Tukey algorithm

Raw observations are noisy. An enumerator might mis-type a digit (UGX 42,000 instead of UGX 4,200). A vendor might give a wholesale price by accident. If those bad observations flow straight into the CPI, the index lurches.

HCPI uses the **Tukey algorithm** to flag observations as outliers before they reach the index.

Tukey's method is simple but robust:

1. For each (item, basket, month) group of observations, compute the **median** and the **first** and **third quartiles** (Q1 and Q3).
2. Compute the **interquartile range** (IQR) = Q3 − Q1.
3. Any observation outside the range `[Q1 − k·IQR, Q3 + k·IQR]` is flagged as an outlier. The multiplier `k` is configurable (typically 1.5 or 3).

Median and IQR are *robust* — a few extreme values don't sway them. That's why Tukey is preferred over a simple mean-and-standard-deviation test.

In HCPI:

- Each `hcpi.data.collection.line` has `is_outlier` and `is_inlier` computed booleans.
- Supervisors review flagged outliers during the `validation` state of the questionnaire workflow.
- An outlier can be **manually accepted** (`is_rejected = False`) if the supervisor confirms it's a real price; or **rejected** (excluded from index computation).

## The collection workflow

Pulling the data side together:

```
┌──────────────────────────────────────────────────────────────────┐
│ MONTH START                                                       │
│ 1. Supervisor opens hcpi.data.collection (questionnaire) per      │
│    outlet × month. Pre-populated with the outlet's items.         │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ FIELD COLLECTION                                                  │
│ 2. Enumerator visits outlets, records observations on the         │
│    questionnaire lines (web UI or Flutter mobile app).            │
│    state: draft → survey                                          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ STANDARDIZATION                                                   │
│ 3. Observed quantities (e.g. "350 g sachet") are normalized       │
│    to the item's standard unit (e.g. "per kg").                   │
│    state: survey → standardization                                │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                        │
│ 4. Tukey outlier flagging runs. Supervisor reviews outliers,      │
│    accepts or rejects each. Zero-price warnings checked.          │
│    state: standardization → validation                            │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ READY FOR INDEX                                                   │
│ 5. Questionnaire marked done. Validated observations feed         │
│    hcpi_index computation jobs.                                   │
│    state: validation → done                                       │
└──────────────────────────────────────────────────────────────────┘
```

This is the state machine on `hcpi.data.collection`. Day 10 of the training walks through it with the validation algorithms in detail.

## The index computation chain

Once observations are validated, indices roll up the COICOP hierarchy and the basket structure:

```
                                       ┌─▶ hcpi.basket.elementary.aggregate.index
                                       │   (per basket, per EA, per month)
   hcpi.outlet.item.observation ───────┤
   (validated, per outlet, per         │
    item, per month)                   │
                                       └─▶ aggregated up the COICOP chain ─▶
                                           hcpi.basket.micro.class.index
                                           hcpi.basket.sub.class.index
                                           hcpi.basket.class.index
                                           hcpi.basket.group.index
                                           hcpi.basket.division.index
                                           hcpi.basket.index (one per basket)
                                                          │
                                                          ▼
                                                weighted across baskets
                                                          │
                                                          ▼
                                          hcpi.national.index
                                          (the headline CPI number)
```

Each of those `hcpi.basket.<level>.index` and `hcpi.national.index` is a real Odoo model in `hcpi_index/`. Wizards triggered from the UI dispatch the actual computation as background jobs through `queue_job`, with progress shown via `web_progress`.

## Special and classification indices

Beyond the COICOP roll-up, HCPI supports two additional kinds of grouping:

- **Special indices** — custom categories like "food inflation" or "core CPI" (which excludes volatile items like food and fuel). Each is a `hcpi.special.index.category` with rules for which items roll up into it.
- **Classification indices** — additional custom groupings parallel to special. Used for ad-hoc analyses or for non-COICOP classifications (e.g. by durability, by origin).

Both have the same three-tier model pattern: category → per-basket → national.

## Where the numbers come from — full chain

To make it concrete, here's the lifecycle of a single observation:

1. **Enumerator records**: "1 kg white rice, Kikuubo stall 4, 2025-09-15, UGX 4,200" → `hcpi.outlet.item.observation` row.
2. **Standardization** confirms 1 kg is the standard quantity → no adjustment needed.
3. **Validation**: Tukey on all white-rice observations in the same basket × month flags one stall (UGX 42,000 — a typo) as an outlier. Supervisor rejects it. Our UGX 4,200 is inlier.
4. **EA-level aggregation**: geometric mean of all white-rice observations in Kikuubo basket → say UGX 4,300 average. Price relative vs base = 4300 / 4000 = 1.075.
5. **Roll up the COICOP chain**: 1.075 contributes to the "Rice" sub-class index, which contributes to the "Bread and cereals" class, then "Food", then "Food and non-alcoholic beverages" division — at each level weighted by composite weights.
6. **Roll up the baskets**: Kikuubo basket's division index combines with all other baskets' division indices, weighted by population → national division index.
7. **National headline CPI**: weighted average across all 12-14 divisions → the single number reported.

Every level is its own model in `hcpi_index/`, with the wizards that compute them in the same module.

## Where to go from here

Now that you have the concepts:

- **[Glossary](glossary.md)** — quick reference for any term that's still fuzzy.
- **[Module Reference](../understanding-the-codebase/module-reference.md)** — re-read the per-module section knowing what each model represents in the real world.
- **[Country Variants](../understanding-the-codebase/country-variants.md)** — see how the same model is anchored to different geographic hierarchies.
- **Day 3+ of training** (coming soon) — hands-on with models, validation, and index computation.
