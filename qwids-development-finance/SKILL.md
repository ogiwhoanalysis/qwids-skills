---
name: qwids-development-finance
description: Get defensible OECD development-finance figures (ODA, CRS, the DAC tables) out of QWIDS 2.0, and read the answer correctly. Use when asked about aid flows, ODA volumes, donors or recipients of development finance, sector or purpose breakdowns, policy markers such as gender or climate, or when a figure needs a citation and a data vintage.
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

There are ten queryable tables (DAC1, DAC2a, DAC2b, DAC3a, DAC4, DAC5, DAC7b,
CPA, GENDER, CRS). Do not choose one unless you have a reason. Use `auto`:

```
GET /v1/datasets/auto/data?donor=France&years=2022&group_by=recipient&measures=disbursement
```

QWIDS tries them coarsest first and answers from the first that can express
every coordinate you selected. `meta.dataset` says which answered.

A table is ruled out by what it cannot carry, not by a hand-written rule: name a
recipient and DAC1 and DAC5 drop out because they have no recipient dimension;
ask for a five-digit purpose code and every DAC table drops out because only CRS
records purposes at that depth. `GET /v1/resolve?...` returns the verdict per
table with `blocking[]` and a `reason`, and costs no data read. Use it when a
user asks why they got a particular table.

Pin a table only when asked to: `dataset=crs`. The response then carries
`routing.forced: true` and lists the tables that would also have answered.

## Read the whole envelope, not just the number

Every response carries:

- `meta.vintage` and `meta.citation`. **Always quote the vintage.** Two requests
  with the same query and vintage return identical numbers; without it a figure
  is not reproducible.
- `warnings[]`. These are not decoration. They say things like "subtotal rows
  were excluded so the breakdown sums to the true total", or that a figure may
  be understated. Pass them on.
- `notes[]`. Defaults that were applied for you, and parameters that were
  ignored.
- `totals`. The figure for the whole selection, consistent with the rows.

If `warnings[]` is non-empty, say what it says. A number quoted without its
caveat is the failure mode this platform is built to avoid.

## What it will refuse, and why that is correct

QWIDS refuses combinations that produce a wrong number rather than warning about
them: an aggregate together with its own members, Part I and Part II of the DAC
List in one figure, a ratio summed as money, memo items mixed with non-memo
ones, a parent code with a code it already contains.

If you get a refusal, do not work around it by pinning a table or by adding
`include_aggregates=true`. Reformulate the question.

`include_aggregates=true` exists to read a table's structure. Never sum what it
returns.

## Measures and prices

- **Commitment** is a firm written obligation; **disbursement** is the actual
  transfer. They are different measures and are never mixed in one column.
  Disbursements can lag commitments by years.
- **Grant equivalent** is reported from 2018 onwards. Earlier years are
  structurally empty, not missing.
- `price_base=constant` gives USD millions in 2024 prices, for comparing volumes
  across time. `current` is the prices of each year. Never mix them in one
  series, and say which you used.

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
span two fields. It is not a pattern: `%` and `_` match themselves.

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

## Finding codes

`GET /v1/codelists/{dimension}?q=...` resolves a name to a code. Filters also
accept names, ISO3 codes and groups (`ldc`, `dac_members`, `oda`), so
`donor=France` works directly.

## The one thing to remember

Cite the vintage, pass on the warnings, and do not quote a figure from the SQL
console as if it came from the API. The console applies none of these rules.
