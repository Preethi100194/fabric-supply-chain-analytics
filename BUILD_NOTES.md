# Build Notes & Design Decisions

## 1. Semantic modeling — design decisions

The model is the foundation; every dashboard is a lens on it. Key choices:

- **Star schema, not snowflake.** Two fact tables (`fact_orders`, `fact_inventory`) surrounded by conformed dimensions. Fewer joins, faster scans, simpler DAX.
- **One-to-many, single-direction relationships** (dimension → fact). Bidirectional filtering is avoided: it creates ambiguous filter paths, slows queries, and can silently weaken Row-Level Security.
- **Conformed dimensions.** `dim_date`, `dim_product`, and `dim_warehouse` are shared by both fact tables, so a single slicer filters orders and inventory consistently.
- **Grain is explicit.** `fact_orders` is one row per order; `fact_inventory` is one row per product/warehouse/month (a snapshot). Grain decides what an aggregation means and whether anything double-counts.
- **Measures reference measures, not columns.** e.g. `Avg Order Value = DIVIDE([Total Revenue], [Order Count])` — one definition to maintain, consistent filter-context behavior.
- **`DIVIDE`, never `/`.** Blank-safe against divide-by-zero.
- **Raw numeric columns hidden.** `Revenue`, `Cost`, `Quantity` are hidden so report authors use governed measures instead of dragging columns and creating rogue implicit sums.
- **Standardized logic via a DAX UDF.** `Supply.GrowthPct(current, prior)` defines period-over-period growth once; `Revenue YoY %` and `Revenue MoM %` both call it. One definition = one source of truth, version-controlled in Git.
- **Additivity is tracked per measure.** Revenue/COGS/Volume are fully additive. Inventory on-hand is **semi-additive** (sum across product and warehouse, but *not* across time — use a closing-balance/`LASTDATE` pattern, never `SUM` over dates). Knowing which is which is core modeling literacy.

---

## 2. Platform gotchas & resolutions

Real issues met during the build, with the *why* — useful reference for anyone working in Fabric / Direct Lake.

**Model-edit mode vs report connection.**
Opening a published semantic model for live editing gives Table / Model / DAX-query views only — no Report view, and no *File → Save as*. A report *live connection* is the opposite: it has Report view but can't change model structure. Resolution: use the right session for the task — model edits (relationships, measures, marking the date table) in the edit session; visuals in a report connection. Two artifacts by design: the shared governed model, and reports that consume it.

**Direct Lake requires a *physical* date table.**
Direct Lake can't materialize DAX calculated tables, so `CALENDAR()` / `CALENDARAUTO()` don't work. Resolution: the date dimension must be a physical table in the Lakehouse/Warehouse, then **Mark as date table** on its date column — which is what makes `SAMEPERIODLASTYEAR`, `TOTALYTD`, etc. resolve.

**"Table cannot be shown because it is not in import mode."**
Expected in a Direct Lake model — data lives in OneLake, not imported locally, so Table view can't page rows. Not an error.

**YoY % that looks wildly wrong (e.g. +47%) with a flat trend.**
A period-over-period measure evaluated with *all years* in context compares the full range against a shifted range that includes an empty prior year — a meaningless ratio. Resolution: **a YoY % is only valid inside a single-period filter context** — add a Year slicer (or evaluate per period). The measure was correct; the context was wrong. Classic time-intelligence trap.

**Filled map disabled.**
`FilledMapVisualNotEnabled` is a tenant/admin setting, not a data problem. Resolution: enable Map/Filled-map visuals — but for only 4 regions a sorted **bar chart** communicates ranking and magnitude better than a choropleth anyway. (Choose the visual that answers the question, not the flashiest one.)

**DAX query view "end of input" / syntax errors.**
DAX query view runs *queries* — every statement must end in `EVALUATE` (optionally with a `DEFINE` block above). *Measures* are `Name = expression` and belong in the formula bar, not the query editor. Different surfaces, different grammar.

**A UDF must be saved to the model before measures can call it.**
Defining `FUNCTION …` inside a query only scopes it to that query. Resolution: use **Update model: Functions** so it becomes a model object; then measures can reference it.

**Git integration is governed at the tenant level.**
Azure DevOps and GitHub have *separate* tenant switches; GitHub's is often off by default, which greys out the tile. Also: on an auto-created `*.onmicrosoft.com` trial tenant, creating an Azure DevOps org requires an Azure subscription. Resolution for a portfolio: version-control the project as **PBIP/TMDL** in a plain GitHub repo (no capacity or subscription dependency, and it outlives the trial).

**Feature availability is gated by Desktop version.**
DAX UDFs need the June 2026+ build; modern visual defaults + Theme pane need August 2026. Resolution: install from the Microsoft Store (auto-updates) and confirm via **Help → About**.

---

## 3. Power BI / Fabric 2026 feature reference (Jan–Aug)

For context on what "current" means in 2026. Items marked ★ are used in this project.

| Month | Feature | Status |
|---|---|---|
| Jan | Modern visual tooltips (drill-actions footer) ★ | GA |
| Jan | Tables/matrices size columns to content by default | GA |
| Jan | PBIR/PBIP default report format ★ | Rollout |
| Feb | Input Slicer (renamed from Text Slicer) | GA |
| Feb | `TABLEOF`, `NAMEOF` DAX functions | GA |
| Mar | **Direct Lake in OneLake** ★ | GA |
| Mar | Translytical task flows (write-back) | GA |
| May | **Visual calculations** ★ | GA |
| May | **Custom totals** ★ | GA |
| Jun | **DAX user-defined functions** ★ | GA |
| Jun | Shape map | GA |
| Jun | Date-picker slicer | Preview |
| Jul | Conditional formatting on lines/legends ★ | GA |
| Jul | Measure descriptions via `///` comments ★ | — |
| Aug | Modern visual defaults + Theme pane ★ | GA |
| Aug | Outer padding controls; granular refresh controls | GA |

**Cross-cutting themes:** Direct Lake maturing into the default modeling path; PBIR enabling real Git-based ALM; Copilot / AI woven through authoring and consumption (e.g. Fabric Data Agents for natural-language Q&A over a governed model).

---

*Personal learning project. Synthetic data only. Notes kept for reference and interview prep.*
