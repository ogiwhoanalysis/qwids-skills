# What the skills change

Three worked examples. Every number here was read off a live deployment
(vintage 2024.1) rather than written from memory, and each one shows a case
where an agent with the skill answers differently from one without it.

---

## 1. The same question, two tables, two correct answers

**Asked:** *"How much aid did France give Kenya in 2022?"*

An agent without the skill picks an endpoint, gets a number, and reports it.

An agent with `qwids-development-finance` lets QWIDS route, and then notices
what the envelope is telling it:

```bash
curl "$QWIDS/v1/datasets/auto/data?donor=France&recipient=Kenya&years=2022&measures=disbursement"
```

```
dataset : dac2a
value   : 41.99  (USD millions)
note    : "DAC2A holds several aid type values that must not be added together,
           so 'ODA: Total Net' was applied by default. Set aid_type= to choose
           another."
```

Add one coordinate DAC2a cannot express and the router moves to CRS, which
answers a different question:

```bash
curl "$QWIDS/v1/datasets/auto/data?donor=France&recipient=Kenya&years=2022&measures=disbursement&flow=oda"
```

```
dataset          : crs
usd_disbursement : 80.38
```

**41.99 and 80.38 are both right.** DAC2a is the published aggregate on a
specific aid type applied by default; CRS is activity-level ODA gross
disbursements. The failure mode is quoting one of them as "the" answer.

What the skill adds: report the number **with the table that produced it and
the note that shaped it**, and offer the other when the difference matters.

---

## 2. A policy marker is not a percentage of everything

**Asked:** *"Which donors focus most on gender equality?"*

The trap: dividing gender-marked aid by total aid. Activities that were never
screened for the marker are **not zeros**, they are absent, and including them
in the denominator invents a number.

The published indicator is the share of *screened* aid:
`(principal + significant) / (not targeted + significant + principal)`.

```bash
curl "$QWIDS/v1/datasets/auto/data?marker.gender=not_targeted,significant,principal\
&group_by=donor,gender_marker&years=2022&measures=disbursement"
```

Answered by GENDER, the published table. 2022, providers with a screened base
over USD 1bn:

| Provider | Share of screened aid | Screened base |
|---|---|---|
| Netherlands | 80% | 3,067m |
| Sweden | 75% | 2,609m |
| United Kingdom | 65% | 5,740m |
| Switzerland | 63% | 2,219m |
| Canada | 62% | 5,867m |
| Korea | 20% | 2,169m |
| United States | 15% | 41,263m |

What the skill adds: the right denominator, the right table, and the base
reported alongside the share, because 80% of 3bn and 15% of 41bn are not
comparable claims.

---

## 3. The SQL console will happily return a number 4.6x too big

**Asked:** *"Total non-concessional flows in DAC2B for 2022."*

The obvious query:

```sql
SELECT sum(value) FROM dac2b WHERE year = 2022 AND aidtype = 296
```

```
3,761,709.7
```

That figure is wrong. DAC2B carries group totals such as "DAC Countries, Total"
in the same column as their members, so the sum counts most of the money twice.
Splitting the rows shows it:

```sql
SELECT p.is_aggregate, count(*) AS rows, round(sum(d.value), 1) AS usd_m
FROM dac2b d JOIN dim_provider p ON p.code = d.donor
WHERE d.year = 2022 AND d.aidtype = 296
GROUP BY 1 ORDER BY 1
```

```
is_aggregate | rows | usd_m
false        | 3028 |   824,230.4     <- the members
true         | 3540 | 2,937,479.3     <- totals that already contain them
```

**824,230.4 is the answer. The naive query was 4.6x too big.**

The typed API excludes those rows by default and says so in `warnings[]`. The
console does not, because there you are writing the SQL. That is the single
most useful thing `qwids-sql-console` carries.

---

## 4. A refusal is an answer

**Asked:** *"Total ODA from DAC countries and France in 2022."*

QWIDS refuses rather than warning, because no caveat rescues a number someone
is about to quote. An agent without the skill sees an error and tries to work
around it, often by pinning a table or switching to the SQL console, and
produces the wrong number by a different route.

An agent with the skill recognises the refusal as correct and says why: "DAC
countries, total" already contains France, so adding them double-counts. The
useful reply is the question reformulated, not the guard defeated.

---

## Checking any of this yourself

Everything above is reproducible from a shell:

```bash
QWIDS=https://qwids-web.whitetree-87b148ac.francecentral.azurecontainerapps.io
curl -s "$QWIDS/v1/sql/schema" | jq '.tables[].name'       # what the console holds
curl -s "$QWIDS/v1/routing"    | jq '.priority'            # the routing order
curl -s "$QWIDS/v1/resolve?donor=France&recipient=Kenya"   # why a table was chosen
```
