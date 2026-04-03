# Tableau to Power BI Migration

## What maps cleanly
Workbook → .pbix | Worksheet → Visual | Dashboard → Report page
Published Data Source → Shared semantic model | Extract → Import mode
Live connection → DirectQuery | Quick filters → Slicers
Highlight actions → Cross-highlight | Relative date filters → Relative date slicers

## What requires work
- Dual axis → Combo chart (limited mark mixing)
- Reference lines → Analytics pane
- Parameters → What-If (numeric) or disconnected slicer tables (text)
- LOD expressions → CALCULATE/ALLEXCEPT patterns
- Table calculations → WINDOW/RANKX functions

## Migration blockers (no Power BI equivalent)
- Page shelf animation
- Set actions (complex DAX workarounds needed)
- Marks card flexibility (drag-drop encoding)
- Hierarchical layout containers
- Custom shapes on marks
- Multi-layer maps (needs ArcGIS/custom visuals)
- Parameter actions

## Critical architectural shift
Tableau uses flat, denormalized datasets. Power BI performs best with star schemas.
50-column denormalized table → decompose into fact + dimension tables.

- Custom SQL → Power Query M with native query folding
- Data blending → composite models or proper star schema
- Power BI has no per-visual blending — data must be modeled with relationships
- Power BI supports only ONE active relationship between two tables
- Tableau's join culling has no Power BI equivalent
- Microsoft recommends "Semantic Layer-First": build centralized models, then reports

## Key DAX mappings

LOD expressions:
- `{FIXED [Region] : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), ALLEXCEPT(Sales, Sales[Region]))`
- `{FIXED : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), ALL(Sales))`
- `{INCLUDE [Product] : SUM([Sales])}` + AVG → `AVERAGEX(SUMMARIZE(...), [ProdSales])`
- `{EXCLUDE [Cat] : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), REMOVEFILTERS(Sales[Cat]))`

Table calculations:
- RUNNING_SUM → CALCULATE with FILTER/ALLSELECTED date range
- WINDOW_AVG → AVERAGEX with DATESINPERIOD
- RANK → RANKX(ALLSELECTED(...))
- LOOKUP → CALCULATE with OFFSET (2023+ DAX)
- Percent of total: `SUM / TOTAL(SUM)` → `DIVIDE(SUM, CALCULATE(SUM, ALL))`

Common function gotchas:
- DATEDIFF: Tableau interval is FIRST arg, DAX interval is LAST arg
- CONTAINS: Tableau = string function, DAX CONTAINS = table function → use CONTAINSSTRING
- REPLACE: Tableau = pattern-based, DAX = positional → use SUBSTITUTE
- AVG → AVERAGE, COUNTD → DISTINCTCOUNT, ATTR → SELECTEDVALUE
- No DAX: SPLIT, REGEXP_*, PROPER → need Power Query

## Complexity scoring

| Factor | Weight | Low (1) | Medium (2) | High (3) |
|---|---|---|---|---|
| Data sources | 3x | 1, extract | 2-3, mixed | 4+, cross-DB |
| Calculated fields | 2x | <10 simple | 10-30 nested | 30+ complex |
| LOD expressions | 4x | 0 | 1-5 FIXED | 5+ or nested |
| Table calculations | 3x | 0 | 1-5 default | 5+ custom addressing |
| Custom SQL | 2x | None | 1-2 simple | 3+ complex |
| Parameters | 2x | 0 | 1-3 numeric | 4+ or param actions |
| Dashboard actions | 3x | 0-2 filter | 3-5 mixed | 6+ or set/URL |
| Sets/set actions | 4x | None | 1-2 static | Dynamic + set actions |
| Data blending | 3x | None | 1 blend | Multiple blends |

Score: ≤30 = Low (1-2 days) | 31-60 = Medium (3-5 days) | 61+ = High (5-15+ days)

## RLS translation
Tableau: user filters with USERNAME() as data source filters.
Power BI: DAX role definitions with USERPRINCIPALNAME(). Only applies to Viewer role.

## Viz type mapping
Native equivalents: bar, column, line, scatter, treemap, waterfall.
Manual setup needed: heat map, highlight tables (matrix + conditional formatting).
No native: Gantt, box plot, bullet, packed bubbles, histogram (need custom visuals).
Partial: dual axis → combo chart (limited).
