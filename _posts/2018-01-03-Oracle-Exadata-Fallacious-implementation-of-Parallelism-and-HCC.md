---
layout: post
title: Oracle Exadata - Fallacious implementation of Parallelism & HCC

description: Oracle Exadata - Fallacious implementation of Parallelism & HCC
comments: true
keywords: Oracle, Exadata, Force Parallelism, HCC, cell single block physical read, update, compression
---

Issue decription
-----------------
In data warehousing database one of the ```MERGE``` statement with dimension and staging table was taking long time and never gets completed even after 4 hours. Also this ```MERGE``` statement was causing the whole database performance down to its knees due to high CPU and other resource usage. Database has been hosted in Exadata commodity and thus expectation were high with the MERGE statement response time. Dimension table was having about 500 Million rows and staging table was having about 8 Million rows. 

```MERGE``` statement in question is as shown below:

```sql
MERGE /*+ PARALLEL(t1,8)*/
INTO    DIMENSION t1
USING   (
          SELECT /*+ PARALLEL(t3,8)*/ *
          FROM   stage t3
        ) t2
ON ( t1.dim_key = t2.dim_key )
WHEN MATCHED THEN
  UPDATE
  SET     t1.col1 = t2.col1,
      t1.col2 = t2.col2,
      t1.col3 = t2.col3,
      t1.col4 = t2.col4,
WHEN NOT MATCHED THEN
  INSERT (
      t1.dim_key,
      t1.col1,
      t1.col2,
      t1.col3,
      t1.col4 )
  VALUES (
      t2.dim_key,
      t2.col1,
      t2.col2,
      t2.col3,
      t2.col4 )
/
```

Troubleshooting / Investigation
--------------------------------
Since ```MERGE``` statement ran for more than 4 hours there are high chances that it might have been captured in the AWR. Thus finding the execution plan along with it's response time break down waiting on events at each operation in the execution plan was easier through ASH data stored in AWR. Though ASH data present in the AWR doesn't contain every second detail but infact has every 10th second details captured its worth to consider it in this investigation. Also we have the privilege to re-execute ```MERGE``` from application team so that DBA can closely monitor the situation and collect all the necessary information. But before proceeding it's better to gain historical background information of this ```MERGE``` statement as much as possible by mining AWR base tables. 

```sql
SQL> select * from table(dbms_xplan.display_awr('b4k66afzzbbmw'));

Plan hash value: 452146253

----------------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name                       | Rows  |   TQ  |IN-OUT| PQ Distrib |
----------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT                       |                            |       |       |      |            |
|   1 |  PX COORDINATOR                       |                            |       |       |      |            |
|   2 |   PX SEND QC (RANDOM)                 | :TQ10002                   |     8M| Q1,02 | P->S | QC (RAND)  |
|   3 |    INDEX MAINTENANCE                  | DIMENSION                  |       | Q1,02 | PCWP |            |
|   4 |     PX RECEIVE                        |                            |     8M| Q1,02 | PCWP |            |
|   5 |      PX SEND RANGE                    | :TQ10001                   |     8M| Q1,01 | P->P | RANGE      |
|   6 |       MERGE                           | DIMENSION                  |       | Q1,01 | PCWP |            |
|   7 |        PX RECEIVE                     |                            |     8M| Q1,01 | PCWP |            |
|   8 |         PX SEND HYBRID (ROWID PKEY)   | :TQ10000                   |     8M| Q1,00 | P->P | HYBRID (ROW|
|   9 |          VIEW                         |                            |       | Q1,00 | PCWP |            |
|  10 |           NESTED LOOPS OUTER          |                            |     8M| Q1,00 | PCWP |            |
|  11 |            PX BLOCK ITERATOR          |                            |       | Q1,00 | PCWC |            |
|  12 |             TABLE ACCESS STORAGE FULL | STAGE                      |     8M| Q1,00 | PCWP |            |
|  13 |            TABLE ACCESS BY INDEX ROWID| DIMENSION                  |     1 | Q1,00 | PCWP |            |
|  14 |             INDEX UNIQUE SCAN         | DIMENSION_PK               |     1 | Q1,00 | PCWP |            |
----------------------------------------------------------------------------------------------------------------
```

Output has been edited for brevity. As per the execution plan extracted from AWR it looks like ```DIMENSION``` and ```STAGE``` tables will be joined by using ```NESTED LOOP OUTER``` and since ```STAGE``` table is the driving table for each row in ```STAGE``` table ```DIMENSION``` table will be looked up for corresponding row using primary key index ```DIMENSION_PK```. By looking at the parallel operations in the execution plan it can be confirmed that both DML and SELECT were executed in parallel. Now let's check how much time it has spent for each execution of this ```MERGE```, this can be found easily by seggregating the data for each ```sql_exec_id``` which is unique id for each execution of the same sql. Since ASH information in AWR is sample of sample data having 1 out of 10 records we need to multiply the result by 10 to approximately find elapsed time of this ```MERGE``` statement for each distinct execution. 

```sql
SQL> select   sql_id,sql_plan_hash_value,sql_exec_id,round((count(*) * 10)/60/60, 2) as Hours
     from     dba_hist_active_sess_history 
   where    sql_id='b4k66afzzbbmw' 
   group by   sql_id,sql_plan_hash_value,sql_exec_id 
   order by   4;

SQL_ID        SQL_PLAN_HASH_VALUE SQL_EXEC_ID      Hours
------------- ------------------- ----------- ----------
b4k66afzzbbmw           452146253    67108864       5.31
b4k66afzzbbmw           452146253    83886080       9.92
```

This ```MERGE``` statement has indeed ran for more than 4 hours, so now let's check for what purpose did ```MERGE``` has consumed most of it's elapsed time by seggregating the data at each operation of ```MERGE``` execution plan for distinct ```sql_exec_id``` on individual object. 

```sql
SQL> l
  1  select dash.sql_exec_id,dash.SQL_PLAN_OPERATION||' '||dash.SQL_PLAN_OPTIONS as Operation,dsp.OBJECT_NAME,count(*)
  2  from dba_hist_active_sess_history dash, dba_hist_sql_plan dsp
  3  where dash.sql_id='b4k66afzzbbmw'
  4  and   dash.SQL_PLAN_LINE_ID=dsp.id
  5  and   dash.sql_id=dsp.sql_id
  6  and   dash.sql_plan_hash_value=dsp.plan_hash_value
  7* group by dash.sql_exec_id,dash.SQL_PLAN_OPERATION,dash.SQL_PLAN_OPTIONS,dsp.OBJECT_NAME order by dash.sql_exec_id,count(*)
SQL> /

SQL_EXEC_ID OPERATION                      OBJECT_NAME                       COUNT(*)
----------- ------------------------------ ------------------------------- ----------
   67108864 PX RECEIVE                                                              1
            INDEX UNIQUE SCAN              DIMENSION_PK                             3
            TABLE ACCESS STORAGE FULL      STAGE                                    4
            PX SEND HYBRID (ROWID PKEY)    :TQ10000                                 8
            TABLE ACCESS BY INDEX ROWID    DIMENSION                              218
            MERGE                          DIMENSION                             1677

   83886080 PX BLOCK ITERATOR                                                       1
            VIEW                                                                    1
            INDEX UNIQUE SCAN              DIMENSION_PK                             2
            TABLE ACCESS STORAGE FULL      STAGE                                    3
            PX RECEIVE                                                              8
            PX SEND HYBRID (ROWID PKEY)    :TQ10000                                11
            TABLE ACCESS BY INDEX ROWID    DIMENSION                              666
            MERGE                          DIMENSION                             2874
```

This distribution confirms that index lookup on dimension table is the one consuming most of the time and in result causing high CPU usage. So the question arises why ```NESTED LOOP``` is being performed annoyingly instead of ```HASH JOIN``` when we are joining huge amount of ```DIMENSION``` and ```STAGE``` tables data. This made us curious to execute the ```MERGE``` statement manually by DBA and confirm the hypothesis of Optimizer not considering ```HASH JOIN```.

Upon manually executing the ```MERGE``` statement, execution plan shows it is using ```HASH JOIN``` instead of ```NESTED LOOP``` join as shown below. Then why MERGE is selecting different ```NESTED LOOP``` plan when executed from Application ?, and also its obvious that performing ```NESTED LOOP``` join by probing the primary key index for about 485 Million times was not pratical enough as it totally avoids ```SMART SCAN```. 

```
-----------------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name                       | E-Rows |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT                        |                            |        |       |       |          |
|   1 |  PX COORDINATOR                        |                            |        |       |       |          |
|   2 |   PX SEND QC (RANDOM)                  | :TQ10004                   |   8433K|       |       |          |
|   3 |    INDEX MAINTENANCE                   | DIMENSION                  |        |       |       |          |
|   4 |     PX RECEIVE                         |                            |   8433K|       |       |          |
|   5 |      PX SEND RANGE                     | :TQ10003                   |   8433K|       |       |          |
|   6 |       MERGE                            | DIMENSION                  |        |   256K|   256K|          |
|   7 |        PX RECEIVE                      |                            |   8433K|       |       |          |
|   8 |         PX SEND HYBRID (ROWID PKEY)    | :TQ10002                   |   8433K|       |       |          |
|   9 |          VIEW                          |                            |        |       |       |          |
|* 10 |           HASH JOIN OUTER BUFFERED     |                            |   8433K|  2047M|    63M|          |
|  11 |            PX RECEIVE                  |                            |   8433K|       |       |          |
|  12 |             PX SEND HASH               | :TQ10000                   |   8433K|       |       |          |
|  13 |              PX BLOCK ITERATOR         |                            |   8433K|       |       |          |
|* 14 |               TABLE ACCESS STORAGE FULL| STAGE                      |   8433K|  1025K|  1025K|          |
|  15 |            PX RECEIVE                  |                            |    478M|       |       |          |
|  16 |             PX SEND HASH               | :TQ10001                   |    478M|       |       |          |
|  17 |              PX BLOCK ITERATOR         |                            |    478M|       |       |          |
|* 18 |               TABLE ACCESS STORAGE FULL| DIMENSION                  |    478M|  1025K|  1025K|          |
-----------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  10 - access("T1"."DIM_KEY"="T3"."DIM_KEY")
  14 - storage(:Z>=:Z AND :Z<=:Z)
  18 - storage(:Z>=:Z AND :Z<=:Z)
```

After colleting all these details we asked Application to provide us all the information regarding their job which performs this MERGE but vain. So the only option was to re-execute the ```MERGE``` and gather all the required data for further troubleshooting. Our first attempt was to collect all the session level Optimizer related settings for this session by querying ```V$SES_OPTIMIZER_ENV``` as shown below.

```sql
SQL> select * from V$SES_OPTIMIZER_ENV where sid=123 and ISDEFAULT='NO';

       SID    ID NAME                             SQL_FEATURE      ISD VALUE        
---------- ----- -------------------------------- ---------------- --- -------------
      1283     4 parallel_dml_forced_dop          QKSFM_CBO        NO  default
      1283    13 parallel_threads_per_cpu         QKSFM_CBO        NO  1
      1283    25 _pga_max_size                    QKSFM_ALL        NO  838860 KB
      1283    36 parallel_dml_mode                QKSFM_ALL        NO  forced
      1283   247 parallel_min_time_threshold      QKSFM_PQ         NO  10
      1283   258 optimizer_use_invisible_indexes  QKSFM_INDEX      NO  true
      1283   268 cell_offload_plan_display        QKSFM_EXECUTION  NO  ALWAYS
      1283   274 parallel_max_degree              QKSFM_PQ         NO  128
      1283   291 _parallel_cluster_cache_policy   QKSFM_PQ         NO  adaptive
      1283   324 total_processor_group_count      QKSFM_ALL        NO  8
```

Bingo !! Session has set FORCE DML parallelism. Immediately we stopped the ```MERGE``` job and tried to reproduce the ```NESTED LOOP``` execution plan manually by executing the ```MERGE``` with ```FORCE DML``` set at session level and got the exact ```NESTED LOOP``` plan which was used by Application initiated ```MERGE``` job. By just enabling the DML parallelism without ```FORCE``` option ```HASH JOIN``` was chosed by the Optimizer as shown below. 
```
-----------------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name                       | E-Rows |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT                        |                            |        |       |       |          |
|   1 |  PX COORDINATOR                        |                            |        |       |       |          |
|   2 |   PX SEND QC (RANDOM)                  | :TQ10004                   |   8433K|       |       |          |
|   3 |    INDEX MAINTENANCE                   | DIMENSION                  |        |       |       |          |
|   4 |     PX RECEIVE                         |                            |   8433K|       |       |          |
|   5 |      PX SEND RANGE                     | :TQ10003                   |   8433K|       |       |          |
|   6 |       MERGE                            | DIMENSION                  |        |   256K|   256K|          |
|   7 |        PX RECEIVE                      |                            |   8433K|       |       |          |
|   8 |         PX SEND HYBRID (ROWID PKEY)    | :TQ10002                   |   8433K|       |       |          |
|   9 |          VIEW                          |                            |        |       |       |          |
|* 10 |           HASH JOIN OUTER BUFFERED     |                            |   8433K|  2047M|    63M|          |
|  11 |            PX RECEIVE                  |                            |   8433K|       |       |          |
|  12 |             PX SEND HASH               | :TQ10000                   |   8433K|       |       |          |
|  13 |              PX BLOCK ITERATOR         |                            |   8433K|       |       |          |
|* 14 |               TABLE ACCESS STORAGE FULL| STAGE                      |   8433K|  1025K|  1025K|          |
|  15 |            PX RECEIVE                  |                            |    478M|       |       |          |
|  16 |             PX SEND HASH               | :TQ10001                   |    478M|       |       |          |
|  17 |              PX BLOCK ITERATOR         |                            |    478M|       |       |          |
|* 18 |               TABLE ACCESS STORAGE FULL| DIMENSION                  |    478M|  1025K|  1025K|          |
-----------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  10 - access("T1"."DIM_KEY"="T3"."DIM_KEY")
  14 - storage(:Z>=:Z AND :Z<=:Z)
  18 - storage(:Z>=:Z AND :Z<=:Z)
```

This is because when we set ```FORCE``` parallelism Optimizer will change its cost calculation by reducing the cost estimates for full table scans which is directly proportional to degree of parallelism. Application team were not aware of this and thought ```FORCE``` option does something forcefully to obey the degree of parallelism requested which is not true.

At this moment we suggested to use ```"alter session enable parallel dml"``` instead of ```"alter session force parallel dml"``` to resolve this issue, so that Optimizer will perform ```HASH JOIN``` and smart scan could occur to improve the ```MERGE``` response time. But to our surprise Application came back saying that their job is still taking more time and is currently running from past 30 minutes. Below is the monitoring report for currently running ```MERGE``` job.

```
Global Information
------------------------------
 Status              :  EXECUTING
 Instance ID         :  1
 Session             :  DW (1092:45235)
 SQL ID              :  b4k66afzzbbmw
 SQL Execution ID    :  16777216
 Execution Started   :  02/19/2016 10:58:48
 First Refresh Time  :  02/19/2016 10:58:48
 Last Refresh Time   :  02/19/2016 11:38:52
 Duration            :  2612s
 Module/Action       :  sqlplus@db10 (TNS V1-/-
 Service             :  stage
 Program             :  sqlplus@db10 (TNS V1-

Global Stats
=========================================================================================================================
| Elapsed |   Cpu   |    IO    | Application | Concurrency | Cluster  | Buffer | Read | Read  | Write | Write |  Cell   |
| Time(s) | Time(s) | Waits(s) |  Waits(s)   |  Waits(s)   | Waits(s) |  Gets  | Reqs | Bytes | Reqs  | Bytes | Offload |
=========================================================================================================================
|   22271 |    7530 |    14261 |        0.00 |        0.33 |      480 |   119M |  24M | 274GB | 34515 |   8GB |  -9.89% |
=========================================================================================================================

SQL Plan Monitoring Details (Plan Hash Value=2664465517)
==================================================================================================================================================
| Id    |                Operation                 |    Name     | Rows   |  Rows   | Cell  | Activity |             Activity Detail             |
|       |                                          |             |(Estim) |(Actual) |Offload|   (%)    |               (# samples)               |
==================================================================================================================================================
|     0 | MERGE STATEMENT                          |             |        |       0 |       |     0.04 | Cpu (7)                                 |
|     1 |   PX COORDINATOR                         |             |        |       0 |       |     0.01 | name-service call wait (1)              |
|     2 |    PX SEND QC (RANDOM)                   | :TQ10004    |     8M |         |       |          |                                         |
|     3 |     INDEX MAINTENANCE                    | DIMENSION   |        |         |       |          |                                         |
|     4 |      PX RECEIVE                          |             |     8M |         |       |          |                                         |
|     5 |       PX SEND RANGE                      | :TQ10003    |     8M |         |       |          |                                         |
|  -> 6 |        MERGE                             | DIMENSION   |        |       0 |       |     6.70 | gc buffer busy release (14)             |
|       |                                          |             |        |         |       |          | gc cr block 2-way (7)                   |
|       |                                          |             |        |         |       |          | gc cr block busy (6)                    |
|       |                                          |             |        |         |       |          | gc cr failure (1)                       |
|       |                                          |             |        |         |       |          | gc cr request (1)                       |
|       |                                          |             |        |         |       |          | gc current block 2-way (4)              |
|       |                                          |             |        |         |       |          | gc current block 3-way (39)             |
|       |                                          |             |        |         |       |          | gc current block busy (274)             |
|       |                                          |             |        |         |       |          | gc current grant busy (120)             |
|       |                                          |             |        |         |       |          | gc current request (5)                  |
|       |                                          |             |        |         |       |          | gc current retry (3)                    |
|       |                                          |             |        |         |       |          | gc remaster (4)                         |
|       |                                          |             |        |         |       |          | Cpu (465)                               |
|       |                                          |             |        |         |       |          | gcs drm freeze in enter server mode (2) |
|       |                                          |             |        |         |       |          | cell single block physical read (372)   |
|  -> 7 |         PX RECEIVE                       |             |     8M |      1M |       |     0.04 | Cpu (7)                                 |
|  -> 8 |          PX SEND HYBRID (ROWID PKEY)     | :TQ10002    |     8M |      1M |       |     0.06 | Cpu (9)                                 |
|       |                                          |             |        |         |       |          | PX Deq: reap credit (2)                 |
|  -> 9 |           VIEW                           |             |        |      1M |       |     0.01 | Cpu (1)                                 |
| -> 10 |            HASH JOIN OUTER BUFFERED      |             |     8M |      1M |       |     1.26 | Cpu (187)                               |
|       |                                          |             |        |         |       |          | direct path read temp (46)              |
|       |                                          |             |        |         |       |          | direct path write temp (14)             |
|    11 |             PX RECEIVE                   |             |     8M |      8M |       |     0.08 | Cpu (6)                                 |
|       |                                          |             |        |         |       |          | PX Deq: reap credit (9)                 |
|    12 |              PX SEND HASH                | :TQ10000    |     8M |      8M |       |     0.08 | Cpu (12)                                |
|       |                                          |             |        |         |       |          | PX Deq: reap credit (3)                 |
|    13 |               PX BLOCK ITERATOR          |             |     8M |      8M |       |          |                                         |
|    14 |                TABLE ACCESS STORAGE FULL | STAGE       |     8M |      8M |       |     0.11 | gc cr grant 2-way (1)                   |
|       |                                          |             |        |         |       |          | gc cr multi block request (1)           |
|       |                                          |             |        |         |       |          | gc current grant 2-way (1)              |
|       |                                          |             |        |         |       |          | Cpu (15)                                |
|       |                                          |             |        |         |       |          | cell list of blocks physical read (2)   |
|       |                                          |             |        |         |       |          | cell multiblock physical read (1)       |
|    15 |             PX RECEIVE                   |             |   478M |    478M |       |     2.66 | Cpu (415)                               |
|       |                                          |             |        |         |       |          | PX Deq: Table Q Normal (2)              |
|       |                                          |             |        |         |       |          | IPC send completion sync (1)            |
|       |                                          |             |        |         |       |          | PX Deq: reap credit (105)               |
|    16 |              PX SEND HASH                | :TQ10001    |   478M |    478M |       |     6.04 | Cpu (974)                               |
|       |                                          |             |        |         |       |          | PX Deq Credit: need buffer (1)          |
|       |                                          |             |        |         |       |          | PX Deq: reap credit (212)               |
|    17 |               PX BLOCK ITERATOR          |             |   478M |    478M |       |          |                                         |
|    18 |                TABLE ACCESS STORAGE FULL | DIMENSION   |   478M |    478M | -7.53%|    82.94 | gc current block 2-way (9)              |
|       |                                          |             |        |         |       |          | gc current block congested (1)          |
|       |                                          |             |        |         |       |          | Cpu (2676)                              |
|       |                                          |             |        |         |       |          | cell single block physical read (13607) |
|       |                                          |             |        |         |       |          | cell smart table scan (9)               |
==================================================================================================================================================
```

Above sql monitoring report has been edited for brevity. As per the monitoring report more than 80% of the time is spent on Id 18 performing full scan of table ```DIMENSION```. But though full scan of the ```DIMENSION``` table has performed smart scan we see that it has been waiting on the wait event ```"cell single block physical read"``` for 13K times. This raised the concern about why full table scan has to perform 13K single block reads which is reducing the beneficial effects of smart scan by sending the blocks back to compute nodes from cell nodes and thus raising the CPU usage on compute nodes for decompressing the HCC compression units as ```DIMENSION``` table is ```HCC compressed with Query High``` option. 

Updates on HCC enabled table will have adverse impact, because when row is update in HCC tables then row will be migrated within the segment which is managed using OLTP compression type. So even if there is a single migrated row in HCC compression unit then the block will be accessed by using ```"cell single block physical read"```. With this hypothesis we can say that ```MERGE``` statement was suffering from migrated rows due to updates on the HCC ```DIMENSION``` table. 

So how do we find how many rows are migrated in table and whether migrated rows are using OLTP compression method or not? Let's create a demo to simulate row migration on HCC table and study its behaviour.

Create a table with one million rows as HCC with Query High option.

```sql
SQL> create table t1 tablespace
     as
     select
             trunc((rownum-1)/10)    n1,
             trunc((rownum-1)/10)    n2,
             rpad(rownum,200)        v1
     from
             dual
     connect by
             level <= 1e6
/
 
SQL> exec dbms_stats.gather_table_stats(user,'t1',method_opt => 'for all columns size 1');
```

Now to find the compression type used by each row in an block we can use ```dbms_compression``` package having function ```get_compression_type```. This function returns few known values which can be inferred by looking at the definition of each values in ```dbmscomp.sql``` script used to create this package. 

```sql
SQL> !cat $ORACLE_HOME/rdbms/admin/dbmscomp.sql | grep ^COMP_[NFB]
COMP_NOCOMPRESS               CONSTANT NUMBER := 1;
COMP_FOR_OLTP                 CONSTANT NUMBER := 2;
COMP_FOR_QUERY_HIGH           CONSTANT NUMBER := 4;
COMP_FOR_QUERY_LOW            CONSTANT NUMBER := 8;
COMP_FOR_ARCHIVE_HIGH         CONSTANT NUMBER := 16;
COMP_FOR_ARCHIVE_LOW          CONSTANT NUMBER := 32;
COMP_BLOCK                    CONSTANT NUMBER := 64;
```

By using these values we can construct an sql to decode each value and represent it in meaningful way. Just to have stable demo and to check smart scan efficiency we will force smart scan by setting  ```"_serial_direct_read"``` to ```ALWAYS``` at session level.

```sql
SQL> alter session set "_serial_direct_read"=ALWAYS;

SQL> SELECT compression_type, n_rows as num_rows
     FROM  (`
        SELECT compression_type, sum(rows) n_rows
        FROM  (
           SELECT decode(dbms_compression.get_compression_type(USER, 'T1', min_rowid),
                         1, 'No Compression',
                         2, 'Basic/OLTP Compression',
                         4, 'HCC Query High',
                         8, 'HCC Query Low',
                        16, 'HCC Archive High',
                        32, 'HCC Archive Low',
                        64, 'From HCC to OLTP',
                            'Unknown Compression Level' ) AS compression_type, rows
           FROM  ( SELECT   min(rowid) as min_rowid,count(*) as rows 
               FROM     T1 
           GROUP BY dbms_rowid.rowid_block_number(rowid) 
         )
      )
        GROUP  BY compression_type
    )
/

COMPRESSION_TYPE                ROWS
------------------------- ----------
HCC Query High               1000000

Elapsed: 00:00:06.19
```

This sql ran for 6 seconds and it reports that we have one million rows compressed as HCC Query High option. Now let's update the rows and again find the compression type used by each row.

```sql
SQL> update t1
        set v1 = rpad(rownum,190)
        where n1  between 1 and 100000
/
999990 rows updated.

SQL> commit;
Commit complete.

SQL> SELECT compression_type, n_rows as num_rows
     FROM  (
        SELECT compression_type, sum(rows) n_rows
        FROM  (
           SELECT decode(dbms_compression.get_compression_type(USER, 'T1', min_rowid),
                         1, 'No Compression',
                         2, 'Basic/OLTP Compression',
                         4, 'HCC Query High',
                         8, 'HCC Query Low',
                        16, 'HCC Archive High',
                        32, 'HCC Archive Low',
                        64, 'From HCC to OLTP',
                            'Unknown Compression Level' ) AS compression_type, rows
           FROM  ( SELECT   min(rowid) as min_rowid,count(*) as rows 
               FROM     T1 
           GROUP BY dbms_rowid.rowid_block_number(rowid) 
         )
      )
        GROUP  BY compression_type
    )
/

COMPRESSION_TYPE                ROWS
------------------------- ----------
No Compression                    70
HCC Query High                    10
From HCC to OLTP              999920

Elapsed: 00:03:35.92
```

Now the same sql took 201 seconds to provide the report !! As per the report now we have 70 rows which are not using any compression type, because after update rows get migrated from HCC to OLTP compression type and these migrated rows will not be compressed into OLTP type until the block is full, so the 70 rows are residing in the blocks which are not yet full to get OLTP compressed. Further we see that we have 10 rows as HCC Query High and 999920 migrated rows compressed as OLTP. These migrated rows will correspond to return value 64 which is not yet documented any where as migrated rows but simply represents as ```COMP_BLOCK``` in ```dbmscomp.sql``` script. As per the test case we can consider return value 64 as rows migrated from HCC to OLTP. 

Since this reporting sql took 201 seconds after update, it would be interesting to see it's monitoring report to find where it has spent most of its time.

```
SQL Monitoring Report
Global Stats
==============================================================================================================
| Elapsed |   Cpu   |    IO    | Concurrency | Cluster  | PL/SQL  |  Other   | Fetch | Buffer | Read | Read  |
| Time(s) | Time(s) | Waits(s) |  Waits(s)   | Waits(s) | Time(s) | Waits(s) | Calls |  Gets  | Reqs | Bytes |
==============================================================================================================
|     201 |     198 |     0.99 |        0.00 |     0.28 |    4.29 |     1.26 |     2 |    55M | 1205 |  22MB |
==============================================================================================================

SQL Plan Monitoring Details (Plan Hash Value=4120307800)
===============================================================================================================================================================================
| Id |           Operation            | Name |  Rows   | Cost |   Time    | Start  | Execs |   Rows   | Read | Read  |  Mem  | Activity |           Activity Detail           |
|    |                                |      | (Estim) |      | Active(s) | Active |       | (Actual) | Reqs | Bytes | (Max) |   (%)    |             (# samples)             |
===============================================================================================================================================================================
|  0 | SELECT STATEMENT               |      |         |      |       193 |     +8 |     1 |        3 |      |       |       |          |                                     |
|  1 |   HASH GROUP BY                |      |      1M |  116 |       193 |     +8 |     1 |        3 |      |       |  962K |          |                                     |
|  2 |    VIEW                        |      |      1M |   77 |       193 |     +8 |     1 |    25001 |      |       |       |     1.00 | Cpu (2)                             |
|  3 |     HASH GROUP BY              |      |      1M |   77 |       201 |     +0 |     1 |    25001 |      |       |    3M |     3.00 | Cpu (6)                             |
|  4 |      TABLE ACCESS STORAGE FULL | T1   |      1M |   38 |         7 |     +2 |     1 |       1M |   89 | 776KB |       |     1.00 | cell single block physical read (8) |
===============================================================================================================================================================================
```

Though we forced smart scan by setting ```"_serial_direct_read"``` this sql has performed single block physical reads due to migrated rows. Degrade in performance is huge due to this from 6 seconds to 201 seconds after updating the rows. This proves any access of migrated rows will be done through ```"cell single block physical read"``` and thus Exadata uses block shipping mode which in turn increases the CPU usage for decompressing the blocks reducing the efficeincy of smart scan to great order. There are various reasons ```cellsrv``` reverts back to block shipping mode rather smart scan and one of the reason is - Migrated rows in HCC compression tables. When migrated rows are encountered by ```cellsrv``` process there is no guarantee that the migrated row block resides in the same cell server, hence ```cellsrv``` will just ships the entire block back to the compute node to process the request.


Conclusion
----------
This article demonstrates the methodology used for tuning ```MERGE``` statement by drilling down into the issue, intially it was using wrong join mechanism due to FORCE parallel session level setting and then later even after converting the plan to use ```HASH JOIN``` by disabling FORCE parallelism it was taking more than expected time due to migrated rows being accessed by ```"cell single block physical reads"``` and all other rows through smart scan. But the adverse impact due to this was huge and implementing right feature for right requirement is crucial. In this case using HCC compression for frquently updating tables is a bad design strategy. HCC provides tremendous compression ratio but not suitable for tables where frequently conventional DML's are performed, especially the updates. 