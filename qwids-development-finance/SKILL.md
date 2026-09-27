---
name: qwids-development-finance
description: Get defensible OECD development-finance figures (ODA, CRS, the DAC tables, amounts mobilised from the private sector, climate-related development finance) out of QWIDS 2.0, and read the answer correctly. Use when asked about aid flows, ODA volumes or ODA/GNI, donors or recipients of development finance, sector or purpose breakdowns, private finance mobilised, climate finance, policy markers such as gender or climate, or when a figure needs a citation and a data vintage.
---

# QWIDS 2.0: development-finance figures

QWIDS serves OECD DCD development-finance statistics. The hard part is not
fetching a number, it is fetching the *right* number and knowing what it does
not include. This skill covers both.

**Base URL.** Ask the user for theirs, or use `$QWIDS` if it is set. The
current development deployment is
`https://qwids-web.whitetree-87b148ac.francecentral.azurecontainerapps.io`.
Treat that as a default that will move, not as the address of record.

## Let QWIDS pick the table

There are thirteen queryable tables: DAC1, DAC2a, DAC2b, DAC3a, DAC4, DAC5,
DAC7b, CPA, GENDER, Mobilisation (`mobilised`), the two views of
climate-related development finance (`crdf_provider`, `crdf_recipient`) and
CRS. Do not choose one unless you have a reason. Use `auto`:

```
GET /v1/datasets/auto/data?donor=France&years=2022&group_by=recipient&measures=disbursement
```

QWIDS tries them coarsest first and answers from the first that can express
every coordinate you selected. `meta.dataset` says which answered.

A table is ruled out by what it cannot carry, not by a hand-written rule: name a
recipient and DAC1 and DAC5 drop out because they have no recipient dimension;
ask for a five-digit purpose code and every DAC table drops out. GENDER and the
climate tables do record purposes, but they answer only their own questions
(below), so a plain purpose question goes to CRS.

`GET /v1/resolve?...` returns the verdict per table under `routing.datasets[]`,
each with `blocking[]`, a `reason` and a `reason_code`, and costs no data read.
Use it when a user asks why they got a particular table.

Pin a table only when asked to: `dataset=crs`. `/v1/resolve?...&dataset=crs`
then reports `routing.forced: true`, and `routing.alternatives[]` lists the
tables that would also have answered, with how their figures differ.

## Three tables answer only their own question

- **GENDER** answers only when the gender marker is selected
  (`marker.gender=...`). See *Policy markers*.
- **Mobilisation**: `aid_type=mobilised`. The amounts are stored as published.
  When every provider asked for is one DAC1 publishes in every year asked,
  DAC1's row 2233 answers; otherwise (no provider, or a year DAC1 skips)
  Mobilisation answers the whole question. `mechanism=` narrows to an
  instrument: 5401 syndicated loans, 5403 shares in collective investment
  vehicles, 5406 guarantees, 5407 direct investment, 5409 credit lines, 5410
  simple co-financing.
- **Climate-related development finance**: `aid_type=climate`. A provider
  question goes to the provider view, a recipient question to the recipient
  view. **Never add the two**: they hold the same bilateral activities. The
  data are commitments only, so asking for disbursements is refused
  (`flow_basis`). The provider view adds each provider's imputed share of
  multilateral climate finance, so it is not the figure a CRS Rio-marker query
  (`marker.climate_mitigation=...`) returns. Say which one you used.

## Read the whole envelope, not just the number

Every `/data` response carries:

- `meta.vintage` and `meta.citation`. **Always quote the vintage.** Two requests
  with the same query and vintage return identical numbers; without it a figure
  is not reproducible.
- `meta.unit`. `"USD million"`, or `"Per cent"` for a ratio such as ODA as a
  share of GNI (DAC1 `aid_type=11002`). A ratio is one figure per provider and
  year: it has no `totals`, and a query that would add ratios together is
  refused (`ratio_summed`).
- `warnings[]`. These are not decoration. They say things like "subtotal rows
  were excluded so the breakdown sums to the true total", or that a figure may
  be understated. Pass them on.
- `notes[]`, each `{level, text, code, params}`: `note` is always true,
  `caution` means the query ran but not quite as asked (a default was applied,
  a parameter ignored), `problem` means the answer is empty or would be wrong.
  Branch on `code`; `text` is prose and changes.
- `totals`. The figure for the whole selection, consistent with the rows.
- `meta.cross_check`, on aggregate answers. `same_figure[]` lists other tables
  that publish this figure, with their value and whether it ties; `crs` gives
  the plain CRS query and what it differs by (`differs_by`, such as
  `net_of_receipts`); `drill` says whether the figure opens its CRS
  activities. Use it to offer "the other number" rather than re-querying.
- `recipe` (top level). The published source, the steps that reproduce the
  figure from it, and an `api_csv_url`.
- `meta.composition`, for a DAC table: the CRS conditions behind each row.
  `GET /v1/datasets/{id}/composition?code=206` states them and
  `/v1/datasets/{id}/composition/206/activities` lists the activities.

If `warnings[]` is non-empty, or a note is a `caution` or a `problem`, say what
it says. A number quoted without its caveat is the failure mode this platform is
built to avoid.

## What it will refuse, and why that is correct

QWIDS refuses combinations that produce a wrong number rather than warning about
them: an aggregate together with its own members, Part I and Part II of the DAC
List in one figure, a ratio summed at all, memo items mixed with non-memo ones,
a parent code with a code it already contains.

A refusal is an RFC 7807 `application/problem+json` body with a machine `code`
and its `params`: `aggregate_mixed_with_member`, `ratio_summed`, `flow_basis`,
`grant_equivalent_before_2018`, and `unknown_code`, whose `params.suggestions`
holds the closest valid codes.

If you get a refusal, do not work around it by pinning a table or by adding
`include_aggregates=true`. Reformulate the question.

`include_aggregates=true` exists to read a table's structure. Never sum what it
returns.

## Measures and prices

- **Commitment** is a firm written obligation; **disbursement** is the actual
  transfer. They are different measures and are never mixed in one column.
  Disbursements can lag commitments by years.
- **Grant equivalent** is reported from 2018 onwards. Earlier years are
  structurally empty, not missing, and a question only about years before 2018
  is refused (`grant_equivalent_before_2018`).
- `flow=oda` in the CRS is grants (11), concessional loans (13) and equity (19),
  **gross**. DAC2a's default, "ODA: Total Net", is net of loan repayments and
  recoveries, so the two differ; `meta.cross_check.crs` says by what.
- `price_base=constant` gives USD millions in 2024 prices, for comparing volumes
  across time. `current` is the prices of each year. Never mix them in one
  series, and say which you used.
- Income groups and LDC status are read against today's DAC List for every
  year by default; `dac_list=by_year` uses the list in force each year.

## Policy markers

Markers use the DAC three-point scale: 2 principal, 1 significant, 0 screened
but not targeted. **Empty means not screened, which is not the same as 0.**
Always report a marker total together with how much was screened.

The published gender-equality indicator is the share of screened aid:
`(principal + significant) / (not targeted + significant + principal)`.
Activities never screened are outside the base. A gender-marker question is
answered by GENDER, the published table, not by CRS.

```
GET /v1/datasets/auto/data?marker.gender=not_targeted,significant,principal&group_by=donor,gender_marker&years=2022
```

## Free-text search

`q=` matches a literal, case-insensitive substring across an activity's title,
short description and long description, joined with a space, so a phrase can
span two fields. It is not a pattern: `%` and `_` match themselves. It works on
CRS and on both climate tables.

The count is **activities that mention the phrase**, not an amount. Projects
described in other words will not be found, and descriptions are written by each
provider in its own style. Treat it as a way in to the activities, never as a
measured total.

## Exports

`GET /v1/datasets/{id}/export?levels=aggregate,activity&format=xlsx` builds a
workbook with its own parameters, citation and notices.

The parameter is `levels` (plural, a comma list). `level` is a valid parameter
on `/data` and has no effect here; the export says so if you send it, but do not
rely on being told.

## Whole tables

For "the whole table" or "all of CRS for 2023", do not page through `/data` or
build an export: `/export` keeps only the first 500,000 activity rows (75,000
in xlsx) and says so in a notice.
`GET /v1/bulk` lists every table of the vintage as a zipped CSV and a Parquet
file, CRS one pair per year, each with its `rows`, `bytes` and an `href`. The
`href` redirects to a signed storage link valid for an hour, and each zip
carries a README with the citation. Every code comes with its English name.

## SDMX

`/sdmx/data/{agency},{dataflow},{version}/{key}` serves CRS and the ten DAC
dataflows (every table but the two climate views) in the OECD's own key order
and codes, so a key from OECD Data Explorer works unchanged:

```
GET /sdmx/data/OECD.DCD.FSD,DSD_DAC2@DF_DAC2A,1.6/FRA.KEN.206.USD.V?startPeriod=2022&endPeriod=2022&format=csvfile
```

returns France's net ODA to Kenya, 41.99. Each dataflow keys differently (DAC7B
puts SECTOR last); `/sdmx/conformance` lists every key and every deviation.
**Never sum across a wildcard**: an open position returns group totals (`DAC`,
`DPGC`, SECTOR `1000`) beside their members. Ask for the total's code instead.

## Finding codes

`GET /v1/codelists/{dimension}?q=...` resolves a name to a code. Filters also
accept names, ISO3 codes and groups (`ldc`, `dac_members`, `oda`), so
`donor=France` works directly. A group expands to its members, so
`donor=dac_members,France` counts France once; the group's own total row
(`donor=20001`, "DAC Countries, Total") with France is refused.

## Through MCP

`/mcp/` serves four tools: `query_data`, `lookup_codes`, `explain_methodology`
and `create_export`. They reach the same code and the same refusals as the API.
An argument a tool does not have is an error naming it, never silently dropped,
and policy markers go in one object: `"marker": {"gender": "principal"}`.

## The one thing to remember

Cite the vintage and the unit, pass on the warnings and cautions, and do not
quote a figure from the SQL console as if it came from the API. The console
applies none of these rules.
