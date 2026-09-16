# Supply-Chain Analytics on Microsoft Fabric

A personal, end-to-end analytics project built on **Microsoft Fabric** and **Power BI** to demonstrate semantic modeling, DAX, and modern BI delivery on a supply-chain dataset. 
Built on **synthetic data** generated for the project — no real company data is used.

> **Why this project:** I'm a Power BI developer with SAP BW and supply-chain experience (PL-300, DP-600, DP-800 certified).
> I built this on my own Fabric trial to get hands-on, end-to-end, with the full Fabric stack — from lakehouse to a governed semantic model to a report — the way it's done in production.

---

📄 Engineering notes & STAR stories

The modeling decisions, grain choices, and debugging write-ups (including a cartesian fan-out fix and a filter-stable Days-of-Inventory measure) are documented here:**[ENGINEERING_NOTES.md](ENGINEERING_NOTES.md)**

## What it does

Models a manufacturing supply chain (orders + monthly inventory snapshots) and reports on sales performance, with the semantic model designed to power a multi-dashboard suite (Sales first, with Inventory, Service Level, Open Orders, Demand Planning, Production, and Logistics on the same model).

What this project demonstrates
Semantic modeling — a star schema with two fact tables and five conformed dimensions
Direct Lake semantic model on a Fabric Lakehouse (OneLake)
Data ingestion with Dataflow Gen2 and a Data Pipeline
DAX — time intelligence, semi-additive measures, dynamic segmentation, SWITCH, CALCULATE
Report design — KPIs, drill-through, field parameters, Row-Level Security (RLS)
ALM — version control in PBIP / TMDL
Dashboards

Sales
Executive commercial view: revenue, volume, growth, margin, and customer/region/category mix.

KPIs: 
Total Revenue · Total Volume · Revenue YoY % · Avg Order Value · Gross Margin %

Inventory
Inventory-health view: days of stock, turnover, ageing bands, and stock value by warehouse and category.

KPIs:
Closing Stock Value · Days of Inventory (Days on Hand) · Inventory Turnover · E&O % · Stock-out SKUs

Data model
Star schema — facts in the centre, conformed dimensions around them.

Table	Type	Grain (one row = )
fact_orders	Transactional	one order line (one product per order)
fact_inventory	Snapshot	one product × warehouse × month
dim_date	Dimension	one calendar day (marked date table)
dim_product	Dimension	one product
dim_customer	Dimension	one customer
dim_warehouse	Dimension	one warehouse
dim_carrier	Dimension	one carrier

dim_date, dim_product, and dim_warehouse filter both facts (conformed), which is what lets the two facts be analysed together — for example, Days of Inventory divides an inventory measure by an orders measure for the same product and period.

---
## Architecture

```
CSV (synthetic data)  →  Dataflow Gen2 (Power Query / M)  →  Lakehouse (OneLake)
        →  Direct Lake semantic model (star schema)  →  Power BI report
```

- **Storage:** Lakehouse on OneLake
- **Ingestion:** Dataflow Gen2 (Power Query / M)
- **Model:** Direct Lake semantic model — queries OneLake directly, no import copy
- **Report:** Power BI, live-connected to the shared model

---

## Data model (star schema)

**Fact tables**
- `fact_orders` — one row per order (Revenue, Cost, Quantity, OTIF, Status, DemurrageCost + keys)
- `fact_inventory` — monthly snapshot per product/warehouse (OnHandQty, InventoryValue)

**Dimensions** — `dim_date`, `dim_product`, `dim_customer`, `dim_warehouse`, `dim_carrier`

**Modeling principles applied**
- All relationships **one-to-many, single-direction** (dimension filters fact) — no bidirectional filtering
- Conformed dimensions shared across both fact tables
- `dim_date` marked as the date table (physical table, required for Direct Lake time intelligence)
- Raw numeric columns hidden so report authors use governed measures, not implicit sums

---

## DAX highlights

- **Reusable DAX user-defined function** to standardize growth logic across every variance KPI:
  ```dax
  FUNCTION Supply.GrowthPct = ( currentVal : NUMERIC, priorVal : NUMERIC ) =>
      DIVIDE ( currentVal - priorVal, priorVal )
  ```
  `Revenue YoY %` and `Revenue MoM %` both call it — one definition, one source of truth.
- **Time intelligence** — `SAMEPERIODLASTYEAR`, `DATEADD`, `TOTALYTD` off the marked date table.
- **Blank-safe division** (`DIVIDE`) and measures that reference measures, not columns.

---

## BI features used

Direct Lake semantic model · reusable DAX UDF · visual calculations (running total, moving average) · field parameters (metric/dimension switching) · slicers · drill-through · a shared report theme · Row-Level Security.

---

## Version control

Project stored as **Power BI Project (PBIP)** format — the report (PBIR) and semantic model (**TMDL**) serialize to readable text files, so changes are reviewable in Git. 
In an enterprise Fabric tenant, the next step is native **Git integration** on the workspace plus **deployment pipelines** for dev/test/prod.
---

## Tech stack

Microsoft Fabric · OneLake · Lakehouse · Direct Lake · Dataflow Gen2 · Power BI · DAX · Power Query (M)

---

fabric-supply-chain-analytics/
├── README.md                     # this file
├── ENGINEERING_NOTES.md          # modeling decisions + STAR stories
├── sales_dashboard.png           # Sales dashboard screenshot
├── Inventory_dashboard.png       # Inventory dashboard screenshot
├── data_model_star_schema.png    # data model diagram
└── <project files>               # PBIP / TMDL semantic model + report

*Personal learning project. Synthetic data only.*
