# What the skills change

Four worked examples. Every number here was read off a live deployment
(vintage 2024.1) rather than written from memory, and each one shows a case
where an agent with the skill answers differently from one without it.

---

## 1. The same question, two tables, two correct answers

**Asked:** *"How much aid did France give Kenya in 2022?"*

An agent without the skill picks an endpoint, gets a number, and reports it.

An agent with `qwids-development-finance` lets QWIDS route, and then reads what
the envelope is telling it:

```bash
curl "$QWIDS/v1/datasets/auto/data?donor=France&recipient=Kenya&years=2022&measures=disbursement"
```

```
dataset : dac2a
value   : 41.99  (USD million)
note    : "DAC2a holds several aid type values that must not be added together,
           so 'ODA: Total Net' was applied by default. Set aid_type= to choose
           another."
meta.cross_check.crs:
  comparable : dac2a 240 "Memo: ODA Total, Gross disbursements"  116.86
  differs_by : ["net_of_receipts"]
```

The same envelope already names the other figure. Ask CRS for ODA and it
answers that one:

```bash
curl "$QWIDS/v1/datasets/auto/data?donor=France&recipient=Kenya&years=2022&measures=disbursement&flow=oda"
```

```
dataset          : crs
usd_disbursement : 116.86
meta.cross_check.same_figure: dac2a 240, 116.86, ties
```

**41.99 and 116.86 are both right.** DAC2a's default is ODA *net* of loan
repayments and recoveries; CRS adds up gross disbursements activity by activity,
and ties to DAC2a's gross memo row (116.86 both ways). The failure mode is
quoting one of them as "the" answer.

What the skill adds: report the number **with the table that produced it, the
note that shaped it, and whether it is net or gross**, and offer the other when
the difference matters.

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

## 3. The SQL console will happily return a number 80 times too big

**Asked:** *"Total non-concessional flows in DAC2B for 2022."*

The obvious query:

```sql
SELECT sum(value) FROM dac2b WHERE year = 2022 AND aidtype = 296
```

```
3,761,709.7
```

That figure is wrong twice over. DAC2B carries group totals ("DAC Countries,
Total", "Developing countries, total") in the same column as their members, and
it carries every figure twice, in current (`price = 'A'`) and constant
(`price = 'D'`) prices. Splitting the rows shows it:

```sql
SELECT d.price, p.is_aggregate AS provider_total, r.is_aggregate AS recipient_total,
       count(*) AS rows, round(sum(d.value), 1) AS usd_m
FROM dac2b d
JOIN dim_provider p ON p.code = d.donor
JOIN dim_recipient r ON r.recipient_code = d.recipient
WHERE d.year = 2022 AND d.aidtype = 296
GROUP BY ALL ORDER BY ALL
```

```
price | provider_total | recipient_total | rows |       usd_m
A     | false          | false           |  483 |    46,986.5   <- the answer
A     | false          | true            | 1031 |   348,121.9
A     | true           | false           | 1284 |   190,834.6
A     | true           | true            |  486 | 1,217,541.6
D     | false          | false           |  483 |    50,570.1   <- the same, constant prices
D     | false          | true            | 1031 |   378,551.9
D     | true           | false           | 1284 |   207,363.4
D     | true           | true            |  486 | 1,321,739.7
```

**46,986.5 is the answer, in current prices. The naive query was 80 times too
big.** It matches the typed API's 46,986.46 for
`/v1/datasets/dac2b/data?years=2022&aid_type=296&group_by=donor`.

The typed API excludes those rows by default and says so in `warnings[]`. The
console does not, because there you are writing the SQL. That is the single
most useful thing `qwids-sql-console` carries.

---

## 4. A refusal is an answer

**Asked:** *"Total ODA from DAC countries and France in 2022."*

Sent with the group's own total row, the question is refused:

```bash
curl "$QWIDS/v1/datasets/auto/data?donor=20001,France&years=2022"
```

```
422  code: aggregate_mixed_with_member
     "donor: [20001] is an aggregate that already includes [4]. Summing them
      double-counts. Pick one or the other."
```

QWIDS refuses rather than warning, because no caveat rescues a number someone
is about to quote. An agent without the skill sees an error and tries to work
around it, often by pinning a table or switching to the SQL console, and
produces the wrong number by a different route.

An agent with the skill recognises the refusal as correct and says why: "DAC
Countries, Total" already contains France. The useful reply is the question
reformulated, not the guard defeated. Asked as a group, `donor=dac_members,France`,
it is answered: the group expands to its members, France is counted once, and a
note says the group includes EU Institutions.

---

## Checking any of this yourself

Everything above is reproducible from a shell:

```bash
QWIDS=https://qwids-web.whitetree-87b148ac.francecentral.azurecontainerapps.io
curl -s "$QWIDS/v1/sql/schema" | jq '.tables[].name'       # what the console holds
curl -s "$QWIDS/v1/routing"    | jq '.priority'            # the routing order
curl -s "$QWIDS/v1/resolve?donor=France&recipient=Kenya"   # why a table was chosen
```
