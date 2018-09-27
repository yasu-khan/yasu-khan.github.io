---
layout: post
title: Oracle - Bloom Filter
description: Bloom Filter operation in Oracle
comments: true
keywords: Oracle, performance, Bloom Filter, cardinality misestimate
---


Introduction
-------------
Bloom filter was invented in 1970 by Burton H. Bloom, it is a light weight probalistic algorithm which provides probability of presence and due to its nature there is a possiblity of false prositive of a presence, but it can never run into false negative. This helps in finding whether a element is member of set or not, its memory footprint is very small and thus reduces the unwanted traffic by fitering the data at earlier stages. Oracle has leveraged this in several places to improve the response time of the queries.
	
In Oracle there is no need of any manual intervention for this to be enable, it gets created automatically and used in the execution plan wherever it desires to be fit for the use case. Bloom filter can occur for both parallel and serial operations. Due to the way Bloom filters works it is applicable only for HASH JOIN but not for other types of joins. At certain scenarios multiple Bloom filters can also be applied for the same table. Reporting creation and usage of Bloom filters in execution plan was very confusing and in-complete in versions prior to 12c, from 12c onwards execution plan has complete reporting details of Bloom filter creation and usage.

It is mainly used for -
* Reducing traffic between parallel slave processes when performing joins
* Replacement of Subquery partition pruning if desired.
* Existance of data in server result cache
* Filtering data efficiently in Exadata when joining ```FACT``` and ```DIMESION``` tables in data warehousing environment.
	
Bloom filter use case - Filtering
---------------------------------
Let's see how Bloom filters gets created and used by creating ```FACT``` and ```DIMENSION``` tables and join it using ```HASH JOIN``` with filtering condition on ```DIMENSION``` table. This kind of environment and requirement is very usual and common in data wareshousing eco-systems. 

```sql 
SQL> CREATE TABLE DIMENSION
  (
     col1,
     col2
  ) AS
  SELECT MOD( ROWNUM, 10 ),
         ROWNUM
  FROM   DUAL
  CONNECT BY ROWNUM <= 100;

Table created.

SQL> CREATE TABLE FACT
  (
     col1,
     col2
  ) AS
  SELECT MOD( ROWNUM, 25 ),
         ROWNUM
  FROM   DUAL
  CONNECT BY ROWNUM <= 1000000;

Table created.

SQL> exec dbms_stats.gather_table_stats(USER, 'DIMENSION');

PL/SQL procedure successfully completed.

SQL> exec dbms_stats.gather_table_stats(USER, 'FACT');

PL/SQL procedure successfully completed.
```

Now lets execute the query in parallel which join these two tables along with filter condition on ```DIMENSION``` table. 
```sql
SQL> SELECT /*+ parallel(8) */ count( * )
     FROM   FACT,
            DIMENSION
     WHERE  DIMENSION.col1 = 1 AND
            DIMENSION.col2 = FACT.col2;

  COUNT(*)
----------
       100
```

Pull the execution plan from the cursor using ```dbms_xplan``` along with ```ALLSTATS ALL``` option to get the aggregated statistics from all the parallel slaves and Query Co-ordinator. 	   
```sql
SQL> select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS ALL'));

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------
SQL_ID  587ty27pjwphu, child number 0
-------------------------------------
SELECT /*+ parallel(8) */ count(*) FROM FACT, DIMENSION WHERE DIMENSION.col1 = 1 AND DIMENSION.col2 = FACT.col2

Plan hash value: 4106007966
-----------------------------------------------------------------------------------------------------
| Id  | Operation                | Name      | Starts | E-Rows |   TQ  |IN-OUT| PQ Distrib | A-Rows |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT         |           |      1 |        |       |      |            |      1 |
|   1 |  SORT AGGREGATE          |           |      1 |      1 |       |      |            |      1 |
|   2 |   PX COORDINATOR         |           |      1 |        |       |      |            |      8 |
|   3 |    PX SEND QC (RANDOM)   | :TQ10000  |      0 |      1 | Q1,00 | P->S | QC (RAND)  |      0 |
|   4 |     SORT AGGREGATE       |           |      4 |      1 | Q1,00 | PCWP |            |      4 |
|*  5 |      HASH JOIN           |           |      4 |    100 | Q1,00 | PCWP |            |    100 |
|   6 |       JOIN FILTER CREATE | :BF0000   |      4 |    100 | Q1,00 | PCWP |            |    400 |
|*  7 |        TABLE ACCESS FULL | DIMENSION |      4 |    100 | Q1,00 | PCWP |            |    400 |
|   8 |       JOIN FILTER USE    | :BF0000   |      4 |   1000K| Q1,00 | PCWP |            |    102 |
|   9 |        PX BLOCK ITERATOR |           |      4 |   1000K| Q1,00 | PCWC |            |    102 |
|* 10 |         TABLE ACCESS FULL| FACT      |     71 |   1000K| Q1,00 | PCWP |            |    102 |
-----------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   5 - access("DIMENSION"."COL2"="FACT"."COL2")
   7 - filter("DIMENSION"."COL1"=1)
  10 - access(:Z>=:Z AND :Z<=:Z)
       filter(SYS_OP_BLOOM_FILTER(:BF0000,"FACT"."COL2"))

Note
-----
   - Degree of Parallelism is 8 because of hint
```

I have trimmed the execution plan output for pretty print. As per the execution plan Bloom filter got created at line id 6 for the joining ```COL1``` of table ```DIMENSION``` and the same Bloom filter got used at line id 8 for the ```FACT``` table. Further more if we look at the predicate section we could see a function ```SYS_OP_BLOOM_FILTER``` has been used when accessing ```FACT``` table at line id 10. This is where Bloom filter is getting used to filter the data from large ```FACT``` table before it could be processed with the HASH join. Actual rows column proves it that though ```FACT``` table is going for full table scan it has filtered down the records from 1 million down to just few 102 rows. But Bloom filter can provide false positive results and thus these false positive results has to be eliminated at later stage. Here when performing join between two sets at line id 5 false positive results will be verified. This concludes that though Bloom filters are efficient probalistic algorithm we need to have filter condition due to chances of false positive results. 

In this case the usage of Bloom filter is based on join selectivity estimation along with the total amount of data to be processed. Thus if we want to skip these estimation for Bloom filter enabling/disabling purpose we can use the hints ```PX_JOIN_FILTER/NO_PX_JOIN_FILTER``` accordingly. But in some cases if we have very less amount of data to be processed and if we force usage of Bloom filter through hint ```PX_JOIN_FILTER``` then it may not work. 

Let's check what overhead will be when we disable Bloom filter usage using hint ```NO_PX_JOIN_FILTER```.
```sql
SQL> SELECT /*+ parallel(8) no_px_join_filter(FACT) */ count( * )
     FROM   FACT,
            DIMENSION
     WHERE  DIMENSION.col1 = 1 AND
            DIMENSION.col2 = FACT.col2;

  COUNT(*)
----------
       100

SQL> select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS ALL'));

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------------
SQL_ID  dw58fyg7xdtv6, child number 1
-------------------------------------
SELECT /*+ parallel(8) no_px_join_filter(FACT) */ count(*) FROM FACT,DIMENSION WHERE DIMENSION.col1 = 1 AND DIMENSION.col2 = FACT.col2

Plan hash value: 4252532053
--------------------------------------------------------------- ------------------------------------
| Id  | Operation               | Name      | Starts | E-Rows |   TQ  |IN-OUT| PQ Distrib | A-Rows |
--------------------------------------------------------------- ------------------------------------
|   0 | SELECT STATEMENT        |           |      1 |        |       |      |            |      1 |
|   1 |  SORT AGGREGATE         |           |      1 |      1 |       |      |            |      1 |
|   2 |   PX COORDINATOR        |           |      1 |        |       |      |            |      4 |
|   3 |    PX SEND QC (RANDOM)  | :TQ10000  |      0 |      1 | Q1,00 | P->S | QC (RAND)  |      0 |
|   4 |     SORT AGGREGATE      |           |      4 |      1 | Q1,00 | PCWP |            |      4 |
|*  5 |      HASH JOIN          |           |      4 |    100 | Q1,00 | PCWP |            |    100 |
|*  6 |       TABLE ACCESS FULL | DIMENSION |      4 |    100 | Q1,00 | PCWP |            |    400 |
|   7 |       PX BLOCK ITERATOR |           |      4 |   1000K| Q1,00 | PCWC |            |   1000K|
|*  8 |        TABLE ACCESS FULL| FACT      |     52 |   1000K| Q1,00 | PCWP |            |   1000K|
--------------------------------------------------------------- ------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   5 - access("DIMENSION"."COL2"="FACT"."COL2")
   6 - filter("DIMENSION"."COL1"=1)
   8 - access(:Z>=:Z AND :Z<=:Z)

Note
-----
   - Degree of Parallelism is 8 because of hint
```

As per the execution we processed complete 1 million rows from table ```FACT``` without any filteration. This also increases the traffic between parallel query slaves causing direct increase in response time.   
	   
So why do we have ```(:Z>=:Z AND :Z<=:Z)``` in predicate section ? whats the purpose of it ? 
Usually these occurs when performing parallel table scans, each parallel slave process gets range of rowid's to be scan and thus each parallel slave does the check in this way to ensure their assigned rowid's are within the range.


Bloom filter use case - Partition Pruning
-----------------------------------------
During run-time partition pruning Oracle will generate the execution plan that allows a subquery to be run against the dimension table to determine the target partitions. This is the case only when HASH join is in use but not for NESTED LOOP join as query would access only the partitions corresponding to the row from the driving table, so pruning would take place implicitly. If WHERE predicate is applied to the table is small compared to the partitioned table, and is rows are filtered for the partition table is significant, then it performs subquery partition pruning using a recursive subquery, the decision purely depends upon internal cost-based decision of the optimizer.

Let's recreate our ```FACT``` as partitioned table and perform serial query instead of parallel to check the behaviour of Subquery Pruning.
```sql
SQL> drop table FACT;

Table dropped.

SQL> CREATE TABLE FACT
       (
          col1,
          col2
       )
      PARTITION BY RANGE(col1)
      SUBPARTITION BY HASH(col2)
      SUBPARTITIONS 8
      (PARTITION P1 VALUES LESS THAN (5),
       PARTITION P2 VALUES LESS THAN (10),
       PARTITION P3 VALUES LESS THAN (15),
       PARTITION P4 VALUES LESS THAN (20),
       PARTITION P5 VALUES LESS THAN (25))
     AS
     SELECT MOD( ROWNUM, 25 ),
            ROWNUM
     FROM   DUAL
     CONNECT BY ROWNUM <= 1000000;

Table created.

SQL> exec dbms_stats.gather_table_stats(USER, 'FACT');

PL/SQL procedure successfully completed.
```
Now we need to disable Bloom partition pruning to observe Subquery partition pruning behaviour, but since we don't have any hint to disable this we need to set parameter ```"_bloom_pruning_enabled"``` to false.  

```sql
SQL> alter session set "_bloom_pruning_enabled"=FALSE;

Session altered.

SQL> SELECT count( * )
     FROM   FACT,
            DIMENSION
     WHERE  DIMENSION.col1 = 1 AND
            DIMENSION.col2 = FACT.col2;
	   
  COUNT(*)
----------
        10

SQL> select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS ALL'));

PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------
SQL_ID  adrsj1m9a4mbt, child number 1
-------------------------------------
SELECT count( * ) FROM FACT, DIMENSION WHERE DIMENSION.col1 = 1 AND DIMENSION.col2 = FACT.col2

Plan hash value: 2660913463
-------------------------------------------------------------------------------------------
| Id  | Operation                  | Name      | Starts | E-Rows | Pstart| Pstop | A-Rows |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |           |      1 |        |       |       |      1 |
|   1 |  SORT AGGREGATE            |           |      1 |      1 |       |       |      1 |
|*  2 |   HASH JOIN                |           |      1 |     10 |       |       |     10 |
|*  3 |    TABLE ACCESS FULL       | DIMENSION |      1 |     10 |       |       |     10 |
|   4 |    PARTITION RANGE ALL     |           |      1 |   1000K|     1 |     5 |    750K|
|   5 |     PARTITION HASH SUBQUERY|           |      5 |   1000K|KEY(SQ)|KEY(SQ)|    750K|
|   6 |      TABLE ACCESS FULL     | FACT      |     30 |   1000K|     1 |    40 |    750K|
-------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("DIMENSION"."COL2"="FACT"."COL2")
   3 - filter("DIMENSION"."COL1"=1)
```

The operation ```"PARTITION HASH SUBQUERY"``` at line id 5 says that subquery partition pruning has occurred on ```FACT``` table as shown by the value as ```"KEY(SQ)"``` for Pstart and Pstop. But in recent Oracle versions this behaviour has changed, Bloom partition pruning will be taking place wherever Subquery pruning is used. Let's reset this parameter ```"_bloom_pruning_enabled"``` to TRUE(Default value) and see how Bloom will be used for pruning partitions.

```sql
SQL> alter session set "_bloom_pruning_enabled"=TRUE;

Session altered.

SQL> SELECT count( * )
     FROM   FACT,
            DIMENSION
     WHERE  DIMENSION.col1 = 1 AND
            DIMENSION.col2 = FACT.col2;

  COUNT(*)
----------
        10

SQL> select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS ALL'));

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
SQL_ID  gfq5ptr464upb, child number 1
-------------------------------------
SELECT count( * ) FROM FACT, DIMENSION WHERE DIMENSION.col1 = 1 AND DIMENSION.col2 = FACT.col2

Plan hash value: 3885867154
----------------------------------------------------------------------------------------------
| Id  | Operation                     | Name      | Starts | E-Rows | Pstart| Pstop | A-Rows |
----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |           |      1 |        |       |       |      1 |
|   1 |  SORT AGGREGATE               |           |      1 |      1 |       |       |      1 |
|*  2 |   HASH JOIN                   |           |      1 |     10 |       |       |     10 |
|   3 |    PART JOIN FILTER CREATE    | :BF0000   |      1 |     10 |       |       |     10 |
|*  4 |     TABLE ACCESS FULL         | DIMENSION |      1 |     10 |       |       |     10 |
|   5 |    PARTITION RANGE ALL        |           |      1 |   1000K|     1 |     5 |    750K|
|   6 |     PARTITION HASH JOIN-FILTER|           |      5 |   1000K|:BF0000|:BF0000|    750K|
|   7 |      TABLE ACCESS FULL        | FACT      |     30 |   1000K|     1 |    40 |    750K|
----------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("DIMENSION"."COL2"="FACT"."COL2")
   4 - filter("DIMENSION"."COL1"=1)
```

In the above execution plan Bloom filter has been created at line id 3 based on the data returned by line id 4, by using this Bloom filter partition pruning on table ```FACT``` occurs(Bloom usgae in Pstart and Pstop) and hence only the partitions containing the relevant data is scanned. But when we check Pstart(1) and Pstop(40) columns it says all partitions(5 partitons * 8 subpartitions=40) of the ```FACT``` table have been scanned, intrestingly if you check Actual-Rows it is 750K but ```FACT``` table has 1 Million rows in total, this confirms Bloom filter was used for pruning subpartions though all the partitions have been scanned.     

Bloom filter use case - Offload in Exadata
------------------------------------------
Bloom filters plays vital role in Exadata. Since HASH join and full table scan are popular operations in Exadata, Bloom filters extrapolate the sql's response time by offloading the Bloom filter funtions down to the cell servers. Data will be filtered at cell server level and thus data returned to the compute node will be drastically saved. We can check if Bloom filter functions can be offloaded or not by querying view ```V$SQLFN_METADATA``` as shown below

```sql
SQL> select name,OFFLOADABLE from V$SQLFN_METADATA where name like '%BLOOM%';

NAME                           OFF
------------------------------ ---
SYS_OP_BLOOM_FILTER            YES
SYS_OP_BLOOM_FILTER_LIST       YES
```

Let's see how execution plan looks like in Exadata when Bloom filters are in use.
```sql
SQL> SELECT count( * )
     FROM   FACT,
            DIMENSION
     WHERE  DIMENSION.col1 = 1 AND
            DIMENSION.col2 = FACT.col2; 

  COUNT(*)
----------
       100

SQL> select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS ALL'));

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------
SQL_ID  adrsj1m9a4mbt, child number 0
-------------------------------------
SELECT count( * ) FROM   FACT, DIMENSION WHERE  DIMENSION.col1 =1 AND DIMENSION.col2 = FACT.col2

Plan hash value: 1328814738
-----------------------------------------------------------------------------
| Id  | Operation                    | Name      | Starts | E-Rows | A-Rows |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |           |      1 |        |      1 |
|   1 |  SORT AGGREGATE              |           |      1 |      1 |      1 |
|*  2 |   HASH JOIN                  |           |      1 |    100 |    100 |
|   3 |    JOIN FILTER CREATE        | :BF0000   |      1 |    100 |    100 |
|*  4 |     TABLE ACCESS STORAGE FULL| DIMENSION |      1 |    100 |    100 |
|   5 |    JOIN FILTER USE           | :BF0000   |      1 |   1000K|   1614 |
|*  6 |     TABLE ACCESS STORAGE FULL| FACT      |      1 |   1000K|   1614 |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("DIMENSION"."COL2"="FACT"."COL2")
   4 - storage("DIMENSION"."COL1"=1)
       filter("DIMENSION"."COL1"=1)
   6 - storage(SYS_OP_BLOOM_FILTER(:BF0000,"FACT"."COL2"))
       filter(SYS_OP_BLOOM_FILTER(:BF0000,"FACT"."COL2"))
```

In Exadata though it is serially executed Bloom filter were created and used. At line id 3 Bloom filter got created and used to filter the rows from FACT table at line id 5. In predicate section we see that Bloom function ```SYS_OP_BLOOM_FILTER``` is offloaded to cell server to futher reduce the data by filtering it in cell server. We can confirm how much data have been offloaded by measuring metrics through session level statistics. In principle Bloom filters existance in Exadata is dominant as it can be completly processed in cell server. 

Bloom partition pruning works exactly similar to non-exadata deployments as it doesn't have anything to offload in cell server except pruning the partitions of the ```FACT``` table. 

Bloom filter use case - Result Cache
------------------------------------
Bloom filters are used in server side result cache to support lookup of the information when query is ran to check if the result of the query is already been cached by some other session in the database. If result is already available then it will be retrieved by the query from server result cache instead of gathering it from database blocks. This kind of validation is done by the queries using Bloom filters to check the probability of result existance in any other database session.

Memory used by Bloom filters are very minimum and negligible as shown below.
```
SQL> EXEC DBMS_RESULT_CACHE.MEMORY_REPORT(TRUE);
R e s u l t   C a c h e   M e m o r y   R e p o r t
[Parameters]
Block Size          = 1K bytes
Maximum Cache Size  = 10496K bytes (10496 blocks)
Maximum Result Size = 524K bytes (524 blocks)
[Memory]
Total Memory = 583144 bytes [0.013% of the Shared Pool]
... Fixed Memory = 25208 bytes [0.001% of the Shared Pool]
....... Memory Mgr = 208 bytes
....... Cache Mgr  = 256 bytes
....... Bloom Fltr = 2K bytes
.......  = 4088 bytes
....... RAC Cbk    = 6240 bytes
....... State Objs = 12368 bytes
... Dynamic Memory = 557936 bytes [0.013% of the Shared Pool]
....... Overhead = 131952 bytes
........... Hash Table    = 64K bytes (4K buckets)
........... Chunk Ptrs    = 24K bytes (3K slots)
........... Chunk Maps    = 12K bytes
........... Miscellaneous = 131952 bytes
....... Cache Memory = 416K bytes (416 blocks)
........... Unused Memory = 15 blocks
........... Used Memory = 401 blocks
............... Dependencies = 153 blocks (153 count)
............... Results = 248 blocks
................... SQL     = 67 blocks (67 count)
................... PLSQL   = 4 blocks (4 count)
................... CDB     = 130 blocks (104 count)
................... Invalid = 47 blocks (47 count)

PL/SQL procedure successfully completed.
```
In this database Bloom filter memory footprint is just 2 KB for the purpose of serving Result cache feature.

Monitoring
----------
Usage of Bloom filter can be monitored through,
1. Real time SQL monitoring
	This requires license and it provides greater detail of rows processed at each stage in the execution plan. Data reduction can be interpret by looking at the actual amount of data processed at each stage by aggregating accorss all the parallel slave processes, this can be correlate with the execution plan to find the impact of Bloom filters.
	
2. Querying ```V$PQ_TQSTAT```
	This view provides details of work ditribution among the parallel processes within the same session where parallel query has been executed. This limits the actual usage of this view in production environment as we can't see other sessions parallel query work distribution.
	
3. Querying ```V$SQL_JOIN_FILTER``` - (Provides data only for active Bloom filters)
	This view can be used to view the performance metrics of join filters. But there is a limitation, it provides information of only active join filters. Few interesting columns in this view like FILTERED and PROBED can be used to derive total number of rows which have been rejected due to Bloom filters.
	
Conclusion
----------
In this article we saw how Bloom filters are used at several places in Oracle to improve the sql reponse time drastically. Bloom filter can be managed by using hints like ```PX_JOIN_FILTER/NO_PX_JOIN_FILTER``` for controlling join filters, it can also be controlled using parameter ```"_bloom_filter_enabled"```. But there are no hints available to control Bloom pruning feature, only way to control is through parameter ```"_bloom_pruning_enabled"```. There is no manual intervention required for Bloom filter to be used, it gets create and used based on internal Optimizer calculations. In Exadata it plays a vital role as Bloom filters functions can be offloaded to cell servers. Bloom filters existance is high when using parallelism, but it also works in serial mode. Monitoring them at each sql level in production provide some insights on how well it is performing through view ```V$SQL_JOIN_FILTER```.
