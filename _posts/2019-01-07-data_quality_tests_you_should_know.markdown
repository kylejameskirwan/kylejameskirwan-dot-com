---
layout: post
title:  "Data Quality Tests You Should Know"
date:   2019-01-07
categories: data
---

It seems like there isn’t a data scientist out there who hasn’t stumbled over a data quality issue at some point. Sometimes they’re temporary and obvious, other times subtle and undetected for weeks (or months). In most cases I’ve observed, the issue ~~could~~ should have easily been caught with simple testing — not for some esoteric edge case, but just simple test coverage.

When I was working on the internal data quality product at Uber, I often saw customer teams got bogged down not by using the tool itself, but thinking through what tests to write. Once we reviewed their data with them and went through seemingly-stupidly-simple test cases, they got going on their own and achieved strong test coverage afterward.

This post provides a list of some data quality tests inspired by these stories, some that I’ve witnessed myself, and others relayed to me by former colleagues and industry friends. At the end we discuss briefly how to implement these, as well as plug our own product, Toro Data Quality, which makes it easy to spin these up and run them in your own analytics warehouse.

All of these can all be written without knowing anything about the relationship between this table an any other — you simply need to know business logic for the product or process represented by the analytics data in the table.

In future posts we plan to cover more advanced tests like between-table tests to check for ETL-job mistakes, cross-datacenter tests, and more.

---

### Too many or too few rows
The simplest case is to check that any rows exist — which may not be necessary for commonly accessed tables, but might be helpful to use on lookup tables that ETLs use but that humans don’t. More commonly useful would be checking that each partition in a partitioned table has a sane number of rows (e.g. in a time-partitioned table compare today’s count to the trailing N-day average count) to guard against missing or partially-loaded partitions.

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
    AVERAGE(cnt_rows) + (3 * STDDEV(cnt_rows)) AS cnt_rows_upper_bound,
    AVERAGE(cnt_rows) - (3 * STDDEV(cnt_rows)) AS cnt_rows_lower_bound
  FROM date_counts
  WHERE date_partition < NOW()::DATE - INTERVAL '1 day')

SELECT cnt_rows BETWEEN cnt_rows_lower_bound AND cnt_rows_upper_bound AS cnt_rows_out_of_range
FROM lst_date_stats JOIN prev_5day_stats ON TRUE
```

*NOTE: If you're willing to trade higher querying load for more detail in your alert, the above test can easily be written as two tests, one for upper and one for lower bound deviations. This way your alert provides more granular information.*

---

### Unexpected NULLs
It’s always wise to check for any nulls in columns that are meant to never have them (e.g. a user_uuid) as this can wreak serious havoc on ETLs, queries, etc.

```
SELECT COUNT(*)
FROM my_table
WHERE my_column IS NULL
```

For data which is expected to be null occasionally (e.g. a field that didn’t exist several app versions ago) another test can make sense — *but beware* — you are now doing crude anomaly detection. Your alert may no longer indicate bad data, but unexpected user behavior.

```
SELECT ROUND(COUNT(CASE WHEN my_column IS NULL) / COUNT(*), 2) * 100 AS pct_null
FROM my_table
```

---

### Implied types
Sometimes (often for historical reasons) you may find yourself with one *implied* type (say an integer) stored in a column that has a different *actual* type (say a string). Testing that the data is actually an integer, as you expect it to be, can save you if some unexpectedly typed values start appearing — especially if this column is fed into a model or application that can break if there’s a deviation.

TODO: Here’s a cheatsheet for the relevant regexes / LIKE statements….

---

### Formatted strings
A cousin of the *Implied Types* check, ensure that email addresses, phone numbers, etc. that are stored in string fields are in their respective formats properly by using regexes.

TODO: Here’s a cheatsheet for the relevant regexes / LIKE statements….

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
If some timestamps should always occur before others (e.g. created-at and updated-at) you can check their sequence. You can also check for timestamps created in the future, or too far in the past.

```
SELECT COUNT(*)
FROM my_table
WHERE timestamp_2 < timestamp_1
```

```
SELECT COUNT(*)
FROM my_table
WHERE timestamp >= NOW() + INTERVAL '1 minute'
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

## Ok cool, so how do I do this?
If you want to do it all yourself, set up queries yourself on a schedule using Airflow, Luigi, or another scheduler/orchestrator. TODO: DETAILS ON HOW TO ACTUALLY DO THIS?

If you want to get going faster and with less overhead, sign up for Toro Data Quality. All you need is a read-only account and you can start writing and deploying tests on your analytics data.

---

# Other posts you may enjoy
- Between-table tests (coming soon!)
- Cross-datacenter testing (coming soon!)
- Anomaly detection vs. data-damage detection (coming soon!)
