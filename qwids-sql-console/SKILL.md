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

Specifically, the DAC tables contain group-total rows such as "DAC Countries,
Total" in the same column as their members. `SELECT sum(value) FROM dac2b`
double-counts. The typed API excludes those rows and says it did; here you have
to exclude them yourself.

If the figure is going into a publication, get it from `/v1/datasets/...` or the
Explorer instead, and use the console to explore.

## What is in it

Twenty tables: the ten fact tables (`crs`, `dac1`, `dac2a`, `dac2b`, `dac3a`,
`dac4`, `dac5`, `dac7b`, `cpa`, `gender`) and the `dim_*` reference tables that
give readable labels (`dim_provider`, `dim_recipient`, `dim_purpose`,
`dim_channel`, `dim_finance`, `dim_modality`, `dim_aidtype`, `dim_agency`,
`dim_marker`, `dim_dacsector`).

Join a fact table to a dim on its code column to get names:
`JOIN dim_provider d ON d.code = c.donor_code`. Every dim carries `name_en` and
`name_fr`.

**`crs` here has 100 columns, not the full 102.** `project_title`,
`short_description` and `long_description` are left out: they are 348 MB of the
648 MB the table occupies, and excluding them keeps the console's copy roughly
the size of the warehouse it mirrors. **Free-text search is therefore not
possible in the console.** Use `GET /v1/datasets/crs/data?q=...` or the Activity
search page for that.

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
If they disagree, the usual cause is subtotal rows in your SQL.
