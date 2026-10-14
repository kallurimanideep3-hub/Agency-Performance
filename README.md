# Life Insurance Agency Dashboard (Power BI, PBIP)

A four-page Power BI report built on the supplied CSV files: Executive Summary, Agent Performance Grid (Deneb), Agent Profile Card (HTML), and Persistency Cohort. Saved as a PBIP project; one row-level security role, "Territory Manager".

## Repository layout

```
/mockups/page1_wireframe.html   Page 1 wireframe, drawn before building (also page 2-4 mock-ups)
/deneb/agent_performance_grid.vl.json   Vega-Lite spec used on Page 2
/AgencyPerformance.SemanticModel/          model (TMDL)
/AgencyPerformance.Report/                 report definition
README.md
```

Open the `.pbip` file in Power BI Desktop. The CSV folder is the `DataFolder` parameter in Power Query; point it at the folder holding the supplied files and refresh.

## Pages

| Page | What it shows | Visuals |
|---|---|---|
| 1 Executive Summary | APE YTD, APE vs Target %, Active Agents, Persistency Rate; monthly APE with target line; Top 10 agents; Region / Channel / Year slicers | native; mobile layout included |
| 2 Agent Performance Grid | one row per agent, bar = APE as % of target, tick = 100%, colour = above / on track / below | Deneb (Vega-Lite) |
| 3 Agent Profile Card | name, territory, YTD APE and rank, MoM arrow with 12-month sparkline, top product, persistency | HTML Content Secure, driven by DAX |
| 4 Persistency Cohort | renewal rate by issue-year cohort, cohort detail, Active Policy Count by year | matrix, table, column chart |

## Data model decisions

**Star schema, fact_sales at the centre.** `fact_sales`, `fact_persistency` and `fact_target` are facts; `dim_date`, `dim_agent` and `dim_product` are shared dimensions. Every relationship is many-to-one with single-direction filtering (dimension filters fact). I did not use bi-directional filtering anywhere; it is not needed and it makes ambiguity easy to introduce.

**Things the data brief did not spell out, and what I did:**

1. **`fact_target` is not in the brief's data list.** It is in the supplied files, so I used it. Targets are at agent and month grain. `fact_target` has no date key, so I added a small `dim_month` bridge (one row per month start) and relate `fact_target[month_start_date]` to it. `APE Target` uses `TREATAS` from the date filter's months onto `dim_month`. This keeps the date dimension at day grain without forcing a day-level relationship onto monthly targets.
2. **Orphan agent_ids in `fact_sales`** (sales rows whose agent is not in `dim_agent`). Dropping them would silently lose revenue; leaving them would produce blank agents. I remapped them to an explicit "Unmapped" agent (`agent_id = -1`) so total APE still reconciles, and added `dim_agent[is_mapped_agent]`. The Unmapped agent is excluded from the Top 10, ranks and target comparison (it has no target). Page 1 carries a footnote with its share of APE (about 5.1%), and the `Unmapped Agent APE` and `Unmapped Agent APE %` measures are in the Data Quality folder.
3. **Two date keys in `fact_persistency`.** There is one active relationship to `dim_date` on `renewal_due_date_key` and one inactive relationship on `issue_date_key`. Persistency is a renewal event, so renewal due date is the default. `Persistency Rate (by Issue Date)` activates the other with `USERELATIONSHIP` and is hidden from the report view; the cohort page uses `fact_persistency[issue_year]` directly, which needs no date relationship.
4. **Channel.** `channel` lives on `dim_agent` (an agent's channel), so the Channel slicer filters through the agent relationship.

**Row-level security.** Role "Territory Manager" filters `dim_agent` by the territory mapped to `USERPRINCIPALNAME()` in a small `security_user_territory` table. Because `dim_agent` filters all three facts, one rule covers sales, targets and persistency. If a login is not in the mapping table the role returns no rows (it fails closed). To demonstrate: Modeling > View as > Territory Manager, with a user from `security_user_territory`.

## DAX: the hardest measure, Active Policy Count

The brief says this is not a simple SUM. Active policies are a point-in-time quantity: a policy is active at date D if it was issued on or before D and had not lapsed by D. Summing monthly values double counts, so the measure is a snapshot at the last date in the filter context:

```DAX
Active Policy Count =
VAR _asOf = MAX(dim_date[date])
VAR _k    = YEAR(_asOf) * 10000 + MONTH(_asOf) * 100 + DAY(_asOf)
VAR _issued =
    CALCULATE(COUNTROWS(fact_persistency),
        REMOVEFILTERS(dim_date),
        fact_persistency[issue_date_key] <= _k)
VAR _lapsed =
    CALCULATE(COUNTROWS(fact_persistency),
        REMOVEFILTERS(dim_date),
        fact_persistency[renewal_status] = "lapsed",
        fact_persistency[renewal_due_date_key] <= _k)
RETURN _issued - _lapsed
```

Why it is built this way:

- `REMOVEFILTERS(dim_date)` and a comparison on the key columns turn the date filter into "up to and including the as-of date", so the result is cumulative, not the slice for one year or month.
- The as-of date is `MAX(dim_date[date])`, so on a year axis each year shows the position at the end of that year (2023: 5,034; 2024: 8,349; 2025: 7,242). A year-total row would not be the sum of the years and the page labels it as a snapshot.
- Policies in grace count as active; only "lapsed" removes a policy, and only once its due date has passed.
- On the cohort table (by issue year, no date filter) it gives policies issued less those lapsed by the latest date in the data: 3,795 for 2023 and 3,447 for 2024, 7,242 in total. This ties to the 2025 snapshot.

Other measures worth explaining:

- **APE vs Target %**: actual APE for mapped agents divided by `APE Target` for the same agents and months. Blank when no target exists, so a month without a target is not shown as 0%.
- **Agent Rank by APE**: `RANKX` over `ALLSELECTED(dim_agent)` so the rank responds to slicers but not to the row being ranked. The Page 3 version (`Card Rank`) uses `ALL(dim_agent)` instead, because a drill-through filter on one agent would otherwise make every agent rank 1.
- **Persistency Rate**: renewed divided by renewed, lapsed and grace policies. Policies not yet due are excluded from the denominator.

All required measures are in `_Measures`, grouped in display folders (0 Base, 1 Required, 2 Supporting, 3 Data Quality, 4 Cohort, 5 Visual feeds, 6 Agent Card).

## Visual notes and known behaviour

- **Deneb does not receive Decimal measures.** Currency measures arrive as undefined, which shows as `$NaN` in tooltips. The grid is fed by two floating-point "chart feed" measures (`APE (chart feed)`, `APE Target (chart feed)`); the achievement percentage is already a double.
- **Cross-filter vs. cross-page.** Power BI cannot cross-filter between pages, so "cross-filtering to Page 3" is implemented as drill-through: left-click on the grid cross-filters within Page 2; right-click an agent > Drill through > Agent Profile Card opens that agent on Page 3 (`dim_agent[agent_label]` is the drill-through field). The Deneb context menu setting must be on.
- **Achievement above 100%.** Targets in the supplied data are well below actual sales for most agents (the median agent in December 2025 is about 265% of target). The grid caps bars at 500% for readability and the label always shows the real percentage. I treated this as a property of the data, not a calculation error. I recomputed it from the CSV files with pandas: target rows total $12,856,918 against $39,278,766 of APE, and the actual-to-target ratio is about 2.6 to 2.9 at monthly, quarterly and annual grain, so it is not a unit or period mismatch (for example, targets entered as quarterly figures). The measure divides actual by target exactly as the data gives them. If the business confirms the targets are understated, only the source file needs correcting; no DAX changes.
- **Status thresholds**: above target at 100% or more, on track from 90% to under 100%, below target under 90%. The thresholds are in the `Target Status` measure and mirrored in the Deneb spec.
- **Page 3 card** is a DAX-generated HTML string rendered by the certified HTML Content Secure visual. It uses its own as-of date logic so page-level date filters do not change it.

## What I would improve with more time

1. Replace the hard-coded status thresholds (100% / 90%) with a small parameter table so a manager can tune them.
2. Add an incremental or dynamic date range so the Page 3 "latest month" does not depend on the maximum date in the data.
3. Real time-based cohorts: the supplied persistency data covers one renewal step per issue year, so the cohort view is a two-by-two grid. With more history I would add "years since issue" columns and a retention curve.
4. Calculation group for time intelligence (YTD, rolling, MoM) instead of separate measures.
5. Automated model checks (Tabular Editor best-practice rules, DAX unit tests against a pandas reference) wired into CI, plus a deployment pipeline.
6. A proper test of RLS with several real territory managers, and an RLS-aware Unmapped agent rule so territory managers do not see the Unmapped total.
7. Report-page tooltips on the Top 10 and trend visuals, and accessibility pass (alt text, tab order, contrast).
