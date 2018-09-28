---
layout: post
title: Oracle 12c - History reports of Real Time SQL Monitoring
description: Oracle 12c has capability to store history information of Real Time SQL Monitoring
comments: true
keywords: Oracle, performance, Real Time SQL Monitoring, DBMS_AUTO_REPORT, DBA_HIST_REPORTS_DETAILS
---

Introduction
--------
Real-Time SQL Monitoring in 12c has been enhanced tremendously, as in earlier releases this feature was meant to monitor only at sql level but in 12c it has been improved to monitor multiple sql and pl/sql in batch called as Database Operations. One of the biggest enhancement in 12c is that now Real-Time SQL Monitoring reports are stored in AWR, which means now we can easily navigate through sql performance history by viewing Real-Time SQL Monitoring reports accross previous point in time according to AWR retention policy.

AWR in 12c has been enhanced to store reports of Real-Time SQL Monitoring, Real-Time ADDM and Database Operations Monitoring. But there is no straight way to access Real-Time SQL Monitoring reports as all of these reports has been merged into ```DBA_HIST_REPORTS``` and ```DBA_HIST_REPORTS_DETAILS``` which is in fact the framework of Real-Time ADDM. Mechanics of capturing these reports are available in ```DBA_HIST_REPORTS_CONTROL``` which says mode of execution of automatic report capture can be either ```REGULAR```(report capture subject to DBTIME budget) or ```FULL_CAPTURE```(report capture without DBTIME budget). Also in 12c ```REPORT_SQL_MONITOR``` has been moved from ```DBMS_SQLTUNE``` to DBMS_SQL_MONITOR package.

Real-Time SQL Monitoring - Batch mode
-------------------------------------
```DBMS_SQL_MONITOR``` has begin and end procedures to start and end the monitoring of database opeations grouped logically into a single operation. This will be helpful in finding the reason for batch job slowness and then find which sql has consumed most of the response time. For example I can enable monitoring for a batch job which creates the table, alter the table and then select from it using pl/sql block.

```sql
SQL> variable exec_id number;
SQL> begin
  2    :exec_id := dbms_sql_monitor.begin_operation ( dbop_name => 'DBOP10', forced_tracking => dbms_sql_monitor.force_tracking );
  3  end;
  4  /

PL/SQL procedure successfully completed.
```

I have enabled forced_tracking to override the default behaviour of tracking if it has consumed 5 seconds of CPU or I/O time.
```sql
SQL> create table tab1 as
  2  select
  3      rownum  id,
  4      rpad(rownum,10) v1,
  5      rpad('x',100)   v2
  6  from
  7      all_objects
  8  where
  9      rownum <= 10000
 10  /

Table created.

SQL> alter table tab1 add constraint tab1_pk primary key(id);

Table altered.

SQL> begin
  2    for x in (select /*+ index(tab1) */ v1 from tab1 where id > 0 )
  3    loop
  4  dbms_lock.sleep(0.01);
  5    end loop;
  6  end;
  7  /

PL/SQL procedure successfully completed.

SQL> begin
  2    dbms_sql_monitor.end_operation ( dbop_name => 'DBOP10', dbop_eid => :exec_id );
  3  end;
  4  /

PL/SQL procedure successfully completed.

SQL> select STATUS,SQL_ID,DBOP_NAME,DBOP_EXEC_ID,CON_NAME,ELAPSED_TIME from v$sql_monitor;

STATUS              SQL_ID        DBOP_NAME                      DBOP_EXEC_ID CON_NAME                       ELAPSED_TIME
------------------- ------------- ------------------------------ ------------ ------------------------------ ------------
DONE                0000000000000 DBOP10                                    5 PDBT2                               1328405

SQL> SELECT
  2     DBMS_SQL_MONITOR.report_sql_monitor( dbop_name => 'DBOP10', type => 'TEXT', report_level => 'ALL') AS report
  3  FROM dual
  4  /
SQL Monitoring Report

Global Information
------------------------------
 Status              :  DONE
 Instance ID         :  2
 Session             :  TEST (2327:20905)
 DBOP Name           :  DBOP10
 DBOP Execution ID   :  5
 First Refresh Time  :  09/15/2015 13:35:17
 Last Refresh Time   :  09/15/2015 13:40:19
 Duration            :  302s
 Module/Action       :  SQL*Plus/-
 Service             :  pdbt2
 Program             :  sqlplus@dbalabserver (TNS V1-V3)

Global Stats
==========================================================================================================================
| Elapsed |   Cpu   |    IO    | Application | Concurrency | Cluster  |  Other   | Buffer | Read | Read  | Write | Write |
| Time(s) | Time(s) | Waits(s) |  Waits(s)   |  Waits(s)   | Waits(s) | Waits(s) |  Gets  | Reqs | Bytes | Reqs  | Bytes |
==========================================================================================================================
|    1.33 |    1.03 |     0.19 |        0.00 |        0.01 |     0.03 |     0.07 |    204 |   37 |   1MB |     9 |   1MB |
==========================================================================================================================
```

Above report is the consolidated report which includes monitoring for CREATE TABLE, ALTER TABLE and PL/SQL block operations performed by the batch job database operation 'DBOP10'

Real-Time SQL Monitoring - AWR(REGULAR)
---------------------------------------
Real-Time SQL Monitoring reports are stored in AWR in form of XML and it captures every minute only for the sql statements which are not currently executing or queued and have completed execution since the last cycle of capture. Not all the reports are stored in AWR, but only the top 5 sql's which are expensive in terms of sql elapsed time. Since these SQL monitoring reports are part of the Real-Time ADDM framework we need to manually fetch ```REPORT_ID``` by querying ```DBA_HIST_REPORTS``` becasue for generating Real-Time SQL Monitoring reports from AWR we need to have ```REPORT_ID``` which can be used in ```DBMS_AUTO_REPORT``` package.

```sql
SQL> select distinct COMPONENT_NAME from DBA_HIST_REPORTS;

COMPONENT_NAME
-----------------------------------------------------------
sqlmonitor
perf
```

So ```COMPONENT_NAME``` can be used to differentiate Real-Time SQL and ADDM reports stored in AWR. For each of these component the information of report is stored in ```REPORT_SUMMARY``` column in XML format, and complete report is stored in ```DBA_HIST_REPORTS_DETAILS``` view. To make it easier we can fetch sql level details from this ```REPORT_SUMMARY``` XML column and get the corresponding ```REPORT_ID``` for generating complete Real-Time SQL Monitoring report using ```DBMS_AUTO_REPORT```. 

Lets take an random sample XML output from ```REPORT_SUMMARY``` column of ```DBA_HIST_REPORTS``` and build an XML query to fetch all the sql details oriented in the form of columns.

```
SQL> select xmlserialize(CONTENT xmltype(report_summary) as CLOB indent size=1) PRETTY_XML from dba_hist_reports where report_id=5182;

PRETTY_XML
------------------------------------------------------------------------------------------
<report_repository_summary>
 <sql sql_id="agmfaj1f82k5h" sql_exec_start="08/28/2015 21:05:07" sql_exec_id="16777216">
  <status>DONE (ALL ROWS)</status>
  <sql_text>
SELECT  /*+first rows */
pdb.name,
d.tablespace_name,
NVL(a.bytes / :&quot;SYS_B_00&quot; / :&quot;SYS_B_01&quot;, :&quot;SY</sql_text>
  <first_refresh_time>08/28/2015 21:05:07</first_refresh_time>
  <last_refresh_time>08/28/2015 21:09:56</last_refresh_time>
  <refresh_count>179</refresh_count>
  <inst_id>1</inst_id>
  <session_id>62</session_id>
  <session_serial>11014</session_serial>
  <user_id>48</user_id>
  <user>DBSNMP</user>
  <con_id>1</con_id>
  <con_name>CDB$ROOT</con_name>
  <module>emagent_SQL_rac_database</module>
  <action>tbspAllocation_cdb</action>
  <service>SYS$USERS</service>
  <program>JDBC Thin Client</program>
  <plan_hash>2732199398</plan_hash>
  <is_cross_instance>Y</is_cross_instance>
  <dop>3</dop>
  <instances>2</instances>
  <px_servers_requested>8</px_servers_requested>
  <px_servers_allocated>8</px_servers_allocated>
  <stats type="monitor">
   <stat name="duration">289</stat>
   <stat name="elapsed_time">304583924</stat>
   <stat name="cpu_time">42251575</stat>
   <stat name="user_io_wait_time">221517325</stat>
   <stat name="application_wait_time">4211</stat>
   <stat name="concurrency_wait_time">11443</stat>
   <stat name="cluster_wait_time">40799370</stat>
   <stat name="user_fetch_count">2</stat>
   <stat name="buffer_gets">723986</stat>
   <stat name="read_reqs">182749</stat>
   <stat name="read_bytes">1500487680</stat>
  </stats>
 </sql>
</report_repository_summary>

```

As per the above XML output it seems that XML document is structured in a way to store most of the monitored sql details. So building the sql to query each XML nodes is very easy. ```REPORT_SUMMARY``` column has been defined as VARCHAR2(4000) so none of the XML optimizations can be applied by Optimizer, hence its always better to provide hint ```/*+ NO_XML_QUERY_REWRITE */``` whenever we query on this column. Note that I have deliberately selected all the XML nodes from XML report in form of columns so that you can easily select any column based on your requirement criteria and filter the records. 

Let's say for example I want to find all the Real-Time SQL Monotoring reports stored in AWR for the SQL's which has taken more than 200 seconds to execute.

```sql
SELECT /*+ NO_XML_QUERY_REWRITE */ t.report_id, x1.sql_id, x1.plan_hash, x1.sql_exec_id, x1.elapsed_time/1000000 ELAP_SEC
FROM dba_hist_reports t    
   , xmltable('/report_repository_summary/sql'    
       PASSING xmlparse(document t.report_summary)    
       COLUMNS    
         sql_id                path '@sql_id'     
       , sql_exec_start        path '@sql_exec_start'    
       , sql_exec_id           path '@sql_exec_id'      
       , status                path 'status'    
       , sql_text			   path 'sql_text'
       , first_refresh_time    path 'first_refresh_time'
       , last_refresh_time     path 'last_refresh_time'
       , refresh_count         path 'refresh_count'
       , inst_id               path 'inst_id'
       , session_id            path 'session_id'
       , session_serial        path 'session_serial'
       , user_id               path 'user_id'
       , username              path 'user'
       , con_id                path 'con_id'
       , con_name              path 'con_name'
       , modul                 path 'module'
       , action                path 'action'
       , service               path 'service'
       , program               path 'program'
       , plan_hash             path 'plan_hash'
       , is_cross_instance     path 'is_cross_instance'
       , dop				   path 'dop'
       , instances             path 'instances'
       , px_servers_requested  path 'px_servers_requested'
       , px_servers_allocated  path 'px_servers_allocated'
       , duration              path 'stats/stat[@name="duration"]'  
       , elapsed_time          path 'stats/stat[@name="elapsed_time"]'  
       , cpu_time              path 'stats/stat[@name="cpu_time"]'  
       , user_io_wait_time     path 'stats/stat[@name="user_io_wait_time"]'
       , application_wait_time path 'stats/stat[@name="application_wait_time"]'
       , concurrency_wait_time path 'stats/stat[@name="concurrency_wait_time"]'
       , cluster_wait_time     path 'stats/stat[@name="cluster_wait_time"]'
       , plsql_exec_time       path 'stats/stat[@name="plsql_exec_time"]'
       , other_wait_time       path 'stats/stat[@name="other_wait_time"]'
       , buffer_gets           path 'stats/stat[@name="buffer_gets"]'
       , read_reqs             path 'stats/stat[@name="read_reqs"]'
       , read_bytes            path 'stats/stat[@name="read_bytes"]'
     ) x1 
where x1.elapsed_time/1000000 > 200
and   t.COMPONENT_NAME = 'sqlmonitor'
order by 5
/			

 REPORT_ID SQL_ID               PLAN_HASH            SQL_EXEC_ID           ELAP_SEC PX_REQ     PX_ALLOC
---------- -------------------- -------------------- -------------------- --------- ---------- ----------
      7687 3zk1hp8dk2zs6        3736380315           16777220                   215 14         14
      9196 duxwqqg8un28r        0                    16777216                   268
      3344 agmfaj1f82k5h        2732199398           33554432                   295 8          8
      5182 agmfaj1f82k5h        2732199398           16777216                   305 8          8
```

Now use corresponding REPORT_ID to generate complete report stored in AWR for specific SQL.

```sql
SELECT DBMS_AUTO_REPORT.REPORT_REPOSITORY_DETAIL(RID => 5182, TYPE => 'TEXT') FROM DUAL;
```

Real-Time SQL Monitoring - AWR(FULL_CAPTURE)
--------------------------------------------
In some cases we may want to capture all the SQL's Real-Time SQL Monitoring reports and store it in AWR for every minute. In such cases we need to enable ```FULL_CAPURE``` as long as we want and then disable it. 

```
DBMS_AUTO_REPORT.START_REPORT_CAPTURE;
DBMS_AUTO_REPORT.FINISH_REPORT_CAPTURE;
```

After ```FULL_CAPTURE``` finish, every minute capturing of data continues but not for all the SQL's which are monitored but only for those which are considered to top 5 expensive SQL's in terms of elapsed time. But ```FULL_CAPTURE``` will capture all the SQL's whose monitoring has been completed regardless of their elapsed time. In case of RAC it gets enabled/disabled in all the instances. Reports present in AWR can be viewed similar to the way ```REGULAR``` reports are viewed as shown in previous section.


Caveat
------
Even in 12c Real-Time SQL Monitoring reports doesn't contain ```'Predicate Information'``` section, which at times is very crucial information for tuning sql statements.
