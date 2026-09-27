---
name: qwids-sql-console
description: Write read-only SQL against the OECD QWIDS 2.0 warehouse through its public SQL console, including which tables and columns exist and which methodology rules do NOT apply there. Use when a development-finance question needs a join, a window function or a shape the /v1 API cannot express.
---

# QWIDS 2.0: the read-only SQL console

`$QWIDS` below is the QWIDS base URL: ask the user for theirs, or default to
the current development deployment,
`https://qwids-web.whitetree-87b148ac.francecentral.azurecontainerapps.io`.

`POST /v1/sql` runs one read-only SQL statement against a sealed DuckDB copy of
the warehouse and returns rows. It is a side door for people who already know
the tables.

```bash
curl -s -X POST "$QWIDS/v1/sql" -H 'content-type: application/json' \
  -d '{"sql":"SELECT d.name_en, round(sum(c.usd_disbursement),1) AS usd
             FROM crs c JOIN dim_provider d ON d.code = c.donor_code
             WHERE c.year = 2022 GROUP BY 1 ORDER BY 2 DESC LIMIT 10"}'
```

`GET /v1/sql/schema` returns every table and column with its type. **Read it
first** rather than guessing column names; it also reports the row limit, the
timeout and the vintage.

## The rule that matters most

**The console applies none of QWIDS's methodology rules.** No double-counting
guard, no exclusion of subtotal rows, no routing. A query that adds an aggregate
to its own members returns a number, and the number is wrong.

Specifically, the DAC tables hold two things in the one `value` column that
must never be added:

- **Group totals beside their members**, for providers ("DAC Countries,
  Total") and for recipients ("Developing countries, total") alike. Exclude
  them with `dim_provider.is_aggregate` and `dim_recipient.is_aggregate`.
- **Both price bases**: `price = 'A'` is current prices, `price = 'D'`
  constant. Pick one.

`SELECT sum(value) FROM dac2b` does both, and comes out about 80 times too big
(EXAMPLES.md, case 3). The typed API excludes those rows and says it did; here
you have to exclude them yourself.

If the figure is going into a publication, get it from `/v1/datasets/...` or the
Explorer instead, and use the console to explore.

## What is in it

Twenty-five tables: the thirteen fact tables (`crs`, `dac1`, `dac2a`, `dac2b`,
`dac3a`, `dac4`, `dac5`, `dac7b`, `cpa`, `gender`, `mobilised`,
`crdf_provider`, `crdf_recipient`) and twelve `dim_*` reference tables that
give readable labels (`dim_provider`, `dim_recipient`, `dim_purpose`,
`dim_channel`, `dim_finance`, `dim_modality`, `dim_aidtype`, `dim_agency`,
`dim_marker`, `dim_dacsector`, `dim_bimulti`, `dim_incomegroup`).

Most dims are keyed on `code` and carry `name_en` and `name_fr`:
`JOIN dim_provider d ON d.code = c.donor_code`. Three differ:

- `dim_recipient` is keyed on `recipient_code`.
- `dim_aidtype` and `dim_dacsector` are shared by several tables, so they are
  keyed on `(dataset, code)` and named by `label`, in English only. Join on
  both: `JOIN dim_aidtype a ON a.dataset = 'dac2b' AND a.code = d.aidtype`.

**`crs` here has 100 of its 103 columns.** `project_title`,
`short_description` and `long_description` are left out: they are 348 MB of the
648 MB the table occupies, and excluding them keeps the console's copy roughly
the size of the warehouse it mirrors. The climate tables leave out their title
and description the same way. **Free-text search is therefore not possible in
the console.** Use `q=` on `/v1/datasets/crs/data` (or on `crdf_provider`,
`crdf_recipient`), or the Activity search page, for that.

The cubes are not exposed either. Write your own `GROUP BY`.

## Limits, and what they mean

- **One statement.** Two statements are refused so you get one result grid
  rather than silently seeing only the last one's output.
- **Row cap**, default 1,000 and at most 10,000 for JSON; the response sets
  `truncated: true` when there was more. CSV downloads carry more.
- **10-second wall clock.** A runaway query is interrupted and returns 408.
- **Rate limited** at the edge. Space out programmatic calls.
- A replica builds its copy at startup, so `/v1/sql` can answer **503 "still
  starting up"** for a minute or two after a deployment. That is a wait, not a
  failure; retry.

## What it cannot do, by construction

The connection is opened read-only with external access disabled, which is a
one-way latch. So these all fail, and that is the design rather than a blocklist
to probe:

- writing anything (`CREATE`, `INSERT`, `COPY`)
- reading files or blobs (`read_csv('/etc/passwd')`, `read_parquet('az://...')`)
- `ATTACH`, `INSTALL`, `getenv()`
- re-enabling external access

Do not spend turns testing the sandbox. Spend them on the query.

## Checking your answer

The console and the typed API read the same warehouse, so a figure with no
aggregate rows involved should match:

```sql
SELECT donor_code, sum(usd_disbursement) FROM crs WHERE year = 2022 GROUP BY 1
```

against `/v1/datasets/crs/data?years=2022&group_by=donor&measures=disbursement`.
For a DAC table, keep one price base and no group totals on either side:

```sql
SELECT round(sum(d.value), 1)
FROM dac2b d
JOIN dim_provider p ON p.code = d.donor
JOIN dim_recipient r ON r.recipient_code = d.recipient
WHERE d.year = 2022 AND d.aidtype = 296 AND d.price = 'A'
  AND NOT p.is_aggregate AND NOT r.is_aggregate
```

gives 46,986.5, the total of
`/v1/datasets/dac2b/data?years=2022&aid_type=296&group_by=donor`. If they
disagree, the usual cause is subtotal rows or a second price base in your SQL.
