---
title: "理解Oracle AWR报告"
date: 2014-03-24
lastmod: 2014-03-24
draft: false
tags: ["tech", "oracle", "database", "tuning", "performance"]
categories: ["tech"]
description: "对Oracle AWR报告的一些理解，需要慢慢完善"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Oracle AWR
Automatic Workload Repository gathers, processes and maintains performance stats used for problem detection and self-tuning the databases. This collective information can be displayed as views and reports (HTML/text) to better show the current database performance status for further analysis.

AWR gathers the following data:
1. Object Statistics (access / usage stats of DB segments)
2. Time Model Statistics (V$SYS_TIME_MODEL and V$SESS_TIME_MODEL views)
3. Some of the System and Session Statistics (V$SYSSTAT and V$SESSTAT views)
4. ASH (Active Session History) Statistics
5. High load generating SQL Statements

Different Components that uses AWR are:
1. Automatic Database Diagnostic Monitor
2. Undo Advisor
3. SQL Tuning Advisor
4. Segment Advisor

By default, Snapshots are generated every hour and retains the statistics in the repository for 8 days

## awr报告对比
@?/rdbms/admin/awrddrpt.sql
## sqlid执行对比
awrddrpi.sql

## Instance Efficiency Percentages (Target 100%)
Execute to Parse %: 这个值越接近100%越好（执行次数/解析次数）
Soft Parse %:       硬解析的比例等于1-(Soft Parse %:)


# Oracle ADDM
Automatic Database Diagnostic Monitoring Report analyzes the AWR data on a regular basis, to give you overview of the root cause of the problem which is affecting your database's performance. It also provides suggestions or recommendations for rectifying any problem identified and lists the areas which are having no issues. ADDM recommends multiple solutions for the DBA to choose from which includes,

1. Database Configuration
2. Hardware changes
3. Schema changes
4. Application changes &
5. Other advisories

ADDM refines the report based on the key factor DB time (cumulative time spent by database to process user requests). Below mentioned is the script to generate ADDM:

`@$ORACLE_HOME/rdbms/admin/addmrpt.sql`

# ASH (Active Session History) Report
Active Session History Reports shows you the below mentioned stats regarding your Oracle Database,

1. Transient performace problems (usually not captured in ADDM reports)
2. Time / session / action /module / SQL_ID based target performance analysis

Below mentioned is the script which you can run to generate your ASH report,

`@$ORACLE_HOME/rdbms/admin/ashrpt.sql`

For Oracle RAC Environment :

`@$ORACLE_HOME/rdbms/admin/ashrpti.sql`
# SQLPLUS
# format output
set linesize 250
column colunm_name format a30
