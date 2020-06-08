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
"Elapsed" Time is the AWR report time is the "End Snap" time minus the "Begin Snap" time.

DB CPU 也不包括等待在cpu运行队列,统计用的接口是getrusage, (This measure typically does not include time spent waiting on CPU run queues)

The DB Time uses the gettimeofday() call, 'DB Time' is the amount of elapsed time (in microseconds) spent performing Database user-level calls including non-idle wait time, cpu run queues, on cpu, but not including the elapsed time spent on instance background processes such as PMON. The DB Time is generated using the following query:
``` sql
SELECT Round(NVL((e.value - s.value),-1)/60/1000000,2)||' minutes' "DB Time"
FROM   DBA_HIST_SYS_TIME_MODEL s,
       DBA_HIST_SYS_TIME_MODEL e
WHERE  s.snap_id = &AWRStartSnapID AND
       e.snap_id = &AWREndSnapID anD
       e.dbid = s.dbid AND
       e.instance_number = s.instance_number AND
       s.stat_name = 'DB time' AND
       e.stat_id = s.stat_id;
```

## Instance Efficiency Percentages (Target 100%)
Execute to Parse %: 这个值越接近100%越好（执行次数/解析次数）
Soft Parse %:       硬解析的比例等于1-(Soft Parse %:)

The most important Statistic presented here from the point of view of our example is the '% Non-Parse CPU' because this indicates that almost all the CPU  time that we see in the Top Waits section is attributable to Execution and not parse, which means that tuning SQL may help to improve this.

If we were tuning, then 94.48% soft parse rate would show a small proportion of hard parsing which is desirable.  The high execute to parse % indicates good usage of cursors.  Generally, we want the statistics here close to 100%, but remember that a few percent may not be relevant dependent on the application.  For example, in a data warehouse environment, hard parsing may be higher due to usage of materialized views and, or histograms.  So again comparing to baseline report when performance was good is important.

## IO Profile
Suddenly life is more simple. You want to know the total IOPS and throughput? It’s all in one place. You want to calculate the ratio of reads to writes? Just compare the read and write columns. Happy days.[^3]

One word of warning though: there are other database processes driving I/O which may not be tracked in these statistics. I see no evidence for control file reads and writes being shown, although these are insignificant in magnitude. More significant would be I/O from the archiver process for databases running in archive log mode, as each redo log must be sequentially read and re-written out as an archive log. Are these included? Yet another possibility would be the Recovery Writer (RVWR) process which is responsible for writing flashback logs when database flashback logging is enabled. [Discussions with Jonathan Lewis suggest these stats are all included – and let’s face it, he wrote the book on the subject…!]  It all adds up… Oracle really needs to provide better clarity on what these statistics are measuring.

## File IO Stats
In 11.2.0.4 or later, new stats "1-bk Rds/s" and "Av 1-bk Rd(ms)" appear in "Tablespace IO Stats" and "File IO Stats" in AWR report.
What is the difference between the old stats "Av Rds/s", "Av Rd(ms)" and the new stats "1-bk Rds/s" and "Av 1-bk Rd(ms)"?

The following columns are the combined statistics for both single block reads and multi block reads:
``` text
Av Rds/s : Number of physical reads per second
Av Rd(ms) : Average read time of physical read
```
The following columns are related to single block reads only:

``` text
1-bk Rds/s : Number of single block reads per second
Av 1-bk Rd(ms) : Average read time of single block read
```
"Av Rd(ms)" is obtained by computing READTIM/PHYRDS.
"Av 1-bk Rd(ms)" is obtained by computing SINGLEBLKRDTIM/SINGLEBLKRDS

All the stats above come from the columns of DBA_HIST_FILESTATXS:

``` text
PHYRDS : Number of physical reads
READTIM : Cumulative physical read time (in hundredths of a second)
SINGLEBLKRDS : Number of single block reads
SINGLEBLKRDTIM : Cumulative single block read time (in hundredths of a second)
```

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
set serveroutput off/on

# snapshot
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

# Execution Plan[^1]
/*+ GATHER_PLAN_STATISTICS */

alter session set statistics_level='ALL';

/execute sql/
SELECT * FROM table(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'ALLSTATS LAST ALL +OUTLINE'));


SELECT * FROM TABLE(DBMS_XPLAN.display_cursor(sql_id=>'9c53z6p4t6ddt', format=>'ALLSTATS LAST +outline +PEEKED_BINDS'));

select * from table(dbms_xplan.display_awr(sql_id=>'48u49a2dxufrm', format=>'ALLSTATS LAST +outline +PEEKED_BINDS'));



# oracle patch
## 查看补丁列表
`/cmcc/app/oracle/product/12.1.0/OPatch/opatch lsinventory`

# 常用SQL
## 获取表上所有索引的DDL
set long 9999
`select dbms_metadata.get_ddl('INDEX',index_name) from user_indexes where table_name=upper('wfp_approval_organization');`

## sql id
select sql_id,username from DBA_HIST_ACTIVE_SESS_HISTORY d inner join dba_users u on u.user_id=d.user_id  where sql_id='6wt2v08kfrf0p';


## kill session
SELECT 'ALTER SYSTEM KILL SESSION '''||sid||','||serial#||''' IMMEDIATE;' FROM v$session where username= '';

rac - ALTER SYSTEM KILL SESSION 'sid,serial#,@inst_id' immediate;  // GV$SESSION

## cmcc
select owner,segment_name,blocks,bytes/1024/1024 from dba_segments where segment_name like 'WF%' order by blocks;

## ASH
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

node6 dbid: 3933062106


## shared pool
``` sql
SELECT substr(sql_text,1,800) "Stmt", count(*),
 sum(sharable_mem)/1024/1024 "Mem",
 sum(users_opening) "Open",
 sum(executions) "Exec"
 FROM v$sql
 GROUP BY substr(sql_text,1,800)
 HAVING sum(sharable_mem) > (select current_size*0.01 from v$sga_dynamic_components where component='shared pool');
```

For SQL statements which are identical but are not being shared, query the V$SQL_SHARED_CURSOR view to determine why the cursors are not shared.

## oracle 实例状态
`select status from v$instances`
mount / open
mount一般会比较快，open需要时间，手动open：`alter database open`

## 编辑临时表空间
https://dbaclass.com/article/how-to-drop-and-recreate-temp-tablespace-in-oracle/

## plsql

``` plsql
begin
  for i in 1..3
    loop
      for rec in (select * from testlikeindex)
        loop
          DBMS_OUTPUT.PUT_LINE(''||rec.OWNER);
        end loop;
    end loop;
end;
```
# oracke management
## autotask
``` plsql
SQL> select task_name,status from dba_autotask_task;

TASK_NAME                                                        STATUS
---------------------------------------------------------------- --------
gather_stats_prog                                                ENABLED
auto_space_advisor_prog                                          ENABLED
AUTO_SQL_TUNING_PROG                                             ENABLED
```

# data block dump[^4]
number col: `select UTL_RAW.CAST_TO_NUMBER( 'c303283f' ) from dual;`
SELECT utl_raw.cast_to_number(replace('c2 4a 46',' ')) value FROM dual;
SELECT utl_raw.cast_to_varchar2(replace('53 4d 49 54 48',' ')) value FROM dual;

# oracle docker
$ docker run -d -it --name oracle --network host store/oracle/database-enterprise:12.2.0.1
$ docker exec -it oracle bash -c "source /home/oracle/.bashrc; sqlplus /nolog"
$ sqlplus sys/Oradoc_db1@ORCLCDB as sysdba


[^1]: [https://blogs.oracle.com/optimizer/how-to-generate-a-useful-sql-execution-plan](How to Generate a Useful SQL Execution Plan)
[^2]:[What Elapsed Time, DB Time and DB CPU represent in AWR report and how to calculate Wait time (Non-Idle)](http://deepakbhatnagardba.blogspot.com/2015/12/what-elapsed-time-db-time-and-db-cpu.html)

[^3]: [Oracle AWR Reports: Understanding I/O Statistics](https://flashdba.com/2014/02/26/oracle-awr-reports-understanding-io-statistics/)

[^4]: [Oracle Data block dump](https://www.orafaq.com/wiki/Data_block)
