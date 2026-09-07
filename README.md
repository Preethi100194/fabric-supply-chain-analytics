# Supply-Chain Analytics on Microsoft Fabric

A personal, end-to-end analytics project built on **Microsoft Fabric** and **Power BI** to demonstrate semantic modeling, DAX, and modern BI delivery on a supply-chain dataset. 
Built on **synthetic data** generated for the project — no real company data is used.

> **Why this project:** I'm a Power BI developer with SAP BW and supply-chain experience (PL-300, DP-600, DP-800 certified).
> I built this on my own Fabric trial to get hands-on, end-to-end, with the full Fabric stack — from lakehouse to a governed semantic model to a report — the way it's done in production.

---

## What it does

Models a manufacturing supply chain (orders + monthly inventory snapshots) and reports on sales performance, with the semantic model designed to power a multi-dashboard suite (Sales first, with Inventory, Service Level, Open Orders, Demand Planning, Production, and Logistics on the same model).

**Sales dashboard KPIs:** Total Revenue, Volume, Order Count, Avg Order Value, Gross Margin %, and Revenue YoY / MoM / YTD.

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

*Personal learning project. Synthetic data only.*
