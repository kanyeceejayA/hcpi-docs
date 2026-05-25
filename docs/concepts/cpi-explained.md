# CPI Explained

This page is for **anyone who needs to understand what HCPI is for** — project sponsors, new statisticians, technical leads joining the project, or developers who want the big picture before diving into code. No statistics background assumed. No code.

By the end you'll know what a CPI is, how it's different from "inflation", the four building blocks every CPI is made of, and how the HCPI system turns the raw data into the headline number you see in the news.

For the developer-facing version with model names and computation worked out, see [CPI Concepts](cpi-concepts.md).

## What is a CPI?

A **Consumer Price Index (CPI)** is one number — like **117.3** — that says how expensive everyday life has become compared to some earlier point in time.

The earlier point is called the **base period**, and by convention is set to **100**. So:

- A CPI of **100** means prices are the same as the base period.
- A CPI of **117.3** means prices are **17.3% higher** than the base period.
- A CPI of **95** means prices are **5% lower** than the base period (rare, but possible).

The "consumer" in "Consumer Price Index" means it's about households, not businesses or government. The "index" means it's a *comparison* number, not an *amount* — it tells you how prices have *changed*, not what things actually cost.

## CPI vs. inflation

People often use these words interchangeably. They're related, but not the same thing.

| | CPI | Inflation |
|---|---|---|
| **What it is** | A *level* — how prices compare to the base period | A *rate* — how fast prices are changing |
| **Looks like** | "CPI is 117.3" | "Inflation is 5.2% this year" |
| **Where it comes from** | Direct computation from prices | Calculated *from* the CPI |
| **Time dimension** | A snapshot of one period (usually one month) | A change between two periods |

You can think of CPI as the **odometer** (where are we now?) and inflation as the **speedometer** (how fast are we moving?).

### How they connect

Inflation is computed from the CPI:

```
                  CPI now  −  CPI a year ago
Inflation rate  = ──────────────────────────  ×  100%
                       CPI a year ago
```

If CPI was 110 last year and is 117.3 this year:

```
              117.3 − 110
Inflation  =  ───────────  ×  100%   ≈   6.6%
                  110
```

The CPI is the *underlying measurement*; inflation is the *change in that measurement*.

### Year-on-year vs. month-on-month

Inflation can be measured over different time spans:

- **Year-on-year inflation** — compares this month to the same month last year. The number you see in news headlines.
- **Month-on-month inflation** — compares this month to last month. More sensitive to short-term blips.
- **Annual inflation (calendar)** — January-to-December change.

All of them come out of the same CPI series. HCPI stores the CPI; the inflation rates are derived in reports and dashboards.

### Other terms in the same family

| Term | Meaning |
|---|---|
| **Deflation** | Negative inflation — prices going down. CPI is falling. |
| **Disinflation** | Inflation slowing down (still positive). CPI still rising, but slower than before. |
| **Core CPI** | A version of the CPI that excludes volatile categories (food, fuel). Smooths out short-term jumps. |
| **Headline CPI** | The all-items CPI — the number most often reported. |
| **Real vs. nominal** | "Nominal" = unadjusted for inflation. "Real" = adjusted using the CPI. "Real wages" means wages after subtracting price increases. |

## The four building blocks

Every CPI in the world — from Uganda's to the United States' — is built from the same four pieces. Once you understand these, the rest of the system makes sense.

### 1. Items

**What it is:** a specific good or service that gets priced. Not "food", but "white rice, 1 kg, retail". Not "transport", but "bus fare, Kampala to Entebbe, one-way".

The list is huge — typically 200-500 items per country — chosen so it represents what households actually buy.

In HCPI: the `hcpi.item` model. See the [Module Reference](../understanding-the-codebase/module-reference.md#hcpi_item).

### 2. Outlets

**What it is:** the places where prices get observed. A specific market stall, a supermarket, a roadside seller.

You don't price an item just once — you price it at many outlets across the country, because the same item can cost different amounts in different places. The full picture comes from sampling many outlets.

In HCPI: the `hcpi.outlet` model.

### 3. Observations

**What it is:** one actual recorded price. *On this date, at this outlet, this item cost this much.*

For example: *"On 16 September 2025, at Kikuubo Wholesale Market Stall 4, 1 kg of white rice cost UGX 4,200."*

Millions of observations accumulate over the life of a CPI system. Each one is a single data point feeding the index calculation.

In HCPI: the `hcpi.outlet.item.observation` model.

### 4. Weights

**What it is:** a number attached to each item (and each category of items) representing how *important* it is to a typical household.

Bread matters more than caviar — most households buy bread weekly and never buy caviar — so bread has a much bigger weight in the index.

Weights typically come from a national household expenditure survey, which asks thousands of households what they actually spent money on. Without weights, a 10% jump in caviar prices would affect the index just as much as a 10% jump in bread prices, which would obviously be wrong.

In HCPI: the `composite_weight` field on the COICOP hierarchy.

## How items get grouped: COICOP

Five hundred items can't sit in a flat list — they need to be organized so you can ask questions like "did food get more expensive?" or "what's happening to transport costs?"

The international standard for this grouping is **COICOP** (Classification of Individual Consumption by Purpose). It's a six-level tree that every country uses. The top level has 12-14 broad divisions; the bottom level groups items that are direct substitutes.

```
Division           "01 — Food and non-alcoholic beverages"
   │
   ├─ Group           "01.1 — Food"
   │     │
   │     ├─ Class           "01.1.1 — Bread and cereals"
   │     │     │
   │     │     ├─ Sub-class      "01.1.1.1 — Rice"
   │     │     │     │
   │     │     │     ├─ Micro-class    "01.1.1.1.1 — Rice (white)"
   │     │     │     │     │
   │     │     │     │     └─ Elementary aggregate    "Rice, white, 1 kg, retail"
   │     │     │     │              │
   │     │     │     │              └─ Items (specific brands/origins)
```

The bottom level — called the **elementary aggregate** (EA) — is fine-grained enough that all items inside it are interchangeable. Below that, simple averaging works. Above that, you need weights.

This is why COICOP matters: it's the structure that tells the calculation **when to weight and when to just average**.

## How households get grouped: baskets

Households don't all spend money the same way. A rural farmer's basket of typical purchases looks very different from an urban office worker's. A high-income household and a low-income household have different priorities.

If you computed one CPI across everyone, you'd average their experiences and lose all the interesting information. So instead, you compute multiple CPIs — one for each meaningful group of consumers.

In CPI terminology these groups are called **baskets**, and HCPI calls them **consumption segments**. Examples:

- Urban high-income
- Urban low-income
- Rural
- Kampala metropolitan
- Coastal Kenya

Each outlet is tagged with the basket it serves. Observations from supermarket outlets feed the urban-high-income index; observations from rural markets feed the rural index. The same item might have one price in a supermarket and a different price in a rural market — that's how different baskets get different indices.

In HCPI: the `hcpi.consumption.segment` model.

## How everything relates

The pieces fit together like this:

```
                    COICOP hierarchy                 BASKETS
                  ─────────────────────       ─────────────────
                  Division                    Urban high-income
                     ↓                        Urban low-income
                  Group                       Rural
                     ↓                        Kampala metro
                  Class                       ...
                     ↓
                  Sub-class
                     ↓                                ▲
                  Micro-class                         │ outlet is tagged
                     ↓                                │ with its basket
                  Elementary aggregate ──→ contains ITEMS
                                                      ▲
                                                      │ priced at
                                                      │
                                                   OUTLETS
                                                      ▲
                                                      │ produces
                                                      │
                                                   OBSERVATIONS
                                                   (one price at
                                                    one date)
```

Read it like this:

- An **observation** is *one price* for *one item* at *one outlet* on *one date*.
- An **outlet** has many observations over time (one per visit per item) and belongs to one **basket**.
- An **item** belongs to one **elementary aggregate** in COICOP.
- The **elementary aggregate** is the deepest COICOP level — below it, items are equivalent; above it, weighting kicks in.

## How HCPI computes the CPI

Now the building blocks are in place, here's what the system actually does to produce the headline number.

### Step 1: Collect observations

Enumerators visit outlets every month and record prices. Each price becomes one observation in the database. In HCPI this is done through a **questionnaire** — one questionnaire per (outlet × month), pre-populated with the items that outlet sells.

The questionnaires can be filled in two ways: through the web interface, or through the Flutter mobile app (which syncs back when network is available).

### Step 2: Validate the observations

Some observations are wrong — typos, mistaken units, misread prices. Without filtering, those errors would distort the index.

HCPI uses the **Tukey algorithm**, a robust statistical method, to flag observations that are far outside the typical range for an item. For example: if white rice in Kampala-urban typically costs UGX 4,000-4,500 in September, an observation of UGX 42,000 (a likely typo) would be flagged.

Flagged observations don't get automatically thrown out — a **supervisor** reviews each one. Sometimes anomalies are real (a supply shock, a unique outlet), so human judgment finishes the job.

### Step 3: Compute the lowest-level index

Once observations are validated, HCPI computes a **price relative** for each one:

```
                       current price
Price relative  =  ─────────────────────
                    base-period price
```

So if rice was UGX 4,000 in the base period and is UGX 4,200 now, the price relative is **1.05** (rice is 5% more expensive).

For each elementary aggregate × basket × month, HCPI takes the **geometric mean** (a kind of average that handles ratios well) of all the price relatives from observations in that group. The result is the **elementary aggregate index** — one number per EA per basket per month.

### Step 4: Roll up the COICOP hierarchy

The elementary aggregate indices get combined upward through COICOP, weighted by the composite weights at each level:

```
elementary aggregate indices
   ↓  weighted aggregation
micro-class indices
   ↓
sub-class indices
   ↓
class indices
   ↓
group indices
   ↓
division indices
   ↓  weighted aggregation
basket index (one per basket per month)
```

The composite weights come from household expenditure surveys and represent each category's share of typical spending. Bread gets a bigger weight than caviar; food gets a bigger weight than entertainment.

### Step 5: Combine baskets into the national index

Each basket gives one number per month (the basket index). HCPI combines them, weighted by the **population in each basket**, to produce the **national index** — the headline CPI.

That single number is what gets reported in the news and used to compute the inflation rate.

### Step 6: Display

The dashboard shows the headline CPI plus breakdowns by division, by basket, and trends over time. Year-on-year and month-on-month inflation rates are computed on the fly from the stored CPI series.

## What HCPI gives you that a spreadsheet doesn't

A statistics body could (and historically did) compute CPI in Excel. What HCPI adds:

- **Standardized data entry.** Every observation goes through the same form, with the same validation rules.
- **Mobile data collection.** Enumerators record observations in the field, offline if necessary, with the data syncing automatically.
- **Automated outlier detection.** Tukey runs over all observations every month, flagging anomalies before they reach the index.
- **Audit trails.** Every observation, every state transition, every supervisor decision is logged. You can answer "who validated this price?" or "when was this item added to this outlet's catalog?"
- **Multi-basket support.** Computing CPI for 12 baskets in parallel would be a spreadsheet nightmare; HCPI handles it as a normal mode of operation.
- **Reproducible computation.** Re-running an index gives the same answer, every time. No "Excel macro that only Joseph knows how to run".
- **Real-time validation.** The system warns you about zero-price observations, missing items, broken hierarchies *before* they affect published numbers.
- **Mobile + web parity.** Field workers and office statisticians work on the same data with the same rules.
- **Regional aggregation.** The EAC Secretariat hub pulls indicators from member-state HCPI instances over a secure connection for regional comparison — without anyone exporting CSVs.

The CPI methodology hasn't changed in 70 years. What HCPI changes is the **operational machinery** around it.

## Where to go from here

| If you want... | Read |
|---|---|
| To understand the CPI methodology in more depth | [CPI Concepts](cpi-concepts.md) |
| To see a concrete end-to-end example | [Data Flow Walkthrough](data-flow.md) |
| Quick definitions of any term | [Glossary](glossary.md) |
| The technical architecture | [Module Reference](../understanding-the-codebase/module-reference.md) |
| To see how different countries customize HCPI | [Country Variants](../understanding-the-codebase/country-variants.md) |
