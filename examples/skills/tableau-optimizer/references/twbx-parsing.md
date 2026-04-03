# TWB/TWBX XML Parsing

## File structure
- .twbx is a standard ZIP. .twb XML at root level.
- Extracts in `Data/Datasources/` (.hyper or .tde)
- Images in `Images/`. Local data in `Data/`.

## Root element
`<workbook>` attributes: `source-build` (Tableau version), `source-platform` (win/mac),
`version` (schema version like '18.1').
Always check these first when parsing.

## Datasources
`<datasource>` attributes: `name` (internal ID like 'federated.0abc123'), `caption` (display name),
`inline='true'` (embedded vs published).

Connection types via `<connection class='...'>`:
excel-direct, sqlserver, postgres, redshift, bigquery, snowflake, mysql, dataengine (Hyper extracts).

## Extract vs live detection
- Extract: `<extract enabled='true'>` within `<datasource>`
- Inner `<connection class='dataengine'>` points to .hyper file
- No `<extract>` or `enabled='false'` = live connection
- Incremental refresh: `<refresh incremental-updates='true' increment-key='[FieldName]'>`
- Outer `<connection>` retains live connection info regardless

## Calculated fields
`<column>` elements with child `<calculation>` element.
Naming: `[Calculation_XXXXXXXXXXXXXXXXXXX]` (19-digit random).

LOD, regular, and table calcs share same XML structure.
Classify by parsing formula text:
- `{FIXED`, `{INCLUDE`, `{EXCLUDE` → LOD
- RUNNING_SUM, WINDOW_AVG, RANK, INDEX, LOOKUP → table calc
- Everything else → regular calc

Key attributes: `caption` (name), `datatype`, `role` (dimension/measure),
`type` (quantitative/nominal/ordinal).

## Custom SQL detection
`<relation type='text'>` — SQL in element text content.
Newer versions: `<_.fcp.ObjectModelEncapsulateLegacy.false...relation>`.
Custom SQL = known anti-pattern (wraps as subquery).

## Initial SQL
`one-time-sql` attribute on `<connection>`.

## Parameters
Special datasource: `name='Parameters'`, `hasconnection='false'`.
Each parameter: `<column>` with `param-domain-type` ('all', 'range', 'list').
`<range>` or `<members>` elements define allowable values.

## Data source filters vs worksheet filters
Location in XML tree is the only way to distinguish:
- Data source: `<filter>` directly under `<datasource>`
- Worksheet: `<worksheet>` → `<table>` → `<view>` → `<filter>`

## Worksheets
`<worksheet>` → `<table>` → `<view>`.
Fields on shelves in `<rows>` and `<cols>` text.
Notation: `[Datasource].[aggregation:Field:suffix]`
- 'none:Category:nk' = dimension (no agg, nominal key)
- 'sum:Sales:qk' = measure (SUM agg, quantitative key)

Mark type in `<mark class='...'>`.
Filters: `class='categorical'`, `class='quantitative'`, `class='relative-date'`.

## Dashboards
Nested `<zone>` elements. Match `name` to worksheet names.
Zone types: 'layout-basic' (containers), 'text', 'bitmap', 'web'.
Actions: `<action>` with types 'filter', 'highlight', 'url', 'sheet'.

## Two highest-value automated checks
1. **COUNTD-in-LOD detection** — single most impactful performance anti-pattern
2. **Custom SQL audit** — most common governance gap
