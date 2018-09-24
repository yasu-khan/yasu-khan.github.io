---
layout: post
title: Oracle SQL Plan Directive - (Part 2)
description: Oracle 12c SQL Plan Directive insights - Part 2
comments: true
keywords: Oracle, SPD, Oracle, SQL Plan Directive, Result Cache, Result Cache RC Latch, DS_SVC
---

In previous [article](https://yasu-khan.github.io/SQL-Plan-Directives-(Part-1)) we went through basics of SPD (SQL Plan Directive). In this article we will go through each topic which are very important when we start using SPD in production databases. 

## Table of Contents:

1. [Result Cache](#result-cache)
2. [Dynamic Sampling related sql's time threshold](#dynamic-sampling-related-sqls-time-threshold)
3. [Dynamic sampling due to SPD will be always at level=11(AUTO)](#Dynamic-sampling-due-to-SPD-will-be-always-at-level-11)
4. [Consequences of disabling SPD](#Consequences-of-disabling-SPD)
5. [Finding directive ID used by sql](#Finding-directive-ID-used-by-sql)
6. [Conclusion](#Conclusion)

## <a name="result-cache"></a>1. Result Cache

By default when we use optimizer_adaptive_features then SPD will gets enabled and directives will be created automatically whenever it deems fit to resolve cardinality misestimation. So either it may go for Dynamic Statistics or provide information to dbms_stats for creating Extended Statistics to resolve issue permanently, and if Extended Statistics were of no use in resolving cardinality misestimation then SPD will be the only to resolve it and hence SPD internal state will be ```PERMANENT```. 

One of the main strength of SPD is Dynamic Statistics(renamed from Dynamic Sampling in 11g) and to imporve the performance of Dynamic Statistics along with the goal of reducing parse time SPD will use Result Cache to store the result set of Dynamic Statistics sql's so that any other sql's using SPD can take advantage of it. If we trace the sql which is using SPD we will find Dynamic Statistics related sql's as shown below. 

{% highlight sql %}
DS Query Text:
SELECT
       /* DS_SVC */
       /*+ dynamic_sampling(0) no_sql_tune no_monitoring optimizer_features_enable(default) no_parallel result_cache(snapshot=3600) */
       SUM (1)
FROM   (
         SELECT
                /*+ qb_name("innerQuery") NO_INDEX_FFS( "CUSTOMERS")  */
                1 AS c1
         FROM   "SH"."CUSTOMERS" sample BLOCK (51.5796, 8) seed (1) "CUSTOMERS"
         WHERE  ("CUSTOMERS"."CUST_STATE_PROVINCE" = :B1 )
         AND    ("CUSTOMERS"."COUNTRY_ID" = :B2 )
       ) innerquery
/

{% endhighlight %}

Queries having ```'DS_SVC'``` text are all related to Dynamic Statistics and hint ```'result_cache(snapshot=3600)'``` is the one which enables Dynamic Statistics sql's to use and store their result set in Result Cache. These results will be expired after 3600 seconds and are not dependant which means no matter what the data in result cache will not be obsolete for 3600 seconds. Also these sql's will run without parallelism and monitoring. If we check the default parameters related to result cache then in Enterprise Edition result_cache_mode will be manual(Populate result cache through hints) and ```result_cache_max_size``` will be derived from the values of ```SHARED_POOL_SIZE```, ```SGA_TARGET```, and ```MEMORY_TARGET``` if it is set. 

{% highlight sql %}
NAME                                 VALUE
------------------------------------ ------------------------------
result_cache_max_result              5
result_cache_max_size                10496K
result_cache_mode                    MANUAL
result_cache_remote_expiration       0
{% endhighlight %}

To check the usage of Result Cache we can use ```DBMS_RESULT_CACHE``` package and try to find if currently allocated memory for Result cache is sufficient or not. 
{% highlight sql %}
SQL> set serveroutput on
SQL> EXEC DBMS_RESULT_CACHE.MEMORY_REPORT
R e s u l t   C a c h e   M e m o r y   R e p o r t
[Parameters]
Block Size          = 1K bytes
Maximum Cache Size  = 10496K bytes (10496 blocks)
Maximum Result Size = 524K bytes (524 blocks)
[Memory]
Total Memory = 379440 bytes [0.009% of the Shared Pool]
... Fixed Memory = 25208 bytes [0.001% of the Shared Pool]
... Dynamic Memory = 354232 bytes [0.008% of the Shared Pool]
....... Overhead = 124856 bytes
....... Cache Memory = 224K bytes (224 blocks)
........... Unused Memory = 5 blocks
........... Used Memory = 219 blocks
............... Dependencies = 92 blocks (92 count)
............... Results = 127 blocks
................... CDB     = 102 blocks (101 count)
................... Invalid = 25 blocks (25 count)

PL/SQL procedure successfully completed.

{% endhighlight %}

Above Result cache usage report is from 12c sandbox I am having at the moment in which activity is very minimal, but in case of production environment its very important to keep in mind about Result cache configuration and monitor it since all the dictionary sql's will also be using SPD. If you check the number of directives on schema owned objects then you will see that SYS schema is the one which is having highest number of directives.

{% highlight sql %}
SQL> select owner,count(*) from DBA_SQL_PLAN_DIR_OBJECTS group by owner;

OWNER        COUNT(*)
---------- ----------
CTXSYS              1
DBSNMP              4
XDB                 6
SYS               991

SQL> select owner,OBJECT_NAME,count(*) from DBA_SQL_PLAN_DIR_OBJECTS group by owner,OBJECT_NAME having count(*) > 20 order by 3;

OWNER      OBJECT_NAME                      COUNT(*)
---------- ------------------------------ ----------
SYS        WRI$_ADV_FINDINGS                      21
SYS        X$KSPPI                                22
SYS        WRI$_ADV_TASKS                         24
SYS        WRH$_ACTIVE_SESSION_HISTORY            27
SYS        TS$                                    28
SYS        TAB$                                   29
SYS        USER$                                  69
SYS        OBJ$                                  130
{% endhighlight %}

If application is utilizing huge number of directives and if Result cache is undersized/left at default then sessions may wait on ```"Result Cache: RC Latch"``` due to contention. As of now even in 12.1.0.2 there is no documented parameter to disable this behaviour. Its better to be aware of this behaviour and take necessary actions to accomodate dynamic samplong caused due to SPD. Also its important to closely monitor shared spool resize activity due to demand in result cache.

## <a name="dynamic-sampling-related-sqls-time-threshold"></a>2. Dynamic Sampling related SQL's time threshold

Dynamic sampling phase cannot run indefinitely and it has time threshold which it can't cross. This time limit is calculated based on whether sql is in cache or AWR, if not present at both places then it wil use default threshold of 10 seconds. If sql is present in cache/AWR then threshold will be calculated based on CPU consumption and number of executions. This calculation can be seen by enabling trace of the sql using SPD.

Below sql is used for calculating time threshold if query is in cache:
{% highlight sql %}
SELECT executions,
       end_of_fetch_count,
       elapsed_time / px_servers elapsed_time,
       cpu_time / px_servers     cpu_time,
       buffer_gets / executions  buffer_gets
FROM   ( SELECT SUM( executions )         AS executions,
                SUM( CASE
                       WHEN px_servers_executions > 0 THEN px_servers_executions
                       ELSE executions
                     END )                AS px_servers,
                SUM( end_of_fetch_count ) AS end_of_fetch_count,
                SUM( elapsed_time )       AS elapsed_time,
                SUM( cpu_time )           AS cpu_time,
                SUM( buffer_gets )        AS buffer_gets
         FROM   GV$SQL
         WHERE  executions > 0 AND
                sql_id = :1 AND
                parsing_schema_name = :2 ) 
/
{% endhighlight %}

Below sql is used for calculating time threshold if query is in AWR:
{% highlight sql %}
SELECT executions,
       end_of_fetch_count,
       elapsed_time / px_servers elapsed_time,
       cpu_time / px_servers     cpu_time,
       buffer_gets / executions  buffer_gets
FROM   ( SELECT SUM( executions_delta )         AS EXECUTIONS,
                SUM( CASE
                       WHEN px_servers_execs_delta > 0 THEN px_servers_execs_delta
                       ELSE executions_delta
                     END )                      AS px_servers,
                SUM( end_of_fetch_count_delta ) AS end_of_fetch_count,
                SUM( elapsed_time_delta )       AS ELAPSED_TIME,
                SUM( cpu_time_delta )           AS CPU_TIME,
                SUM( buffer_gets_delta )        AS BUFFER_GETS
         FROM   DBA_HIST_SQLSTAT s,
                V$DATABASE d,
                DBA_HIST_SNAPSHOT sn
         WHERE  s.dbid = d.dbid AND
                Bitand( Nvl( s.flag, 0 ), 1 ) = 0 AND
                sn.end_interval_time > ( SELECT systimestamp AT TIME zone dbtimezone
                                         FROM   DUAL ) - 7 AND
                s.sql_id = :1 AND
                s.snap_id = sn.snap_id AND
                s.instance_number = sn.instance_number AND
                s.dbid = sn.dbid AND
                parsing_schema_name = :2 ) 
/
{% endhighlight %}

If you enable 10053 trace then you can see the detail information regarding time threshold getting calculated as shown below
```
kkoadsTimeLimitFromSrc(Enter) exeStSrc=CC
kkoadsTimeLimitFromSrc(Exit) timeLimit=0
kkoadsTimeLimitFromSrc(Enter) exeStSrc=AWR
kkoadsTimeLimitFromSrc(Exit) timeLimit=0
kkoadsTimeLimit: source=Voodoo timeLimit=10
```

In this case sql was not found in both AWR and Cursor cache and thus time limit has been set to default 10 seconds.

Though there is a mechanism in place for limiting dynamic sampling sql's to not cross the time threshold defined, you may face ```"ORA-10173: Dynamic Sampling time-out"``` error in alert log file. We can confirm that sql's facing error ```ORA-10173``` is related to dynamic sampling by looking at the ```'DS_SVC'``` text in the sql. To get more detail information on this error we can set error trap as following ```"alter <session/system> set events '10173 trace name errorstack level 1';"```, but better to trace at lower level first and then if results are not satisfactory go for system level error trap.

## <a name="Dynamic-sampling-due-to-SPD-will-be-always-at-level-11"></a>3. Dynamic sampling due to SPD will be always at level=11(AUTO)

In 12.1.0.2 release when dynamic sampling is performed due to SPD then dynamic sampling level will be always at 11(AUTO). But if you check the dynamic sampling level specified through ```dbms_xplan.display_cursor``` it always provides the default level defined in parameter ```optimizer_dynamic_sampling``` which is not true. This can be confirmed by performing trace of sql at ```RDBMS.SQL_DS``` component along with 10053 trace.

Below is the snippet of output from trace file.
```
=======================================
SPD: BEGIN context at query block level
=======================================
Query Block SEL$1 (#0)
Applicable DS directives:
   dirid = 13428679272123285773, state = 5, flags = 1, loc = 1 {EC(94258)[9, 11]}
Checking valid directives for the query block
  SPD: Return code in qosdDSDirSetup: NODIR, estType = QUERY_BLOCK
Return code in qosdSetupDirCtx4QB: EXISTS
=====================================
SPD: END context at query block level
=====================================
Access path analysis for CUSTOMERS3
***************************************
SINGLE TABLE ACCESS PATH
  Single Table Cardinality Estimation for CUSTOMERS3[CUSTOMERS3]
  SPD: Directive valid: dirid = 13428679272123285773, state = 5, flags = 1, loc = 1 {EC(94258)[9, 11]}
  SPD: Return code in qosdDSDirSetup: EXISTS, estType = TABLE
  Column (#9):
    NewDensity:0.001109, OldDensity:0.001312 BktCnt:5399.000000, PopBktCnt:2016.000000, PopValCnt:55, NDV:620
  Column (#9): CUST_CITY(VARCHAR2)
    AvgLen: 10 NDV: 620 Nulls: 0 Density: 0.001109
    Histogram: Hybrid  #Bkts: 254  UncompBkts: 5399  EndPtVals: 254  ActualVal: yes
  Column (#11):
    NewDensity:0.000144, OldDensity:0.000009 BktCnt:55500.000000, PopBktCnt:55500.000000, PopValCnt:145, NDV:145
  Column (#11): CUST_STATE_PROVINCE(VARCHAR2)
    AvgLen: 11 NDV: 145 Nulls: 0 Density: 0.000144
    Histogram: Freq  #Bkts: 145  UncompBkts: 55500  EndPtVals: 145  ActualVal: yes

  Table: CUSTOMERS3  Alias: CUSTOMERS3
    Card: Original: 55500.000000qksdsSInitCtx(): qksdsSInitCtx(): timeLimit(ms) = 2500
qksdsCheckPreds():   qksdsCheckPreds(exit): total count=2 usable count=2
qksdsExecute(): qksdsExecute(): enter
qksdsExeStmt():   qksdsExeStmt(): enter
qksdsExeStmt(): ************************************************************
DS Query Text:
SELECT /* DS_SVC */ /*+ dynamic_sampling(0) no_sql_tune no_monitoring optimizer_features_enable(default) no_parallel result_cache(snapshot=3600) */ SUM(C1) FROM (SELECT /*+ qb_name("innerQuery") NO_INDEX_FFS( "CUSTOMERS3")  */ 1 AS C1 FROM "SH"."CUSTOMERS3" SAMPLE BLOCK(51.6129, 8) SEED(1)  "CUSTOMERS3" WHERE ("CUSTOMERS3"."CUST_CITY"='Los Angeles') AND ("CUSTOMERS3"."CUST_STATE_PROVINCE"='CA')) innerQuery
qksdsExeStmt():

qksdsExeStmt(): timeInt = 2 timeLimit = 2 elapTime = 0
**************************************************************
Iteration 1
  Exec count:         1
  CR gets:            797
  CU gets:            0
  Disk Reads:         0
  Disk Writes:        0
  IO Read Requests:   0
  IO Write Requests:  0
  Bytes Read:         0
  Bytes Written:      0
  Bytes Exchanged with Storage:  0
  Bytes Exchanged with Disk:  0
  Bytes Simulated Read:  0
  Bytes Simulated Returned:  0
  Elapsed Time: 5656 (us)
  CPU Time: 5999 (us)
  User I/O Time: 0 (us)
qksdsDumpEStats(): Sampling Input
  IO Size:      8
  Sample Size:  51.612903
  Post S. Size: 100.000000

qksdsExeStmt():   qksdsExeStmt: exit
qksdsDumpStats(): **************************************************************
DS Service Statistics
qksdsDumpStats():   Executions:  1
  Retries:     0
  Timeouts:    0
  ParseFails:  0
  ExecFails:   0
qksdsDumpStats(): qksdsDumpResult(): DS Results: #exps=1, smp obj=CUSTOMERS3
qksdsDumpResult():    T.CARD = qksdsDumpResult(): (mid=924.2, low=924.2, hig=924.2)qksdsDumpResult():
qksdsDumpResult(): end dumping resultsqksdsExecute(): qksdsExecute(): exit
    >> Single Tab Card adjusted from 51.361919 to 924.187500 due to adaptive dynamic sampling
  Rounded: 924  Computed: 924.187500  Non Adjusted: 51.361919
  Scan IO  Cost (Disk) =   422.000000
  Scan CPU Cost (Disk) =   30463232.000000
  Total Scan IO  Cost  =   422.000000 (scan (Disk))
                         + 0.000000 (io filter eval) (= 0.000000 (per row) * 55500.000000 (#rows))
                       =   422.000000
  Total Scan CPU  Cost =   30463232.000000 (scan (Disk))
                         + 2817660.677903 (cpu filter eval) (= 50.768661 (per row) * 55500.000000 (#rows))
                       =   33280892.677903
  Access Path: TableScan
    Cost:  423.056801  Resp: 423.056801  Degree: 0
      Cost_io: 422.000000  Cost_cpu: 33280893
      Resp_io: 422.000000  Resp_cpu: 33280893
  Best:: AccessPath: TableScan
         Cost: 423.056801  Degree: 1  Resp: 423.056801  Card: 924.187500  Bytes: 0.000000

***************************************
```

If you check the text within ```'SPD: BEGIN'``` and ```'SPD: END'``` it verifies the applicable directives and then verifies the valid directives after which it select directive ```13428679272123285773``` having ```state=5(PERMANENT)```. Then it enter into ```'Access path analysis for CUSTOMERS3'``` stage where it performs dynamic sampling with the help of ```/* DS_SVC */``` hinted sql having ```'SAMPLE BLOCK(51.6129, 8)'``` which means it is performin 51 % of block samples from ```CUSTOMERS3``` table having total of 1550 blocks and thus got 797 ```'CR gets' in 'qksdsExeStmt():'``` section. At the end single table cardinality will be adjusted from 51.361919 to 924.187500 due to adaptive dynamic sampling. Please note that to make sure ```'CR gets'``` metric gets populated I flushed the Result cache before tracing.

According to dynamic sampling level 2 it should sample only 64 blocks which is not the case here, hence we can confirm that due to sampling of 797 blocks it is adaptive dynamic sampling performed at level 11.

But if you check the ```NOTE``` section from ```dbms_xplan.display_cursor``` you will find that Dynamic Samping has been done at level=2 which is NOT true because ```dbms_xplan.display_cursor``` pulls the information from ```V$SQL_PLAN.OTHER_XML``` and dynamic sampling level information in ```OTHER_XML``` column gets populated during parse phase but not the one actually used by query optimizer.

{% highlight sql %}
SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------
SQL_ID  3cumb92a8txbm, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*)   FROM   sh.customers3
WHERE  cust_city='Los Angeles'   AND    cust_state_province='CA'

Plan hash value: 178603127

-------------------------------------------------------------------------------------------
| Id  | Operation          | Name       | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |            |      1 |        |      1 |00:00:00.01 |    1521 |
|   1 |  SORT AGGREGATE    |            |      1 |      1 |      1 |00:00:00.01 |    1521 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS3 |      1 |    924 |    932 |00:00:00.01 |    1521 |
-------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - 1 Sql Plan Directive used for this statement
{% endhighlight %}
   
## <a name="Consequences-of-disabling-SPD"></a>4. Consequences of disabling SPD 

At certain situations SPD may cause adverse impact on the sql's due to overhead of dynamic sampling and thus we may require to disable the SPD. Disabling SPD is straight forward process, we can disable directive by using
```exec dbms_spd.alter_sql_plan_directive(<Directive_ID>,'ENABLED','NO');```

Even we can drop the directive but if you drop it then directive can reappear again due to the same previous reason. Also as of now there is no documented option to disable creation of directive on particular table beforehand. With these precuations in mind we need to be careful that disabled directives doesn't gets dropped automatically. Directives will be dropped automatically if ```AUTO_DROP``` is set to YES and ```LAST_USAGE``` exceeds the retention weeks(default 53 weeks). Thus to disable the directive permanently we need to ensure that directive doesn't gets dropped automatically and reappears by setting ```AUTO_DROP``` to NO along with ```ENABLED``` set to NO.
{% highlight sql %}
exec dbms_spd.alter_sql_plan_directive(<Directive_ID>,'AUTO_DROP','NO');
exec dbms_spd.alter_sql_plan_directive(<Directive_ID>,'ENABLED','NO');
{% endhighlight %}

Since ```LAST_USED``` property is used only for dropping directive automatically, frequency of updating ```LAST_USED``` is for 6.5 days. So do not rely on directive ```LAST_USED``` column in ```DBA_SQL_PLAN_DIRECTIVES``` view for finding recent used date of the directive.

There is a caveat which we need to be aware of before planning to disable the directive. Basically directive influence either to do Dynamic Sampling or to create Extended Statistics, if your intention is to just disable dynamic sampling caused due directive then set ```AUTO_DROP``` and ```ENABLED``` to NO. But if your intention is to avoid creating extended statistics due to directives then disabling directive will not prevent creation of extended statistics.

Demo to prove disabled directive will influence creation of extended statistics.

State before disabling directive
{% highlight sql %}
         DIRECTIVE_ID STATE  LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE
--------------------- ------ ------------------------------- --- --- -------------------------------------------------- ---------------
 14887790121833036011 USABLE 10-DEC-15 05.54.37.000000000 AM YES YES {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} MISSING_STATS

SQL> exec dbms_spd.alter_sql_plan_directive(14887790121833036011,'AUTO_DROP','NO');

PL/SQL procedure successfully completed.

SQL> exec dbms_spd.alter_sql_plan_directive(14887790121833036011,'ENABLED','NO');

PL/SQL procedure successfully completed.
{% endhighlight %}

State after disabling directive 
{% highlight sql %}
         DIRECTIVE_ID STATE  LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE
--------------------- ------ ------------------------------- --- --- -------------------------------------------------- ---------------
 14887790121833036011 USABLE 10-DEC-15 05.54.37.000000000 AM NO  NO  {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} MISSING_STATS

SQL> exec dbms_stats.gather_table_stats('SH','CUSTOMERS');

PL/SQL procedure successfully completed.

SQL> select * from dba_stat_extensions where OWNER='SH' and TABLE_NAME='CUSTOMERS4';

OWNER TABLE_NAME EXTENSION_NAME                 EXTENSION                           CREATO DRO
----- ---------- ------------------------------ ----------------------------------- ------ ---
SH    CUSTOMERS  SYS_STSWMBUN3F$#398R7BS0YVS86R ("CUST_CITY","CUST_STATE_PROVINCE") SYSTEM YES
{% endhighlight %} 

## <a name="Finding-directive-ID-used-by-sql"></a>5. Finding directive ID used by sql

At times when tuning sql we may need to find the directive id used by particular sql_id, but since directive id details are not stored into the cursor it becomes challenging to find it. One of the easiest way to find is by doing 'explain plan for' to the sql and then reporting its plan with +metrics option as shown below.

{% highlight sql %}
SQL> explain plan for
  2  SELECT count(*)
  3  FROM   sh.customers
  4  WHERE  cust_city='Los Angeles'
  5  AND    cust_state_province='CA';

Explained.

SQL> select * from table(dbms_xplan.display(null,null,'+metrics'));

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
Plan hash value: 296924608

--------------------------------------------------------------------------------
| Id  | Operation          | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |     1 |    21 |   423   (1)| 00:00:01 |
|   1 |  SORT AGGREGATE    |           |     1 |    21 |            |          |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |   952 | 19992 |   423   (1)| 00:00:01 |
--------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA')

Sql Plan Directive information:
-------------------------------

  Used directive ids:
    12105355473441073000

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - 1 Sql Plan Directive used for this statement
{% endhighlight %} 

But ```'explain plan for'``` method has its own caveats lke - Bind peeking doesn't happens, All bind variables are considered as VARCHAR2 regardless of how we define it, so we need to make sure we use to_number or literals and to_date to get the right type conveyed to explain plan. So in reality using 'explain plan for' is always conundrum.

There is one more way to circumvent this issue by enabling 10053 for the sql, but you need to parse the sql statement to get 10053 trace. If sql is existing in cache then there is a easiest way to enable 10053 and get the drective id used by the sql by using ```DBMS_SQLDIAG``` package, this package will run the sql to hard parse it and also if sql is using any bind variables then it will fetch the bind values used when parsing the sql and execute it by adding in a comment as ```/* SQL Analyze(1443,0) */```.

Say for example sql_id 0ch70x7cfqfvc is existing in cursor cache and I want to find the directive id used by this sql.
{% highlight sql %}
SELECT sql_id,
       child_number,
       Regexp_substr ( dbms_lob.Substr( other_xml, 4000 ), '<(cu)>([^<]+)</\1>', 1, 1, NULL, 2 ) AS "SPD_Used"
FROM   V$SQL_PLAN
WHERE  sql_id = '0ch70x7cfqfvc' 

/

SQL_ID        CHILD_NUMBER SPD_Used
------------- ------------ ----------
0ch70x7cfqfvc            0 1
{% endhighlight %}

Enable 10053 trace by using - 
{% highlight sql %}
exec dbms_sqldiag.dump_trace(p_sql_id=>'0ch70x7cfqfvc',p_child_number=>0,p_component=>'Compiler',p_file_id=>'FIND_SPD_ID_TRACE');
{% endhighlight %}

Below is the snippet of trace file.
```
=====================================
SPD: BEGIN context at statement level
=====================================
Stmt: ******* UNPARSED QUERY IS *******
SELECT COUNT(*) "COUNT(*)" FROM "SH"."CUSTOMERS" "CUSTOMERS" WHERE "CUSTOMERS"."CUST_CITY"='Los Angeles' AND "CUSTOMERS"."CUST_STATE_PROVINCE"='CA'
Objects referenced in the statement
  CUSTOMERS[CUSTOMERS] 93246, type = 1
Objects in the hash table
  Hash table Object 93246, type = 1, ownerid = 12409857813664911764:
    Dynamic Sampling Directives at location 1:
       dirid = 12105355473441073000, state = 2, flags = 1, loc = 1 {EC(93246)[9, 11]}
Return code in qosdInitDirCtx: ENBLD
===================================
SPD: END context at statement level
===================================
```
So directive id used by sql_id ```0ch70x7cfqfvc``` is ```12105355473441073000```. If we want complete details of dynamic sampling along with directive id then we need to enable trace at ```RDBMS.SQL_DS``` as well.

## <a name="Conclusion"></a>6. Conclusion
In this article we went through some of the important concepts behind SPD along with some precautions and tricks which can be handy when we tackle issues related to SPD in production databases. There can be many more unknown facts related to SPD but this feature does really helps the Optimizer. In brief DBA's has to closely monitor SPD usage and judge its importance to critical sql's in the database.

