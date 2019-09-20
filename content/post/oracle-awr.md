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

## WORKLOAD REPOSITORY report for
关注db time和elapsed之间的关系，elapsed(Elapsed real time, real time, wall-clock time)指报告周期内的时间，db time指系统花费在数据库运行上的时间，这个时间包括等待cpu资源可用的时间（可运行队列），所以这个时间有可能会大于elapsed*cpus

## Load Profile
DB Time = DB CPU + Non-Idle wait time (so, Non-Idle wait time = DB Time - DB CPU)[^2]

DB CPU 也不包括等待在cpu运行队列,统计用的接口是getrusage, (This measure typically does not include time spent waiting on CPU run queues)


## Instance Efficiency Percentages (Target 100%)
Execute to Parse %: 这个值越接近100%越好（执行次数/解析次数）
Soft Parse %:       硬解析的比例等于1-(Soft Parse %:)

The most important Statistic presented here from the point of view of our example is the '% Non-Parse CPU' because this indicates that almost all the CPU  time that we see in the Top Waits section is attributable to Execution and not parse, which means that tuning SQL may help to improve this.

If we were tuning, then 94.48% soft parse rate would show a small proportion of hard parsing which is desirable.  The high execute to parse % indicates good usage of cursors.  Generally, we want the statistics here close to 100%, but remember that a few percent may not be relevant dependent on the application.  For example, in a data warehouse environment, hard parsing may be higher due to usage of materialized views and, or histograms.  So again comparing to baseline report when performance was good is important.

## IO Profile
Suddenly life is more simple. You want to know the total IOPS and throughput? It’s all in one place. You want to calculate the ratio of reads to writes? Just compare the read and write columns. Happy days.[^3]

One word of warning though: there are other database processes driving I/O which may not be tracked in these statistics. I see no evidence for control file reads and writes being shown, although these are insignificant in magnitude. More significant would be I/O from the archiver process for databases running in archive log mode, as each redo log must be sequentially read and re-written out as an archive log. Are these included? Yet another possibility would be the Recovery Writer (RVWR) process which is responsible for writing flashback logs when database flashback logging is enabled. [Discussions with Jonathan Lewis suggest these stats are all included – and let’s face it, he wrote the book on the subject…!]  It all adds up… Oracle really needs to provide better clarity on what these statistics are measuring.


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
SET LINESIZE 130
SET PAGESIZE 0


# snapshot
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

# Execution Plan[^1]
/*+ GATHER_PLAN_STATISTICS */

alter session set statistics_level='ALL';

/execute sql/
SELECT * FROM table(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'ALLSTATS LAST ALL +OUTLINE'));


SELECT * FROM TABLE(DBMS_XPLAN.display_cursor(sql_id=>'cwq13zuqc68hz', format=>'ALLSTATS LAST +outline +PEEKED_BINDS'));

select * from table(dbms_xplan.display_awr(sql_id=>'48u49a2dxufrm', format=>'ALLSTATS LAST +outline +PEEKED_BINDS'));


select * from (
select
     ash.SQL_ID , ash.SQL_PLAN_HASH_VALUE Plan_hash, aud.name type,
     sum(decode(ash.session_state,'ON CPU',1,0))     "CPU",
     sum(decode(ash.session_state,'WAITING',1,0))    -
     sum(decode(ash.session_state,'WAITING', decode(wait_class, 'User I/O',1,0),0))    "WAIT" ,
     sum(decode(ash.session_state,'WAITING', decode(wait_class, 'User I/O',1,0),0))    "IO" ,
     sum(decode(ash.session_state,'ON CPU',1,1))     "TOTAL"
from dba_hist_active_sess_history ash,
     audit_actions aud
where SQL_ID is not NULL
   -- and ash.dbid=&DBID
   and ash.sql_opcode=aud.action
   -- and ash.sample_time > sysdate - &minutes /( 60*24)
group by sql_id, SQL_PLAN_HASH_VALUE   , aud.name
order by sum(decode(session_state,'ON CPU',1,1))   desc
) where  rownum < 20


# oracle patch

## 查看补丁列表
`/cmcc/app/oracle/product/12.1.0/OPatch/opatch lsinventory`

# 常用SQL
## 获取表上所有索引的DDL
set long 9999
`select dbms_metadata.get_ddl('INDEX',index_name) from user_indexes where table_name=upper('wfp_approval_organization');`


[^1]: [https://blogs.oracle.com/optimizer/how-to-generate-a-useful-sql-execution-plan](How to Generate a Useful SQL Execution Plan)
[^2]:[What Elapsed Time, DB Time and DB CPU represent in AWR report and how to calculate Wait time (Non-Idle)](http://deepakbhatnagardba.blogspot.com/2015/12/what-elapsed-time-db-time-and-db-cpu.html)

[^3]: [Oracle AWR Reports: Understanding I/O Statistics](https://flashdba.com/2014/02/26/oracle-awr-reports-understanding-io-statistics/)
