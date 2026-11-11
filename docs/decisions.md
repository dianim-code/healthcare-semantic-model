# Decision log

Architecture decisions in a light ADR format: **context → decision → trade-offs**. ADR numbers are
shared with the `healthcare-lakehouse` and `deneb-gallery` repositories. Decisions about the data
itself (sources, contract design, licensing) live in `healthcare-lakehouse/docs/decisions.md`.

---

## ADR-001 · Consume the gold contract as a vendored copy

*2026-11-10 · Accepted*

**Decision.** The model is built on the gold contract owned by `healthcare-lakehouse`. This repo
keeps a read-only copy in `contract/` and `sample-data/gold/` (see `contract/VENDORED.md` for the
version and checksums). Power Query reads the CSVs through a `SampleDataFolder` parameter.
Connecting to a Databricks SQL warehouse is the documented alternative.

**Trade-offs.** No dependency on the lakehouse repo or on a workspace. A contract change has to be
re-vendored from the lakehouse.

## ADR-004 · Power BI: PBIP with TMDL and PBIR

*2026-11-10 · Accepted*

**Context.** The semantic model has to be reviewable as text, and the report pages are built in
Desktop.

**Decision.** Use the PBIP format with a TMDL semantic model. The report uses **PBIR**, which is
enabled. Evidence: a PBIP saved on this machine by Power BI Desktop **2.159.1231.0** has
`definition.pbir` version `4.0` (schema `definitionProperties/2.0.0`), a `definition/pages/`
folder, and `compatibilityLevel: 1606` in `database.tmdl`. Tabular Editor **2.28.9557** is
installed locally. CI runs the Best Practice Analyzer with the Tabular Editor 2 CLI on a Windows
runner.

**Trade-offs.** PBIR and TMDL change between Desktop releases, so syntax is checked against
Microsoft Learn rather than copied from older projects.

## ADR-005 · Language and licensing

*2026-11-10 · Accepted*

**Decision.** Object names, measures, descriptions and docs are in English, and the code is under the
**MIT** license. Data values stay in Italian as published (see ADR-005 in `healthcare-lakehouse`).
The sample data keeps its own licenses, with attribution in `contract/VENDORED.md`.

---

## ADR-010 · TMDL project layout and schema versions

*2026-11-10 · Accepted*

**Decision.** `HospitalBeds.pbip` points to a PBIR report shell (`HospitalBeds.Report`, one empty page,
`definition.pbir` 4.0 by path) and a TMDL semantic model (`HospitalBeds.SemanticModel`:
`definition.pbism` 4.2, `definition/` with `database.tmdl` at `compatibilityLevel: 1606`, `model.tmdl`,
`expressions.tmdl`, `relationships.tmdl`, `cultures/`, `roles/`, `tables/`). Layout and schema versions
were copied from a project saved by Desktop 2.159.1231.0; names, content and ids were not. All files are
UTF-8 without BOM, TMDL uses tabs. Report visuals are built in Desktop, not by hand.
Constructs not present in that reference (descriptions, display folders, `sortByColumn`, `isHidden`,
roles, hierarchies) follow the TMDL reference:
<https://learn.microsoft.com/analysis-services/tmdl/tmdl-overview> and the project docs
<https://learn.microsoft.com/power-bi/developer/projects/projects-overview>,
<https://learn.microsoft.com/power-bi/developer/projects/projects-dataset>.
Descriptions use the `///` block above an object; roles live in `roles/<name>.tmdl` and are listed
with `ref role` in `model.tmdl`.

**Trade-offs.** Desktop rewrites TMDL on save (property order, omitted defaults such as the explicit
`fromCardinality: many` and `crossFilteringBehavior: oneDirection`), so the first save in Desktop
produces a diff. The semantics do not change.

## ADR-011 · Source queries: one parameter, explicit types, codes as text

*2026-11-10 · Accepted*

**Decision.** A parameter query `SampleDataFolder` (`meta [IsParameterQuery=true, Type="Text",
IsParameterQueryRequired=true]`) holds the full path of `sample-data\gold`. Its committed default is the
placeholder `<SET ME: full path of the sample-data\gold folder>`, never a real path. Each table reads
`<SampleDataFolder>\<table>.csv` with `Csv.Document(..., Encoding = 65001)`, `Table.PromoteHeaders`
and one explicit `Table.TransformColumnTypes` with culture `en-US` that covers every contract column.
Codes (`region_code`, `discipline_code`, `municipality_code`, `facility_type_code`, ...) are `type text`
in M and `string` in the model, so leading zeros and `2.1` survive. Nullable text columns
(`site_subcode`, `address`) map empty strings to null. `sourceQueryCulture` is `en-US`.
Contract type mapping: `int` and `bigint` to `Int64.Type`/`int64`, `date` to `type date`/`dateTime` with
`UnderlyingDateTimeDataType = Date`, `boolean` to `type logical`/`boolean`.

**Trade-offs.** The parameter is not set by CI (the BPA reads metadata only), so the placeholder fails
a refresh until the user sets it. That is intended. A second source, the lakehouse's Databricks SQL
warehouse, was added later behind the same table queries (ADR-022).

## ADR-012 · Pure star schema, single direction, no facility-to-region link

*2026-11-10 · Accepted*

**Decision.** Six many-to-one, single-direction, active relationships taken from the contract
`references`: `fact_hospital_beds` to `dim_year`, `dim_facility`, `dim_discipline`, `dim_region`;
`fact_population` to `dim_year`, `dim_region`. `dim_facility.region_code` is also a contract
reference, but it is **not** a relationship: the facts already carry the denormalised `region_code`,
and a second path from `dim_region` to `fact_hospital_beds` (directly and through `dim_facility`) would
be ambiguous. Keys, foreign keys, sort columns and technical codes are hidden; every numeric column has
`summarizeBy: none`; `discourageImplicitMeasures` is on, so all aggregation goes through measures.
`region_name`, `discipline_name` and `facility_type_name` use `sortByColumn` (their codes are 1:1 with the
names in the sample data).

**Trade-offs.** `dim_facility[region_code]` filters facilities, not population or the regional fact
rows; this is why the RLS role (ADR-016) filters `dim_facility` as well.

## ADR-013 · Beds and population are stocks: evaluate at the latest year in context

*2026-11-10 · Accepted*

**Context.** Both beds and population are counted on 1 January. Summing three years of beds triples
the answer.

**Decision.** A visible measure `Reference Year` = `MAX(dim_year[year])`. Every stock measure
(`Beds *`, `Population`, `Facilities With Beds`) is `CALCULATE(<aggregate>, dim_year[year] = _year)`
where `_year` is captured in a variable first (a measure reference is not allowed inside a Boolean
filter argument, and the variable is evaluated in the outer context). With one year selected the
result is that year; with several, the latest; with none, the latest in the model. Per-capita measures
divide two such measures, so beds and population always come from the same year and region.
Prior-year measures shift `Reference Year` by one and return blank when that year is absent.

**Trade-offs.** "Latest year in context" is a convention, not an average; it is stated in every
measure description and `Reference Year` can be placed on a card to show it. A year with beds but no
population gives a blank ratio instead of a mismatched one.

## ADR-014 · Per-capita definitions

*2026-11-10 · Accepted*

**Decision.** Two ratios. `Beds per 1,000 Residents` uses all four regimes and includes day-care
places. `Ordinary Beds per 1,000 Residents` uses `beds_ordinary` only and is the one to compare with
international statistics (overnight inpatient beds). Paying beds (`beds_paying`) are **excluded** from
the ordinary ratio. Facility and discipline filters narrow the numerator only: the denominator is
the population of the selected regions, so "cardiology beds per 1,000 residents" is meaningful.
The rank measures use `ALLSELECTED(dim_region)` (rank among the regions left by slicers), dense
ranking, and show a value only with one region in context.

## ADR-015 · Best Practice Analyzer rules and CI

*2026-11-10 · Accepted*

**Decision.** `quality/bpa-rules.json` is a focused set of 15 rules written for this model (no
vendored Microsoft rules, so no third-party license text to carry). Severity 3 (error) fails CI:
missing table or measure description, numeric column that summarizes, `*_code` column that is not
text, visible foreign key, bidirectional or many-to-many relationship, implicit measures allowed.
Severity 2 rules are warnings: TE2 exits 0 for them. CI (`.github/workflows/semantic-model-ci.yml`)
runs on `windows-latest`, checks that PBIP files have no BOM and no local paths, downloads the
**pinned** Tabular Editor 2 **2.28.0** portable zip (SHA-256 verified) and runs
`TabularEditor.exe <definition folder> -A quality\bpa-rules.json`. TE2 2.28 loads a TMDL `definition`
folder directly (verified locally, including that a malformed TMDL file fails the load with exit
code 1). `TabularEditor.exe` is a GUI-subsystem binary: in PowerShell its output is piped to
`Out-Host` so the shell waits for it and `$LASTEXITCODE` is valid. TE2 parses TMDL and evaluates the
BPA rules; it does **not** validate DAX against an engine, so measure results are checked in Desktop.
Sources: <https://docs.tabulareditor.com>, <https://github.com/TabularEditor/TabularEditor/releases>.

**Trade-offs.** Tabular Editor 2 is no longer the main line (Tabular Editor 3 has a CLI too, but needs
a license); 2.28.0 is pinned and bumped by hand.

## ADR-016 · Row-level security: one static role, dynamic path documented

*2026-11-10 · Accepted*

**Decision.** Role `Region Piemonte` (`modelPermission: read`) with two table filters:
`dim_region[region_code] = "010"` and `dim_facility[region_code] = "010"`. The first reaches both fact
tables through the single-direction relationships; the second is needed because `dim_facility` has no
relationship to `dim_region`, so without it the facility list of all regions would stay visible.
Under the role, `Share of National Beds` and the rank measures only see the user's region (stated in
their descriptions). A dynamic version (user-to-region mapping table filtered with
`USERPRINCIPALNAME()`) is described in the README, not implemented, to keep the sample free of user data.

## ADR-017 · No calculation group for time intelligence

*2026-11-10 · Accepted*

**Decision.** No calculation group. The grain is the year, there is no date table, and only one shift
(prior year) is needed, so explicit `Prior Year`, `YoY Change` and `YoY %` measures are shorter and
easier to audit than an item with `SELECTEDMEASURE()`. A calculation group would also apply to ratios
and ranks, where "prior year" does not compose cleanly, and would need a `formatStringDefinition` per
item. Revisit if more shifts (3-year change, CAGR) are required.

## ADR-018 · Business names for the dimensions, contract names for the hidden facts

*2026-11-10 · Accepted (from a naming review of the model)*

**Context.** The model first used the contract names everywhere (`dim_region`, `region_name`). A report
author and a reader see those names in the field list, and the measure `Population` had the same name as
the column `fact_population[population]` (Tabular names are case-insensitive).

**Decision.**
- The four dimensions are renamed `Year`, `Region`, `Discipline` and `Facility`, and their columns get
  business names (`Region Code`, `Macro Area`, `Facility Type`, `Municipality`...). Each column keeps its
  contract name in `sourceColumn`, and the M queries still read `dim_*.csv`: the gold contract and the
  vendored files do not change.
- The two fact tables keep the contract names. All their columns are hidden, and business names such as
  `Beds Ordinary` would collide with the measures of the same name.
- `Population` becomes `Resident Population`, with `Resident Population Prior Year`, `... YoY Change` and
  `... YoY %`.
- The BPA rule on code columns matches both `... Code` and `..._code`.

**Trade-offs.** DAX now quotes the table names (`'Year'[Year]`, `'Region'[Region Code]`). The README maps every
model table to its contract table.

## ADR-019 · No date table: the time dimension is the year

*2026-11-10 · Accepted (from a best-practice review of the model)*

**Context.** Common Power BI best-practice rules expect a table marked as a date table (`dataCategory: Time`).

**Decision.** None is marked, on purpose. Both facts are at year grain and both are stocks on 1 January,
so a daily calendar would hold one used date per year. No measure uses DAX time-intelligence functions:
prior-year logic shifts `'Year'[Year]` (ADR-013). `Year` cannot be marked as a date table either, because
that needs a contiguous date column.

**Trade-offs.** Built-in date hierarchies and time-intelligence functions are not available. If a fact with
a finer grain arrives (admissions by date, for example), add a calendar then.

## ADR-020 · Deneb visuals: the spec owns title and colors

*2026-11-10 · Accepted (from a review of the report formatting)*

**Decision.** Deneb visuals take their title, subtitle and colors from the Vega-Lite spec of the
`deneb-gallery` template they come from. The Power BI container title stays off, so the title is not drawn
twice, and the hex colors in `visual.json` belong to the spec, not to formatting overrides. Deneb
(`deneb7E15AEF80B9E4D4F8E12924291ECE89A`) is the only custom visual. Whoever publishes the report must
confirm that it is allowed in their tenant.

**Trade-offs.** If the report gets a custom theme, align the template palette with it, or read the theme
colors in the spec with Deneb's `pbiColor()`.

## ADR-021 · Charts show the 20 ISTAT regions

*2026-11-11 · Accepted (contract 1.1.0)*

**Decision.** `Region[ISTAT Region]` (from `istat_region_name`) groups the autonomous provinces of Bolzano
and Trento into Trentino-Alto Adige. The region charts of the report use it, so they show the 20 regions
readers expect. The hidden measure `[Region Order]` (`MIN ( 'Region'[Sort Order] )`) gives their display
order: grouping by `Sort Order` as a column would split Trentino-Alto Adige into two rows. In the Deneb
visuals the two fields keep the names the specs use ("Region", "Sort Order") through `displayName`.

**Trade-offs.** `Region[Region]` and the RLS role still work on the 21 Ministry codes. The per-capita
measures sum beds and population before dividing, so they are correct at the ISTAT level too. The rank
measures need one `Region Code` in context, so they are blank for Trentino-Alto Adige.

## ADR-022 · Two sources behind one function: CSV by default, Databricks on demand

*2026-11-11 · Accepted*

**Context.** The model read only the committed CSVs, so the link to the lakehouse was documented but not
shown. The CSVs must stay the default: anyone who clones the repository has to be able to open and refresh
the project without a Databricks account.

**Decision.** A parameter `DataSource` (`CSV` or `Databricks`) and a function query `GoldTable(name)`. Each
table query calls `GoldTable("<contract table>")` and keeps its own `Table.TransformColumnTypes` step, so
the column names, types, relationships, measures and RLS are the same for both sources. The Databricks
branch uses `DatabricksMultiCloud.Catalogs` (Databricks connector, OAuth (OIDC) sign-in) with the host,
HTTP path, catalog and schema as parameters, and keeps rows from `ReportFromYear` (2021) onwards, on `year`
or, for facilities, `last_year`. The filter folds to the warehouse. Import mode, not DirectQuery: the data
changes once a year, and DirectQuery would wake the warehouse on every interaction.

**Trade-offs.** Workspace values must never be committed: they stay placeholders, and the pre-commit hook
and the CI check reject a real host, HTTP path or schema, or a source other than `CSV`. `facility_key` is an
ordinal within the lakehouse's gold window (contract rule C4), which starts in 2019, so its values differ
from the sample's; it is only used as a join key, so no result changes. `Facility[first_year]` can be 2019
or 2020 from Databricks. A switch between sources is a full refresh. A service refresh would need the
Databricks credentials set on the published semantic model; this project is not published.

---

## Resolved items

| Item | Resolution |
|---|---|
| Tabular Editor 2 version that reads a TMDL folder, and its BPA CLI syntax | 2.28.0 (build 2.28.9557) loads a `definition` folder and runs `-A <rules file>`; see ADR-015 |
| TMDL syntax for RLS roles, calculation groups and parameters at level 1606 | Roles and parameters implemented (ADR-016, ADR-011) per the TMDL reference; calculation group not used (ADR-017), so its syntax is not exercised |

## Verification log

| Date | Topic | Version / fact | Source |
|---|---|---|---|
| 2026-11-10 | Power BI Desktop | 2.159.1231.0, PBIR on (`definition.pbir` 4.0), compat level 1606 | local install and a locally saved PBIP |
| 2026-11-10 | Tabular Editor | 2.28.9557 | `TabularEditor.exe` file version |
| 2026-11-10 | Tabular Editor 2 | 2.28.0 loads TMDL folders; `-A <rules.json>` exits 1 on severity-3 violations, 0 on warnings | local run, see ADR-015 |
| 2026-11-10 | TE2 portable zip | 2.28.0 `TabularEditor.Portable.zip` SHA-256 `ba85f4653c850b06687e6e0b3c8af3d7c951921ec281deb5df615b2ce43d4ef2` | GitHub release asset digest, matches a local download |
| 2026-11-10 | Opened in Power BI Desktop | The PBIP opens; after setting `SampleDataFolder` the refresh loads 35,053 / 1,319 / 69 / 21 / 3 / 63 rows; codes keep leading zeros (`010`, `08`, `2.1`, 6-digit municipalities) | Desktop 2.159.1231.0, read-only DAX queries against the local Analysis Services instance of Desktop |
| 2026-11-10 | Measures on a live engine | All 21 measures evaluate. Every value matches an independent pandas computation on `sample-data/gold`: 2023 beds 212,768, 3.6064 and 3.2467 per 1,000, YoY -8,903 (-4.02%); with 2021 and 2022 selected the measures return 2022, never the sum; 2021 prior year is blank; regional shares and both dense ranks match for all 21 regions | same |
| 2026-11-10 | RLS role `Region Piemonte` | Under impersonation: 1 region, 100 facilities, none from other regions, 18,080 beds, population 4,251,351; `Share of National Beds` = 100% as documented in ADR-016 | same |
| 2026-11-11 | Databricks connector | `DatabricksMultiCloud.Catalogs(host, httpPath, options)`; authentication kinds Databricks Client Credentials, OAuth (OIDC), Personal Access Token; for warehouses on AWS with OAuth | [Power Query Databricks connector](https://learn.microsoft.com/power-query/connectors/databricks) |
| 2026-11-11 | Refresh from Databricks | Free Edition serverless SQL warehouse (2X-Small, auto-stop 10 min), OAuth sign-in in Desktop. Six queries from the Power BI client, `WHERE year >= 2021` / `last_year >= 2021` folded; rows 3 / 69 / 21 / 1,319 / 35,053 / 63, each under 0.7 s. Measures equal the CSV refresh: 2023 beds 212,768, 3.6064 and 3.2467 per 1,000, YoY -8,903 (-4.02%), 1,272 sites, 20 ISTAT regions, Piemonte 3.8421 | warehouse query history; DAX queries against the local Desktop instance |
