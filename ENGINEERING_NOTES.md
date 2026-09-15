# Engineering Notes & STAR Stories

*Semantic modeling decisions and the debugging behind the Sales and Inventory dashboards.*

*Personal portfolio project · synthetic data only (no employer data) · built to demonstrate DP‑600 / DP‑800 hands‑on.*

This document records **how** the model was built and **why** — the grain decisions, the DAX patterns, and two honest debugging stories where visuals looked correct but hid real modeling defects. It's meant to be read alongside the dashboards in this repo.

---

## Project at a glance

| | |
|---|---|
| **Platform** | Microsoft Fabric — Lakehouse (OneLake), Direct Lake semantic model, Dataflow Gen2, Data Pipeline |
| **Data** | Synthetic supply-chain data — no real employer data |
| **Model** | Star schema — 2 fact tables, 5 conformed dimensions |
| **Authoring** | Power BI Desktop live-connected to the Direct Lake semantic model; versioned in PBIP / TMDL |
| **Dashboards documented here** | Sales, Inventory |

---

## The semantic model

Two fact tables share a set of conformed dimensions in a classic star schema. The two facts sit at **different grains** on purpose — that single fact drives almost every measure decision below.

| Table | Type | Grain (what one row means) |
|---|---|---|
| `fact_orders` | Transactional | One row per **order line** (one product per order): Revenue, Cost, Quantity, OnTime, InFull, OTIF, Status, DemurrageCost, OrderDate, ShipDate |
| `fact_inventory` | Snapshot | One row per **product × warehouse × month**: OnHandQty, InventoryValue, SnapshotDate |
| `dim_date` | Dimension | One row per calendar day (marked as the date table) |
| `dim_product` | Dimension | One row per product (UnitPrice, UnitCost, Category) |
| `dim_customer` | Dimension | One row per customer (Region) |
| `dim_warehouse` | Dimension | One row per warehouse (Region) |
| `dim_carrier` | Dimension | One row per carrier |

**Relationships** (all single-direction, one-to-many from dimension to fact): `dim_date`, `dim_product`, and `dim_warehouse` filter **both** facts; `dim_customer` and `dim_carrier` filter `fact_orders` only.

### Grain and granularity — the idea that underpins everything

**Grain** is the business meaning of a single row. Making it explicit *before* writing measures is what keeps totals honest. My two facts have deliberately different grains:

- **Orders are transactional and daily** — one row per order line — so order measures are **fully additive**: Revenue, Cost, and Quantity can be summed across any dimension (product, region, month) and stay correct.
- **Inventory is a monthly snapshot** — one row per product-warehouse-month — so inventory measures are **semi-additive**: they sum across product and warehouse, but you **cannot sum a stock level across time**. Adding January's stock to February's would double-count goods that never moved. Across time you take the **last** snapshot (closing stock) or an **average** — never a `SUM`.

That one distinction is the reason the Inventory dashboard needed the closing-stock and days-on-hand logic described later, while the Sales dashboard could sum freely.

### Conformed dimensions — why they earn their keep

Because `dim_product`, `dim_warehouse`, and `dim_date` filter **both** facts, I can place an orders measure and an inventory measure on the *same* visual and have them respect the *same* filters. That's what makes **Days of Inventory** possible at all — it divides an inventory measure by an orders measure for the same product and period. Without conformed dimensions, the two facts couldn't be compared.

### Additive vs semi-additive — the measure cheat-sheet

| Measure type | Examples | How to aggregate across **time** |
|---|---|---|
| Additive | Revenue, Cost, Quantity, Order Count | `SUM` — safe over any dimension |
| Semi-additive | InventoryValue, OnHandQty | Last snapshot (closing) or average — **never** `SUM` over time |

### Disconnected helper tables — a tool, not a mistake

Some tables in the model are intentionally **not** related to any fact — a band-range table for segmentation and small selector tables for field parameters. Disconnected tables are perfectly valid **when they drive a measure**. They become a bug only when their columns are dropped **directly** onto a visual — which is exactly what went wrong in the Inventory story below.

---

## STAR Story 1 — Sales Dashboard

**Situation** — The first analytics page: an executive **sales view** on the Direct Lake model — revenue, volume, margin, growth trend, top customers, and regional/category mix.

**Task** — Deliver commercial KPIs leadership could trust and self-serve: correct time intelligence, the right grain, and every visual tied back to one consistent set of base measures.

**Action**

- **Grain & date table.** Modeled on `fact_orders` at order-line grain, and marked `dim_date` as the date table so time-intelligence functions work reliably in Direct Lake.
- **Base measures as a single source of truth.** Built a small core — `Total Revenue = SUM(fact_orders[Revenue])`, `Total Volume = SUM(fact_orders[Quantity])`, `Gross Margin % = DIVIDE([Total Revenue] - [Total Cost], [Total Revenue])`, and `Avg Order Value` dividing revenue by the **distinct order count** (not the row count — a grain detail that would otherwise understate AOV) — then sourced every visual from these.
- **Time intelligence.** `Revenue YoY %` via `SAMEPERIODLASTYEAR`, a running-sum trend, and a rolling **3-month moving average** over a `DATESINPERIOD` window.
- **Reusable variance logic.** Standardized percentage-change calculations with a DAX **user-defined function** so the same variance pattern is applied consistently across measures.
- **Breakdowns via conformed dims.** Region, category, and top-customer views all driven by the shared dimensions, with drill-through to detail.

**Result** — A commercial page reporting **₹1.2bn** total revenue, **15M** units, **33.05%** gross margin, and **+47.78%** revenue YoY, with a growth trend, top-10 customers, and regional/category mix — every visual reconciling to the same base measures.

**What I learned** — Anchor a report on a few well-defined base measures rather than ad-hoc per-visual calculations; always mark the date table; and respect grain in ratios (AOV must divide by distinct orders).

**Design refinements (completed)**

- Recolored `Revenue YoY %` so positive growth reads green, not red.
- Replaced the Region pie with a horizontal bar (keeping a single donut for Order Count) to remove a redundant circular encoding and make values easier to compare.
- Simplified the trend chart to Total Revenue plus a 3-month moving average on one scale, so the moving-average signal is readable (the running sum had been dwarfing it).
- Applied one shared theme across the Sales and Inventory pages for a consistent, governed look.

---

## STAR Story 2 — Inventory Dashboard

**Situation** — An **inventory-health page**: Days of Inventory (DOI), turnover, stock value by warehouse and category, and SKU ageing bands (Healthy / Watch / Slow / Obsolete). The first version *looked* right but hid several defects.

**Task** — Before publishing, make every number **reconcile** and behave **sensibly under any filter** — not merely look plausible.

**Action** — I found and fixed four issues:

**1. Band fan-out (a cartesian join).** The ageing `Band` came from a **disconnected** table, so dropping its column directly onto visuals paired *every* SKU with *every* band — SKU-1 appeared as Healthy **and** Watch **and** Slow **and** Obsolete with identical values. This duplicated stock value and inflated the band chart to ~77M against the true **75.70M**.
*Fix:* removed the disconnected column from the table and derived each SKU's band with a `SWITCH`-on-DOI measure so it resolves to exactly one band; rebuilt the band **bar chart** with **dynamic segmentation** — a min/max range table plus a `SUMX`-over-products measure that adds each SKU's stock to only the band its DOI falls in. The four bars now sum exactly to 75.70M.

**2. Days of Inventory that collapsed under filtering.** The original DOI multiplied by "days in the current filter context" (`COUNTROWS(VALUES(dim_date[Date]))`), so selecting a single month dropped DOI to ~8 days — a false empty-warehouse signal.
*Fix:* rebuilt it as a **days-on-hand** measure — closing stock ÷ a fixed **trailing-365-day** COGS, scaled to days — anchored to a fixed window. DOI is now stable and comparable across selections (**≈128 days** unfiltered, **≈124** for a single month, instead of crashing).

**3. Overstated COGS.** The cost measure summed **all** orders, including unshipped "Open" ones, overstating cost of goods *sold* and understating DOI.
*Fix:* filtered COGS to `Status = "Closed"` (shipped) orders, so DOI reflects true sell-through. This correctly **raised** DOI from ~100 to ~128 — a more honest number.

**4. Fragile closing stock.** `Closing Stock Value` used `LASTDATE(dim_date[Date])`, which can land on a calendar day that has **no** monthly snapshot and return a wrong or blank value.
*Fix:* rebuilt it to land on the last date that **actually has** inventory (`MAX(fact_inventory[SnapshotDate])`) — a correct semi-additive closing value.

**Result** — The band bars reconcile exactly to total stock value (parts sum to the whole), DOI is filter-stable and honest, and the totals are correct **weighted ratios** — the grand-total DOI is value-weighted and dominated by slow-moving Resins, *not* an average of the per-SKU rows, which is the correct behaviour to expect and explain.

**What I learned** — *Validate* a model — confirm totals reconcile and measures behave under filtering — rather than trusting a plausible-looking number. This project made four concepts concrete for me: **cartesian fan-out** from disconnected tables, **filter context**, **semi-additivity**, and **dynamic segmentation**.

---

## Appendix — key DAX measures

These are the measures behind the Inventory fixes, using the model's actual column names.

**Each SKU into one band (table):**

```dax
SKU Band =
IF (
    NOT HASONEVALUE ( dim_product[ProductKey] ),
    BLANK (),
    VAR doi = [Days of Inventory (DOI)]
    RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( doi ), BLANK (),
        doi <= 30,  "Healthy",
        doi <= 60,  "Watch",
        doi <= 120, "Slow",
        "Obsolete"
    )
)
```

**Stock value per band (dynamic segmentation, bar chart):**

```dax
Stock Value in Band =
VAR MinDOI = SELECTEDVALUE ( BandRange[MinDOI] )
VAR MaxDOI = SELECTEDVALUE ( BandRange[MaxDOI] )
RETURN
SUMX (
    VALUES ( dim_product[ProductKey] ),
    VAR doi = [Days of Inventory (DOI)]
    RETURN IF ( doi > MinDOI && doi <= MaxDOI, [Closing Stock Value (v2)] )
)
```

**COGS on shipped orders only:**

```dax
COGS (Sold) =
CALCULATE ( SUM ( fact_orders[Cost] ), fact_orders[Status] = "Closed" )
```

**Semi-additive closing stock:**

```dax
Closing Stock Value (v2) =
VAR LastStockDate =
    CALCULATE ( MAX ( fact_inventory[SnapshotDate] ), ALLSELECTED () )
RETURN
CALCULATE (
    SUM ( fact_inventory[InventoryValue] ),
    fact_inventory[SnapshotDate] = LastStockDate
)
```

**Filter-stable Days of Inventory (days on hand):**

```dax
DOI (Days on Hand) =
VAR DaysWindow = 365
VAR TrailingCOGS =
    CALCULATE (
        [COGS (Sold)],
        DATESINPERIOD ( dim_date[Date], MAX ( dim_date[Date] ), -DaysWindow, DAY )
    )
RETURN
    DIVIDE ( [Closing Stock Value (v2)], TrailingCOGS ) * DaysWindow
```

**Representative Sales patterns** (standard implementations behind the commercial page):

```dax
Total Revenue   = SUM ( fact_orders[Revenue] )
Gross Margin %  = DIVIDE ( [Total Revenue] - [Total Cost], [Total Revenue] )
Revenue YoY %   =
    DIVIDE (
        [Total Revenue] - CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( dim_date[Date] ) ),
        CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( dim_date[Date] ) )
    )
```
