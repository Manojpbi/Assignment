# Agency Distribution Dashboard

This repository contains the PBIP report and semantic model for agency sales, targets, and policy persistency.

## Data model decisions

- The model is a star schema with `dim_date`, `dim_agent`, and `dim_product` as dimensions and `fact_sales`, `fact_target`, and `fact_persistency` as facts.
- `fact_sales` is the central sales fact. Dimension-to-fact relationships use single-direction filtering so unrelated facts do not cross-filter each other accidentally.
- `fact_target` arrives at month grain with `year` and `month`. Power Query creates `target_date_key` as the first day of the month, which relates it to `dim_date[date_key]` and keeps standard time intelligence available.
- A synthetic technical key `-1` with the business-facing label `Unknown` is added to `dim_agent`. Any sales agent key not found in the master is mapped to that member, preserving revenue while making the data-quality issue visible. The key remains numeric because `fact_sales[agent_id]` is an integer foreign key.
- `fact_persistency[issue_date_key]` is the active date relationship because the default cohort analysis groups policies by issue year. The renewal due date relationship is inactive and is activated in `Persistency Rate` with `USERELATIONSHIP` while disabling the issue-date relationship for that calculation.
- The `Territory Manager` role filters `dim_agent[territory]` using normalized `USERPRINCIPALNAME()` and `USERNAME()` values. This supports territory codes, UPN-style identities, and Windows-style identities during Desktop testing. A manager-to-territory bridge is still recommended when one manager owns multiple territories.

## DAX approach for the hardest measure

The hardest measure is `Active Policy Count` because policy status is a snapshot, not an additive amount. A policy can be active on one date and lapsed on a later date, so summing premium or summing daily counts would double-count the same policy across time.

The measure uses `LASTNONBLANKVALUE(dim_date[date], ...)` to evaluate the latest available snapshot in the current date context. Within that snapshot it uses `DISTINCTCOUNT(fact_persistency[policy_id])`, disables the issue-date relationship with `CROSSFILTER(..., NONE)`, and counts policies issued on or before the snapshot date whose status is not lapsed. This preserves semi-additive behavior while still respecting agent, product, territory, and other report filters.

The persistency rate uses the renewal due date rather than issue date for the denominator. `USERELATIONSHIP(fact_persistency[renewal_due_date_key], dim_date[date_key])` activates the inactive due-date relationship, while the issue-date relationship is disabled for the calculation. Only `renewed`, `lapsed`, and `grace` records are due; `active` records are excluded. `DIVIDE` prevents divide-by-zero errors. Other measures use `DATEADD` for MoM growth, `DATESINPERIOD` for rolling three months, and `ALLSELECTED` plus `RANKX` for rank within the visible agent population.

## Design and Figma handoff

The Page 1 wireframe was created before report construction in [page1_wireframe.html](mockups/page1_wireframe.html). It is a low-fidelity design artifact that can be recreated directly in Figma as a frame with four zones: Executive Summary header, left filter rail, KPI strip, and chart area. The wireframe keeps the filter rail persistent on the left for desktop scanning and moves it above the content on mobile.

The design direction is intentionally operational rather than decorative: restrained navy text, pale blue-gray surfaces, thin borders, compact KPI tiles, and strong alignment. The KPI strip is placed before charts because agency leaders need an immediate YTD and target read before investigating trend or agent detail. The trend and Top 10 charts are separate boxes so actual-versus-target movement and agent comparison remain visually distinct.

Formatting conventions are consistent across the report: currency measures use `$#,##0`, rates use `0.0%`, counts use `#,##0`, and month names are sorted by month number. Positive MoM performance uses green, negative performance uses red, and unavailable values use a neutral gray. Long agent and product names truncate safely in the HTML card. The mobile layout uses stacked sections and a two-column KPI grid so text and controls remain readable on phones.

## Wireframe

The pre-build Page 1 layout is in `mockups/page1_wireframe.html`. It separates the header, KPI strip, chart area, and filter area, with a responsive two-column KPI strip for phone viewing.

## Improvements with more time

- Add a maintained manager-to-territory bridge for multi-territory RLS.
- Add data-quality KPI cards for orphan agent and product keys.
- Add a dedicated cohort snapshot fact or pre-aggregated monthly snapshot table so the semi-additive active-policy measure remains fast at scale.
- Add automated refresh and model validation to CI once the repository is initialized and source paths are parameterized.
- Add performance analyzer checks for the cohort matrix and Deneb visual, especially under territory RLS.
- Add a formal Figma component library and design tokens for page headers, slicers, KPI cards, chart containers, and mobile breakpoints.
- Add a dedicated data-quality page showing orphan agent keys, unmatched products, missing targets, and late or invalid renewal dates.

## Custom visual prerequisite

Page 2 uses the Deneb custom visual. Deneb must be installed in Power BI Desktop or imported into the report before opening this page: select **Visualizations > Get more visuals**, search for **Deneb**, select **Add**, then reopen or refresh the PBIP. The PBIP definition stores the Deneb visual and Vega-Lite specification, but it cannot package an AppSource custom visual binary by itself.

Page 3 uses the HTML Content custom visual and the dynamic `Agent Profile HTML` measure. Install **HTML Content** from **Visualizations > Get more visuals**, then reopen the PBIP. Page 3 also includes an Agent slicer as a reliable direct-selection fallback. For drillthrough, add `dim_agent[agent_name]` to the Page 3 Drillthrough well in Power BI Desktop; then selecting an agent on Page 2 and choosing Drillthrough > Agent Profile Card carries the agent filter to the HTML card.

Page 4 uses `fact_persistency[issue_year]` as the cohort row, `fact_persistency[renewal_year]` as the renewal-period column, and `Cohort Renewal Rate` as the value. The rate uses due-policy logic and the renewal-date relationship, while `Active Policy Count` remains a semi-additive snapshot measure.