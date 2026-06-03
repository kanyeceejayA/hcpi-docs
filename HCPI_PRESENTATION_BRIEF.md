# HCPI Computation — Presentation Brief & Visual Info-Dump

**Purpose of this file:** a self-contained briefing to paste into Claude (claude.ai / artifacts / a presentation generator) so it can build a **beautiful, animated explainer** — slides, SVG diagrams, and motion illustrations — of how the Harmonised Consumer Price Index (HCPI) is computed. Everything the design tool needs (story arc, exact numbers, worked example, palette, motion ideas, SVG specs) is here. It does **not** need access to the codebase.

---

## 0. Prompt you can give the design tool

> "Using the brief below, create an animated, presentation-quality explainer of how a national inflation index is built from individual shop prices. Produce a sequence of slides with SVG diagrams and subtle motion (numbers counting up, a price climbing a pyramid, ratios forming). Audience: government statisticians and developers. Tone: authoritative, clean, data-journalism style. Use the supplied palette and worked example. Each diagram should be an inline SVG with smooth CSS/SMIL animation."

---

## 1. The story in one arc (the spine of the deck)

A single price quote in a market becomes one number that describes a whole country's cost of living. The journey has **six acts**:

1. **The Quote** — a fieldworker writes down one price, in one shop, on one day.
2. **The Ratio** — that price becomes a *price relative*: how much it changed vs a reference price. (Two flavours: vs the base period, and vs last month.)
3. **The Cleanup** — wild values are flagged and removed (the Tukey check); gaps are filled (imputation).
4. **The Foundation** — for the smallest product group, the ratios are blended with a **geometric mean** → the *elementary aggregate index* (a Jevons index).
5. **The Climb** — indices are **weighted by household spending** and summed up a classification pyramid: micro-class → sub-class → class → group → division.
6. **The Headline** — all the pieces combine into the **national inflation number** (plus regional, special, and classification cuts).

**The single most important visual idea:** two different kinds of averaging.
- **Bottom (geometric mean)** — fair blending of ratios.
- **Above (weighted arithmetic mean)** — importance-weighted by spending.

---

## 2. Glossary for on-screen labels (keep these exact words)

| Term | One-line caption for a slide |
|---|---|
| Observation | "One price · one item · one shop · one month" |
| Price relative | "A ratio: this price ÷ a reference price" |
| Elementary Aggregate (EA) | "The smallest published product group, e.g. *Bread*" |
| Jevons index | "A geometric mean of price ratios" |
| COICOP hierarchy | "The international product-classification ladder" |
| Weight | "How much households spend on it — its importance" |
| Consumption segment / basket | "A population group, e.g. *Kampala Urban*" |
| Laspeyres-type aggregation | "Importance-weighted averaging up the ladder" |
| Chaining | "Linking each month's move onto the last" |

---

## 3. The exact formulas (render as elegant typeset math)

**Price relative**
$$\text{PR} = \frac{P_{\text{now}}}{P_{\text{reference}}}$$
- Long-term: reference = base period.
- Short-term: reference = previous month.

**Elementary aggregate index — Jevons (geometric mean × 100)**
$$I_{\text{EA}} = \left(\prod_{i=1}^{n} \text{PR}_i\right)^{1/n}\times 100 = \exp\!\left(\tfrac{1}{n}\sum_i \ln \text{PR}_i\right)\times 100$$

**Chained short-term index**
$$I_t = \text{GeoMean}(\text{PR}_t)\times I_{t-1}$$

**Aggregation up the ladder — weighted arithmetic mean (Laspeyres-type)**
$$I_{\text{parent}} = \frac{\sum_c I_c\, w_c}{\sum_c w_c}$$

**Tukey outlier bounds** (after trimming the most extreme 5% each side)
$$\text{upper} = \bar g + 2.5(\bar u - \bar g), \qquad \text{lower} = \bar g - 2.5(\bar g - \bar\ell)$$
where $\bar u$ = mean of ratios above 1, $\bar\ell$ = mean below 1, $\bar g$ = overall mean.

---

## 4. A fully worked example (use these exact numbers in animations)

Build the deck around **"Bread"** as the elementary aggregate, in the **Kampala Urban** basket.

### 4a. Three price quotes for bread this month

| Shop | Price now (UGX) | Base price | Long-term relative |
|---|---|---|---|
| Shop A | 2,500 | 2,000 | 1.25 |
| Shop B | 2,200 | 2,000 | 1.10 |
| Shop C | 2,700 | 2,250 | 1.20 |

### 4b. The EA index for Bread (geometric mean × 100)

$$\text{GeoMean}(1.25,\,1.10,\,1.20) = (1.25\times1.10\times1.20)^{1/3} = (1.65)^{1/3} \approx 1.1829$$

$$\boxed{I_{\text{Bread}} \approx 118.3}$$

*(Caption: "Bread costs ~18.3% more than the base period.")*
*(Animation idea: show the three ratios flowing into a blender, the product 1.65 forming, the cube-root pulling it back to 1.1829, then ×100 snapping to 118.3.)*

### 4c. Climbing one rung — "Bread & Cereals" class from three EAs

| EA | Index | Weight (spending share) | Index × Weight |
|---|---|---|---|
| Bread | 118.3 | 0.50 | 59.15 |
| Rice | 110.0 | 0.30 | 33.00 |
| Maize flour | 130.0 | 0.20 | 26.00 |
| **Total** | | **1.00** | **118.15** |

$$I_{\text{class}} = \frac{59.15+33.00+26.00}{0.50+0.30+0.20} = \boxed{118.15}$$

*(Caption: "Bread carries the most weight, so it pulls the class index closest to itself." Animation idea: three bars of different *widths* (= weights) and *heights* (= index); the combined bar lands at the weighted average, visibly nearer the widest bar.)*

### 4d. The Tukey check in action

Suppose one shop reports bread at **9,000 UGX** (relative 4.5) — a typo. With most ratios near 1.1–1.3, the upper limit lands well below 4.5, so the 4.5 is **flagged red and removed** before the index is built.

*(Animation idea: a scatter of dots near 1.2; one rogue dot at 4.5 flashes red and is swept out past a glowing threshold line.)*

---

## 5. Slide-by-slide storyboard

| # | Title | Visual | Motion |
|---|---|---|---|
| 1 | **From a shop shelf to a nation's inflation** | hero: a price tag morphing into a line chart | price tag flips, chart draws in |
| 2 | **The atom: one observation** | a card: item · shop · date · price | card flips to reveal the 4 facts |
| 3 | **Prices can't be averaged — ratios can** | two price tags → a ÷ symbol → "1.25" | division animates, units cancel and vanish |
| 4 | **Two kinds of ratio** | split screen: "vs base period" / "vs last month" | a timeline with two arrows, one long, one short |
| 5 | **Cleaning: the Tukey check** | scatter of ratios with threshold fences | rogue dot ejected; fences slide to position |
| 6 | **Filling gaps: imputation ladder** | small staircase climbing COICOP levels | a missing tile fills from the level above |
| 7 | **The foundation: geometric mean** | three ratios → blender → Jevons index 118.3 | the worked bread example (4b) |
| 8 | **Geometric vs arithmetic — why it matters** | a doubling (×2) and a halving (×0.5) | geo mean returns to 1.0; arithmetic drifts to 1.25 |
| 9 | **The weight: spending = importance** | bars of varying width | widths labelled with shares summing to 100% |
| 10 | **Climbing the pyramid** | the COICOP pyramid, EA → division | a glowing index value rises rung by rung |
| 11 | **Basket → National** | several baskets merging into one flag/number | strata streams converge |
| 12 | **The other cuts** | branches: Special, Classification, Domestic | three side-branches sprout from the trunk |
| 13 | **One method, many countries (harmonised)** | EAC map; identical formula badge on each | flag pins drop; a shared "✓ identical math" stamp |
| 14 | **The headline number** | a single big CPI figure counting up | odometer-style count to the final value |

---

## 6. The COICOP pyramid (the signature diagram)

Build this as the recurring spine graphic. Bottom-to-top:

```
                ┌───────────────────────────┐
                │     NATIONAL INDEX        │   ← weighted across all baskets
                └───────────────────────────┘
              ┌───────────────────────────────┐
              │   BASKET (all-items) INDEX    │   ← one population segment
              └───────────────────────────────┘
            ┌───────────────────────────────────┐
            │            DIVISION               │
            ├───────────────────────────────────┤
            │             GROUP                 │
            ├───────────────────────────────────┤
            │             CLASS                 │   ← weighted arithmetic mean
            ├───────────────────────────────────┤      (Laspeyres-type) at every rung
            │           SUB-CLASS               │
            ├───────────────────────────────────┤
            │          MICRO-CLASS              │
            └───────────────────────────────────┘
          ╔═══════════════════════════════════════╗
          ║   ELEMENTARY AGGREGATE  (e.g. Bread)  ║   ← GEOMETRIC mean (Jevons)
          ╚═══════════════════════════════════════╝
        ·  ·  ·  individual price observations  ·  ·  ·   ← raw quotes
```

**Visual rule:** color the **EA band gold** (geometric mean) and everything above it **green** (weighted mean) to reinforce "two methods." A glowing dot should travel up the pyramid carrying the index value.

---

## 7. SVG / animation specifications

- **Format:** inline SVG, `viewBox="0 0 1280 720"` (16:9). Prefer SMIL `<animate>` or CSS `@keyframes`; keep each animation 1.5–3s, ease-in-out, looping where ambient.
- **Signature motions:**
  - *Ratio formation:* two number tags slide together, a ÷ appears, units fade, the result scales up.
  - *Geometric blend:* ratios drop into a funnel; the product value forms; a root symbol contracts it; ×100 snaps the index.
  - *Weighted bars:* `<rect>` widths animate from 0 to weight-proportional; the resulting average line slides to the weighted position.
  - *Pyramid climb:* a `<circle>` follows a `<path>` up the tiers via `<animateMotion>`, the index label updating at each tier.
  - *Outlier ejection:* dots `<circle>` jitter near 1.2; one tweens to opacity 0 and translates off-canvas past a dashed threshold line.
  - *Odometer:* final CPI number counts up via incremental `<text>` updates.
- **Counting numbers:** animate from 100.0 → final value with 1-decimal precision.
- **Accessibility:** every SVG gets a `<title>` and `<desc>`; motion respects `prefers-reduced-motion` (provide a static fallback frame).

---

## 8. Palette & type

| Role | Hex | Use |
|---|---|---|
| Ink / text | `#0f172a` | headings, body |
| Geometric-mean gold | `#f59e0b` (fill `#fde68a`) | the EA band, Jevons step |
| Weighted-mean green | `#16a34a` (fill `#bbf7d0`) | aggregation steps |
| Accent blue | `#2563eb` | flows, arrows, links |
| Alert red | `#dc2626` | outliers, zero-price block |
| Surface | `#f8fafc` | slide backgrounds |
| Muted | `#64748b` | captions, footnotes |

**Type:** a clean geometric sans (Inter / Söhne / Helvetica Neue) for UI; a mono (JetBrains Mono / IBM Plex Mono) for formulas and code snippets.

---

## 9. Key facts the narration must get right (accuracy guardrails)

- The EA index is a **geometric mean of price relatives** (a **Jevons** index), `EXP(AVG(LN(relative))) × 100`. Only **positive** relatives are included; zeros/negatives are excluded as "no data."
- Everything **above** the EA is a **weighted arithmetic mean** with **expenditure weights** — a **Laspeyres-type** construction.
- Weights are stored **only at the EA level**; higher weights are the **sum** of the EA weights beneath.
- There are **two price relatives**: **long-term** (vs base period) and **short-term** (vs previous month, **chained**).
- Outlier detection is the **Tukey algorithm**: trim 5% each tail, then bounds at **2.5×** the typical move beyond the mean.
- A **zero price halts computation** from that month forward — a deliberate data-quality gate, not a bug.
- The system is **harmonised**: the computation code is **identical across all countries** (Uganda, Kenya, Tanzania, South Africa, …); only the **geographic/data-entry layer differs**. This is the whole point of a *Harmonised* CPI for the East African Community.

**Do NOT claim:** that raw prices are averaged (they are not); that the bottom level is weighted (it is not — that's where geometric mean stands in for missing sub-weights); or that countries use different formulas (they do not).

---

## 10. Pull-quotes for slides

- *"You can't average the price of rice and the price of fuel — but you can average how much each one changed."*
- *"At the bottom, a geometric mean. Above it, spending decides who matters most."*
- *"Bread carries half the weight, so it pulls the class index closest to itself."*
- *"One formula, many flags: harmonised by design."*
- *"A single impossible zero stops the whole calculation — on purpose."*

---

*Companion document: [HCPI_COMPUTATION_FLOW.md](./HCPI_COMPUTATION_FLOW.md) — the full technical walk-through with file/line references.*
