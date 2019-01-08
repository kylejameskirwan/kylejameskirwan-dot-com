---
layout: post
title:  "8 Simple Data Quality Checks"
subtitle: "Basic test coverage for SQL analytics warehouses."
date:   2019-01-07
categories: data
---

Anyone who's spent much time working with analytics data has run into a data quality issue at some point -- sometimes obvious and temporary, other times subtle and undetected for weeks or months. This post reviews some basic data quality failure cases and the SQL-based tests you can run to monitor for them.

This topic was inpsired by two observations I made while leading product on Uber's internal data quality platform:

1. the lion's share of quality problems discovered by hand (usually triggering a fire for the discoverer) could be caught with simple and boring tests
2. most teams blanked when they opened up the data quality tool for the first time

We often broke the write's block with a simple review of their table and basic tests that they could  -- so I've compiled some common test categories that anyone can use to get started covering their analytics data with test coverage.

*In future posts I plan cover some more complex tests, including between-table tests, and cross-datacenter comparisons.*

---

### Too many or too few rows
The simplest case is to check that any rows exist — which may not be necessary for commonly accessed tables, but might be helpful to use on lookup tables that get looked at by ETLs more than by humans. 

For a partitioned table you can also check that each partition has a sane number of rows (e.g. in a time-partitioned table compare today’s count to the trailing N-day average count) to guard against missing or partially-loaded partitions.

```
WITH
date_counts AS (
  SELECT
    date_partition,
    COUNT(*) AS cnt_rows
  FROM my_table
  WHERE date_partition >= NOW()::DATE - INTERVAL '6 days'
  GROUP BY date_partition),

lst_date_stats AS (
  SELECT cnt_rows
  FROM date_counts
  WHERE date_partition = NOW()::DATE - INTERVAL '1 days'),

prev_5day_stats AS (
  SELECT
    AVERAGE(cnt_rows) + (3 * STDDEV(cnt_rows)) AS cnt_rows_upper_bnd,
    AVERAGE(cnt_rows) - (3 * STDDEV(cnt_rows)) AS cnt_rows_lower_bnd
  FROM date_counts
  WHERE date_partition < NOW()::DATE - INTERVAL '1 day')

SELECT
  cnt_rows < cnt_rows_low_bnd AS is_cnt_rows_low,
  cnt_rows > cnt_rows_up_bnd AS is_cnt_rows_high
FROM lst_date_stats JOIN prev_5day_stats ON TRUE
```

---

### Unexpected NULLs
It’s always wise to check for any nulls in columns that are meant to never have them (e.g. a user_uuid) as this can wreak serious havoc on ETLs, queries, etc. like causing cartesian-product explosions from joining NULLs to NULLs.

```
SELECT COUNT(*)
FROM my_table
WHERE my_column IS NULL
```

For data which is expected to be null occasionally (e.g. a field that didn’t exist several app versions ago) another test can make sense — *but beware* — you are now doing crude anomaly detection. Your alert may no longer indicate bad data, but unexpected user behavior.

```
SELECT
  ROUND(COUNT(CASE WHEN my_column IS NULL THEN 1 END) / COUNT(*), 2) * 100 AS pct_null
FROM my_table
```

---

### Duplicates
Especially true with IDs and UUIDs, it can be important to not have duplicates for some types of data.

WARNING: this check can be especially expensive to run on large datasets.

```
WITH uuid_cnts AS (
SELECT
  my_uuid, COUNT(*) AS cnt_freq
FROM my_table
GROUP BY my_uuid)

SELECT
  COUNT(CASE WHEN cnt_freq > 1 THEN 1 END) AS cnt_duplicates
FROM uuid_cnts
```

---

### Implied types
Sometimes (often for historical reasons) you may find yourself with one *implied* type (say an integer) stored in a column that has a different *actual* type (say a string). Testing that the data is actually an integer, as you expect it to be, can save you if some unexpectedly typed values start appearing — especially if this column is fed into a model or application that can break if there’s a deviation.

**Integers**  
<sup><sub>Regex credited to [this Stack thread](https://stackoverflow.com/questions/9043551/regex-match-integer-only)</sub></sup>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(string_of_ints, '^([+-]?[1-9]\d*|0)$') = 0 THEN 1 END) AS cnt_invalid_integers
FROM my_table
```

**UUIDs**  
<sup><sub>Regex credited to [this Stack thread](https://stackoverflow.com/questions/136505/searching-for-uuids-in-text-with-regex#6640851)</sub></sup>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(user_uuid, '^(\+\d{1,2}\s)?\(?\d{3}\)?[\s.-]\d{3}[\s.-]\d{4}$') = 0 THEN 1 END) AS cnt_invalid_email
FROM my_table
```

---

### Formatted strings
A cousin of the *Implied Types* check, you can use regexes to look for improper email addresses, phone numbers, etc. that are stored in string fields.

**Email addresses**  
<sup><sub>Regex credited to [emailregex.com](https://emailregex.com)</sub></sup>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(email_address, '^[A-Z0-9._%-]+@[A-Z0-9.-]+\.[A-Z]{2,4}$') = 0 THEN 1 END) AS cnt_invalid_email
FROM my_table
```

**Phone numbers**  
<sub><sup>Regex credited to [this Stack thread](https://stackoverflow.com/questions/16699007/regular-expression-to-match-standard-10-digit-phone-number#16699507)</sup></sub>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(phone_number, '^(\+\d{1,2}\s)?\(?\d{3}\)?[\s.-]\d{3}[\s.-]\d{4}$') = 0 THEN 1 END) AS cnt_invalid_phone
FROM my_table
```

**US ZIP Codes**  
<sub><sup>Regex credited to [this Stack thread](https://stackoverflow.com/a/7185241/1709587)</sup></sub>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(zip_code, '\d{5}([ \-]\d{4})?') = 0 THEN 1 END) AS cnt_invalid_zip
FROM my_table
```

**Credit Card Numbers**  
<sub><sup>NOTE: Looks for continuous numbers with no spaces or delimiters between.
Regex credited to [this Stack thread](https://stackoverflow.com/a/7185241/1709587)</sup></sub>

```
SELECT
  COUNT(CASE WHEN REGEXP_INSTR(cc_number, '^(?:4[0-9]{12}(?:[0-9]{3})?|[25][1-7][0-9]{14}|6(?:011|5[0-9][0-9])[0-9]{12}|3[47][0-9]{13}|3(?:0[0-5]|[68][0-9])[0-9]{11}|(?:2131|1800|35\d{3})\d{11})$') = 0 THEN 1 END) AS cnt_cc_number
FROM my_table
```

---

### Enums
If you know all values that should be allowed in the field, you may want to be alerted when something new shows up.

```
SELECT COUNT(*)
FROM my_table
WHERE my_column NOT IN (enum_1, enum_2, enum_3)
```

---

### Timestamps
If some timestamps should always occur before others (e.g. created-at and updated-at) you can check for out-of-order timestamps for each step in your funnel.

```
SELECT
  COUNT(CASE WHEN timestamp_2 < timestamp_1 THEN 1 END) AS cnt_ooo_a,
  COUNT(CASE WHEN timestamp_3 < timestamp_2 THEN 1 END) AS cnt_ooo_b
FROM my_table
```

You can also check for timestamps created in the future, or too far in the past.

```
SELECT
  COUNT(CASE WHEN timestamp >= NOW() + INTERVAL '1 minute' THEN 1 END) AS cnt_future_ts,
  COUNT(CASE WHEN timestamp <= NOW() - INTERVAL '5 years' THEN 1 END) AS cnt_ancient_ts
FROM my_table
```

---

### Numerics
Writing these sometimes requires more granular knowledge of your product/business logic, but can alert you when reality deviates from that theoretical logic. Check that non-zero numbers are positive, that mins and maxes are correct.

```
SELECT COUNT(*)
FROM my_table
WHERE my_column < min_value
```

```
SELECT COUNT(*)
FROM my_table
WHERE my_column > max_value
```

*NOTE: It’s easy to start using these as crude anomaly detection tools — if you do this be sure to indicate clearly for your teammates which respond to business behavior and which respond to damaged data.*

---

## How to get started
If you want to do it all yourself, set up the queries as jobs on a DAG orchestrator (such as [Airflow](https://airflow.apache.org/) or [Luigi](https://luigi.readthedocs.io/en/stable/)), then connect that to your notification channels (Slack, Pagerduty, etc.). Easy!

**If you want to get going faster, sign up for [Toro Data Quality](https://torodata.io/)**. Toro makes it easy to deploy tests in minutes, and saves your data engineering team the overhead of maintaining a reliable test orchestrator. All you need to start is a read-only account on your database or warehouse.

---

# Other posts you may enjoy
- Between-table tests (coming soon!)
- Cross-datacenter testing (coming soon!)
- Anomaly detection vs. data-damage detection (coming soon!)
