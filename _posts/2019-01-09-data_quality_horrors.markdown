---
layout: post
title:  "Data Quality Horror Stories"
date:   2019-01-09
categories: data
---

Names, genders, timestamps, and column names have all been modified to protect the identities of the victims. The storylines are, sadly, very real.

---

## Noncents
An analyst was developing a dashboard. For some reason all the USD numbers on the dashboard were whole: none of them had any cent values. After much upstream-sleuthing, he discovered that the column the transaction amounts were coming from had type set to INTEGER.

---

## Down and to the right
The product analytics manager was woken at 2:00AM by his team’s product manager. The dashboard tracking growth in India showed a precipitous drop in their key engagement metric.

The analytics manager woke his team mate who had built the dashboard. While debugging over Slack (now at 3:00-something AM) they realized that an upstream ETL was failing, and their aggregate table had only partially populated, simulating a sudden drop in business.

They kicked the ETL job and went back to bed.

---

## Append-icitus
The ETL that ingested the source CSV adds an additional `etl_timestamp` column when writing to its destination table. An analyst had added a `config_type` column to the schema of the CSV at some point, pushing this `etl_timestamp` column further to the right. The next ETL in the pipeline was not resilient to schema changes, and was now using the new `config_type` column as if it were a timestamp.

Right data, wrong column.

---

Have your own data quality horrors? Toro Data’s DataMonitor makes it easy to deploy the basic tests you should be running on your analytics data warehouse.

With just [8 types of tests](http://torodata.io/blog/8-tests) you can start catching unexpected schema changes, ETL and ingestion failures, off-by-one-column errors, and other annoyances too often discovered by hand (or by query). It sets up quickly with just a read-only account, and tests are easy to write in plain old SQL.

<center><a href="https://torodata.io/">Sign up here to be notified when our beta is available.</a></center>

---

Good luck 🤞  
Team Toro