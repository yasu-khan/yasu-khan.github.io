---
layout: post
title: Oracle - Adaptive Query Optimization(Adaptive Plans)
description: Oracle 12.1.0.1/12.1.0.2 Adaptive Plans part of Adaptive Query Optimization feature.
comments: true
keywords: Oracle 12.1.0.1 or 12.1.0.2, Oracle 12c, Adaptive Plans, Adaptive Query Optimization, statistics collector
---

Introduction
------------
Adaptive Plans is one of the new feature in introduced in 12c and is part of Adaptive Query Optimization feature. Adaptive Plans can be further categorized into two methods as Join method and Parallel distribution methods. In this article we will focus only on Adaptive Join method. 

Adaptive Join method is an mechanism to learn from its mistakes and got introduced into Optimizer to perform run-time adjustments for the execution plans when the derived plan through statistics is not optimal. Thus Optimizer will defer its decision of chosing the final join method to the execution phase of the sql. In earlier releases this was only possible at parse phase of the sql and once sql is into execution phase plan is fixed and it can't be changed. It is important to note that Adaptive plan will delay its decision of chosing final join method untill execution phase and introduces the new operation in the execution plan called as ```'STATISTICS COLLECTOR'``` which buffers the rows and switches from the default estimated join method when buffered rows are more than the inflection point. 

So it is not that executing join method will change to other join method on the fly, infact it is the decision of final join method which is evaluated at run-time. When final join method is decided then ```'STATISTICS COLLECTOR'``` will be disabled. 

Adaptive plan in action:
Table ```DEMO``` is having below structure along with index on column ```LOCATION```

```sql
SQL> desc demo
 Name                 Null?    Type
 -------------------- -------- ----------------------------
 SSN                           NUMBER
 LOCATION                      NUMBER
 DETAILS                       VARCHAR2(200)
```
The data distribution is as follows.
```sql
select count(*),location from demo group by location order by 1;
  COUNT(*)   LOCATION
---------- ----------
        10         10
       100         20
      1000         30
     10000         40
    100000         50
   1000000         60
  10000000         70
```

Now let's execute below sql and check its execution plan. Since ```LOCATION``` column data is skewed Optimizer will not be able to estimate the cardinality precisely and thus it may perform wrong join method. 

```sql
SELECT /*+ gather_plan_statistics */ count(*)
FROM   DEMO a,
       DEMO b
WHERE  a.ssn = b.ssn AND
       a.location = 40 AND
       b.location = 50; 

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------
SQL_ID  9w38rcsj3ph39, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   DEMO a,DEMO b WHERE  a.ssn = b.ssn AND a.location = 40 AND b.location = 50

Plan hash value: 2967406341
--------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name     | Starts | E-Rows | A-Rows |   A-Time   |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |          |      1 |        |      1 |00:00:00.64 |
|   1 |  SORT AGGREGATE                       |          |      1 |      1 |      1 |00:00:00.64 |
|*  2 |   HASH JOIN                           |          |      1 |   1587K|      0 |00:00:00.64 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|  10000 |00:00:00.01 |
|*  4 |     INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|  10000 |00:00:00.01 |
|   5 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|    100K|00:00:00.07 |
|*  6 |     INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|    100K|00:00:00.03 |
--------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("A"."SSN"="B"."SSN")
   4 - access("A"."LOCATION"=40)
   6 - access("B"."LOCATION"=50)

Note
-----
   - this is an adaptive plan
```

So the note section says this plan is the adaptive planand and HASH JOIN has been used as the final joining method, but we dont know yet what was the estimated plan and where did the ```'STATISTICS COLLECTOR'``` occur in this execution plan. Also note that Actual and Estimated rows are still way off, this concludes that Adaptive joins are meant only for deciding final join method but not for rectifying estimated cardinality. To get all these details we can use 'adaptive' as reporting type in ```dbms_xplan``` package. 

```sql
SQL> select * from table(dbms_xplan.display_cursor('9w38rcsj3ph39',null,'adaptive allstats'));

PLAN_TABLE_OUTPUT
-----------------
SQL_ID  9w38rcsj3ph39, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   DEMO a, DEMO b WHERE  a.ssn = b.ssn AND a.location = 40 AND b.location = 50

Plan hash value: 2967406341
-------------------------------------------------------------------------------------------------------
|   Id  | Operation                                | Name     | Starts | E-Rows | A-Rows |   A-Time   |
-------------------------------------------------------------------------------------------------------
|     0 | SELECT STATEMENT                         |          |      1 |        |      1 |00:00:00.64 |
|     1 |  SORT AGGREGATE                          |          |      1 |      1 |      1 |00:00:00.64 |
|  *  2 |   HASH JOIN                              |          |      1 |   1587K|      0 |00:00:00.64 |
|-    3 |    NESTED LOOPS                          |          |      1 |   1587K|  10000 |00:00:00.01 |
|-    4 |     NESTED LOOPS                         |          |      1 |        |  10000 |00:00:00.01 |
|-    5 |      STATISTICS COLLECTOR                |          |      1 |        |  10000 |00:00:00.01 |
|     6 |       TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|  10000 |00:00:00.01 |
|  *  7 |        INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|  10000 |00:00:00.01 |
|- *  8 |      INDEX RANGE SCAN                    | DEMO_IDX |      0 |   1587K|      0 |00:00:00.01 |
|- *  9 |     TABLE ACCESS BY INDEX ROWID          | DEMO     |      0 |      1 |      0 |00:00:00.01 |
|    10 |    TABLE ACCESS BY INDEX ROWID BATCHED   | DEMO     |      1 |   1587K|    100K|00:00:00.07 |
|  * 11 |     INDEX RANGE SCAN                     | DEMO_IDX |      1 |   1587K|    100K|00:00:00.03 |
-------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("A"."SSN"="B"."SSN")
   7 - access("A"."LOCATION"=40)
   8 - access("B"."LOCATION"=50)
   9 - filter("A"."SSN"="B"."SSN")
  11 - access("B"."LOCATION"=50)

Note
-----
   - this is an adaptive plan (rows marked '-' are inactive)
```

Adaptive report says Nested Loop join method was not chosen and ```'STATISTICS COLLECTOR'``` was placed as parent operation for line ID 6 and 7. According to Predicate information line ID 7 is having 'access' predicate for ```"A"."LOCATION"=40```. In principle ```'STATISTICS COLLECTOR'``` will buffer the rows from the index access path of ```DEMO_IDX``` for the ```LOCATION``` value 40 and if total number of buffered rows exceeds the inflection point then ```HASH JOIN``` will be chosed else ```NESTED LOOP``` join will be chosed as final join method. 

But what is the inflection point and how do I find it? Enable 10053 tracing and search for lines starting with AP/DP as shown below.

```
SQL> !egrep "^AP:|^DP:" cdbt0001_ora_21971.trc
AP: Checking validity for query block SEL$1, sqlid=9w38rcsj3ph39.
AP: Computing costs for inflection point at min value 0.00
AP: Using binary search for inflection point search
AP: Costing Nested Loops Join for inflection point at card 0.00
AP: Costing Hash Join for inflection point at card 0.00
AP: lcost=108275.29, rcost=108278.83
AP: Computing costs for inflection point at max value 1587301.43
AP: Costing Nested Loops Join for inflection point at card 1587301.43
AP: Costing Hash Join for inflection point at card 1587301.43
AP: lcost=85933997509.98, rcost=111444.49
AP: Searching for inflection point at value 1.00
AP: Costing Nested Loops Join for inflection point at card 793650.71
AP: Costing Hash Join for inflection point at card 793650.71
AP: lcost=42967052892.63, rcost=110652.44
AP: Searching for inflection point at value 793650.71
AP: Costing Nested Loops Join for inflection point at card 396825.36
AP: Costing Hash Join for inflection point at card 396825.36
AP: lcost=21483526445.56, rcost=110256.42
AP: Searching for inflection point at value 396825.36
AP: Costing Nested Loops Join for inflection point at card 198412.68
AP: Costing Hash Join for inflection point at card 198412.68
AP: lcost=10741817360.42, rcost=110058.41
....
....
....
AP: Costing Nested Loops Join for inflection point at card 48.44
AP: Costing Hash Join for inflection point at card 48.44
AP: lcost=2652780.29, rcost=108278.83
AP: Searching for inflection point at value 48.44
AP: Costing Nested Loops Join for inflection point at card 24.22
AP: Costing Hash Join for inflection point at card 24.22
AP: lcost=1353458.59, rcost=108278.83
AP: Searching for inflection point at value 24.22
AP: Costing Nested Loops Join for inflection point at card 12.11
AP: Costing Hash Join for inflection point at card 12.11
AP: lcost=703797.74, rcost=108278.83
AP: Searching for inflection point at value 12.11
AP: Costing Nested Loops Join for inflection point at card 6.06
AP: Costing Hash Join for inflection point at card 6.06
AP: lcost=378967.31, rcost=108278.83
AP: Searching for inflection point at value 6.06
AP: Costing Nested Loops Join for inflection point at card 3.03
AP: Costing Hash Join for inflection point at card 3.03
AP: lcost=216552.10, rcost=108278.83
AP: Searching for inflection point at value 3.03
AP: Costing Nested Loops Join for inflection point at card 1.51
AP: Costing Hash Join for inflection point at card 1.51
AP: lcost=162413.69, rcost=108278.83
AP: Searching for inflection point at value 1.51
AP: Costing Nested Loops Join for inflection point at card 0.76
AP: Costing Hash Join for inflection point at card 0.76
AP: lcost=108275.29, rcost=108278.83
AP: Costing Nested Loops Join for inflection point at card 0.76
DP: Found point of inflection for NLJ vs. HJ: card = 0.76
```

According to tracing information estimated join cardinality 1587301 has been halved every time and performed cost calculation of Hash join and Nested Loop join untill the other join exceeds the cost. In this case inflection point has been found at cardinality 0.76 which means if buffering of rows by ```'STATISTICS COLLECTOR'``` for 'access' predicate ```"A"."LOCATION"=40``` is more than 0.76 then HASH join will be chosen. Amount of work done to find the inflection point will depends upon the size of the tables and clustering factor of the index, and it increases linearly when table size and clustering factor of index delve each other to reach the inflection point.

NOTE: We can also use ```optimizer_adaptive_reporting_only``` parameter to control the reporting-only mode. When this parameter is enabled all the adaptive optimizations information will be gathered but will not be implemented. To view the report of adaptive optimizations we can use ```DBMS_XPLAN``` package.

Breaking point
--------------
This raises an interesting question, what happens if my data is volatile enough to change this inflection point ? 
Well, inflection point is calculated only at hard parse phase fo the sql, so if change in data is presumed to change the inflection point then hard parse has to occur else inflection point will not be re-calculated.

Let's continue with the previous example and see what happens if we delete all the rows having ```"A"."LOCATION"=40```

```sql
SQL> delete from demo where location=40;
10000 rows deleted.

SELECT /*+ gather_plan_statistics */ count(*)
FROM   DEMO a,
       DEMO b
WHERE  a.ssn = b.ssn AND
       a.location = 40 AND
       b.location = 50; 

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------
SQL_ID  9w38rcsj3ph39, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   DEMO a,DEMO b WHERE  a.ssn = b.ssn AND a.location = 40 AND b.location = 50

Plan hash value: 2967406341
--------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name     | Starts | E-Rows | A-Rows |   A-Time   |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |          |      1 |        |      1 |00:00:00.01 |
|   1 |  SORT AGGREGATE                       |          |      1 |      1 |      1 |00:00:00.01 |
|*  2 |   HASH JOIN                           |          |      1 |   1587K|      0 |00:00:00.01 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|      0 |00:00:00.01 |
|*  4 |     INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|      0 |00:00:00.01 |
|   5 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      0 |   1587K|      0 |00:00:00.01 |
|*  6 |     INDEX RANGE SCAN                  | DEMO_IDX |      0 |   1587K|      0 |00:00:00.01 |
--------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("A"."SSN"="B"."SSN")
   4 - access("A"."LOCATION"=40)
   6 - access("B"."LOCATION"=50)

Note
-----
   - this is an adaptive plan

```
Plan has not changed to use ```NESTED LOOP``` join method as there are no rows with location=40 and is less than inflection point cardinality 0.76, but if we force this sql to go for hard parse then ```STATISTICS COLLECTOR``` will detect this and switches the plan to use ```NESTED LOOP``` join.

```sql
SQL> alter system flush shared_pool;
SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));

PLAN_TABLE_OUTPUT
-----------------------------------------------------------------------------------------------------------
SQL_ID  9w38rcsj3ph39, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   DEMO a,DEMO b WHERE  a.ssn = b.ssn AND a.location = 40 AND b.location = 50

Plan hash value: 3074188767
---------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name     | Starts | E-Rows | A-Rows |   A-Time   |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |          |      1 |        |      1 |00:00:00.01 |
|   1 |  SORT AGGREGATE                        |          |      1 |      1 |      1 |00:00:00.01 |
|   2 |   NESTED LOOPS                         |          |      1 |   1587K|      0 |00:00:00.01 |
|   3 |    NESTED LOOPS                        |          |      1 |        |      0 |00:00:00.01 |
|   4 |     TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|      0 |00:00:00.01 |
|*  5 |      INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|      0 |00:00:00.01 |
|*  6 |     INDEX RANGE SCAN                   | DEMO_IDX |      0 |   1587K|      0 |00:00:00.01 |
|*  7 |    TABLE ACCESS BY INDEX ROWID         | DEMO     |      0 |      1 |      0 |00:00:00.01 |
---------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   5 - access("A"."LOCATION"=40)
   6 - access("B"."LOCATION"=50)
   7 - filter("A"."SSN"="B"."SSN")

Note
-----
   - this is an adaptive plan
```

This confirms that inflection point is calculated only at hard parse of the sql statement and thus contradicts this feature with higly volatile data which may influence inflection point. The threats from this feature in databases where data is higly volatile resembles similar to that long live bind variable peeking issue. So one of the best method to solve these kind of issues is to know your data well and then gain complete control over the cursors by modelling them which requires special care. Better to closely monitor the performance of sql's using adaptive plans, information can be found at many places and one of the important column is ```IS_RESOLVED_ADAPTIVE_PLAN``` in ```V$SQL/V$SQL_PLAN```.

Persisting Adaptive plan by using SQL Plan Baseline
---------------------------------------------------
Since adaptive plans doesn't persist its learning information we can use SPM to instruct sql's for using the ultimate adaptive plans without going through buffering of rows by ```'STATISTICS COLLECTOR'``` for finding the inflection point. Using SPM to fix the adaptive plans when you are using ```cursor_sharing=EXACT``` will enforce to create multiple baselines for each literal value which seems to be impractical, in such cases we can leverage the ```force_match``` option of Sql profile to moderate the plan for all the similar sql's which differs only by literal values. Infact Sql profile will use f```orce_match_signature``` when force_match option is enabled for the sql. Using band aid features like SPM, Sql profile are all not meant for providing permanent fix as it will go sour over the period of time and thus its an additional overhead for maintaining it. 

Lets implement SPM for previous sql and check whether buffering of rows will be done by ```STATISTICS COLLECTOR``` or not. 

```sql
DECLARE
return_var binary_integer;
BEGIN
 return_var :=	
 dbms_spm.load_plans_from_cursor_cache(
		sql_id          => '9w38rcsj3ph39', 
		plan_hash_value => 2967406341,
		fixed           => 'NO',
		enabled         => 'YES');
END;
/

SQL> select sql_text,enabled,LAST_EXECUTED,ADAPTIVE from dba_sql_plan_baselines;

SQL_TEXT                                           ENABLED LAST_EXECUTED    ADA
-------------------------------------------------- ------- ---------------- ---
SELECT /*+ gather_plan_statistics */ count(*)      YES                      NO

SELECT /*+ gather_plan_statistics */ count(*)
FROM   DEMO a,
       DEMO b
WHERE  a.ssn = b.ssn AND
       a.location = 40 AND
       b.location = 50;

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));	   
SQL_ID  9w38rcsj3ph39, child number 2
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   DEMO a,DEMO b WHERE  a.ssn = b.ssn AND a.location = 40 AND b.location = 50

Plan hash value: 2967406341
--------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name     | Starts | E-Rows | A-Rows |   A-Time   |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |          |      1 |        |      1 |00:00:00.27 |
|   1 |  SORT AGGREGATE                       |          |      1 |      1 |      1 |00:00:00.27 |
|*  2 |   HASH JOIN                           |          |      1 |   1587K|      0 |00:00:00.27 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|  10000 |00:00:00.22 |
|*  4 |     INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|  10000 |00:00:00.16 |
|   5 |    TABLE ACCESS BY INDEX ROWID BATCHED| DEMO     |      1 |   1587K|    100K|00:00:00.06 |
|*  6 |     INDEX RANGE SCAN                  | DEMO_IDX |      1 |   1587K|    100K|00:00:00.02 |
--------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("A"."SSN"="B"."SSN")
   4 - access("A"."LOCATION"=40)
   6 - access("B"."LOCATION"=50)

Note
-----
   - SQL plan baseline SQL_PLAN_7yjjgk7bkg32yf1102560 used for this statement

SQL> select sql_text,enabled,LAST_EXECUTED,ADAPTIVE from dba_sql_plan_baselines;
   
SQL_TEXT                                      ENABLED LAST_EXECUTED                ADA
--------------------------------------------- ------- ---------------------------- ---
SELECT /*+ gather_plan_statistics */ count(*) YES     07-JAN-16 04.09.45.000000 AM NO
```
                                                           
As you see SQL plan baseline has been used to use final adaptive plan without ```STATISTICS COLLECTOR``` operator. The Adaptive column in this view states whether this plan is adaptive or not when it was automatically captured. If adaptive plan is found for the sql already having baseline then adaptive column will be set to YES, and this adaptive plan will be tested and will be accepted if it performs better than existing baseline plans, at last adaptive column for this plan will set to NO since it is resolved plan and no longer an adaptive plan.

In case of Adaptive plan it doesn't mean that all the time taken by ```STATISTICS COLLECTOR``` for buffering the rows until final join method found is waste of time/resource and overhead. Because the buffered time will be negligible when compared to the penalty caused by using wrong join method with disastrous effect on the total response time of the sql. 

Plan hash value for Adaptive plans
----------------------------------
There is a new column ```FULL_PLAN_HASH_VALUE``` in ```V$SQL/V$SQL_PLAN/V$SQLAREA/V$SQL_PLAN_STATISTICS``` which represents the plan hash value for adaptive plans. We need to be careful enough and not get confused as each ```PLAN_HASH_VALUE``` can have multiple ```FULL_PLAN_HASH_VALUE``` and viceversa. 

In case of sql using SQL plan baseline ```FULL_PLAN_HASH_VALUE``` will be the result of ```PHV2``` and this ```PHV2``` is derived from ```OTHER_XML``` column of ```V$SQL_PLAN```. SPM will always considers ```PHV2``` of the plan and then compare it with stored ```'Plan ID'``` to check the possibility of reproducing accepted plan stored in baseline, if ```PHV2``` and ```"Plan ID"``` doesn't match then baseline will be ignored. SPM uses ```PHV2``` but not ```PLAN_HASH_VALUE``` because ```PHV2``` is independent of the intermediate objects name conventions which Oracle creates. In case of SQL profile/patches it never uses ```PHV2```, it just applies the hints stored to come up with the execution plan. SPM also applies hints but later on compares the ```PHV2``` and stored ```'Plan ID'```. Interesting thing is all these hints are stored in sqlobj$data but used by SPM/Profile/Patches in their own distintive way.

Below is the report derived by using different scenarios to generate multiple combination of ```FULL_PLAN_HASH_VALUE``` and ```PLAN_HASH_VALUE```.

```sql
SELECT p.sql_id,
       p.child_number,
	   p.plan_hash_value,
       t.phv2,
       p.full_plan_hash_value
FROM   V$SQL_PLAN p,
       XMLTABLE(' for $i in /other_xml/info
                  where $i/@type eq "plan_hash_2"
                  return $i' 
				  passing xmltype(p.other_xml) 
				  COLUMNS phv2 NUMBER path '/') t
WHERE  p.sql_id = '9w38rcsj3ph39' AND
       p.other_xml IS NOT NULL 

SQL_ID        CHILD_NUM PLAN_HASH_VALUE       PHV2 FULL_PLAN_HASH_VALUE 
------------- --------- --------------- ---------- -------------------- 
9w38rcsj3ph39         0      2967406341 4044367200           1001692354 -> Adaptive plan using HASH JOIN
9w38rcsj3ph39         0      3074188767 2183012257           1001692354 -> Adaptive plan using NESTED LOOP JOIN
9w38rcsj3ph39         1      2967406341 4044367200           4044367200 -> Not Adaptive using HASH JOIN
9w38rcsj3ph39         2      2967406341 4044367200           4044367200 -> Not Adaptive but using baseline(HASH JOIN)
```

In above report we can obeserve that for Adaptive plans ```FULL_PLAN_HASH_VALUE``` is same when ```PLAN_HASH_VALUE/PHV2``` is different in terms of joining methods(Hash and Nested), this concludes ```FULL_PLAN_HASH_VALUE``` is calculated for all the access operations in the execution plan regardless of whether they were active or not. To avoid confusion consider ```FULL_PLAN_HASH_VALUE``` for a child cursor only when it is Adaptive plan.

Conclusion
----------
This Adaptive Query Optimization feature learns from its mistakes and choses the most efficient join method between two datasets by buffering few records until inflection point is reached during runtime of the sql. Adaptive join method will influence only the join methods(HASH and NESTED) but not join orders like cardinality feedback or ACS does. In upcoming post we will discuss about how Adaptive Query Optimization feature influences the Parallel ditribution methods.

