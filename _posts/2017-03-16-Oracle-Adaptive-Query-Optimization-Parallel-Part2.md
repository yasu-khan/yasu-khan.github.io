---
layout: post
title: Oracle - Adaptive Query Optimization(Parallel Distribution Methods)
description: Oracle 12.1.0.1/12.1.0.2 Parallel Distribution Methods part of Adaptive Query Optimization feature.
comments: true
keywords: Oracle 12.1.0.1 or 12.1.0.2, Oracle 12c, Parallel Distribution, Adaptive Query Optimization, statistics collector, broadcast
---

Introduction
------------
In previous [article](https://yasu-khan.github.io/Oracle-Adaptive-Optimization-Adaptive-Plans-Part1) we discussed about 12c new feature Adaptive plans related to Join Methods, in this article we will discuss about Adaptive plans related to Parallel Distribution Methods. 

Usually when parallel sql statements are executed then Optimizer has to decide on the distribution method for the parallel processes involved for performing tasks such as joins, sorts or any king of aggregation. Ditribution method is mainly based on estimated number of rows, so parallel sql statement performance is directly proportional to accuracy of cardinality estimation. If estimation goes wrong for whatsoever reason then many of the parallel server processes will be underutilized and thus contradicts the main strenght of parallelism by not distributing the work evenly for all the parallel server processes involved. 

12c introduced new feature called 'Adaptive Parallel Distribution Method' which inturn works similarly to 'Adaptive Join' but not exactly due to major difference in mechanism of these two methods. Adaptive Parallel Distribution Method uses new ```HYBRID HASH``` distribution and it defers its distribution method until sql execution phase, during this phase ```'STATISTICS COLLECTOR'``` will be placed to buffer the rows at build table and if total number of buffered rows exceeds the threshold then it will use HASH distribution and if buffered rows doesn't exceed the threshold then distribution method will be converted from HASH to BROADCAST method. At the same time for probe table it will broadcast using ```RANDOM/ROUND ROBIN``` method if build table is using ```BROADCAST``` method, otherwise it will use HASH broadcasting.

What is the threshold and how does it gets calculated ?
-------------------------------------------------------
Simple, it is 2 * DOP. Say for example sql is executed with 4 parallelism and ```'STATISTICS COLLECTOR'``` has buffered 9 rows, in this case it has exceeds the threshold (2*4=8) and thus uses HASH distribution for the parallel server processes, if it doesn't exceeds the threshold then it will use ```ROADCAST``` distribution method.

Adaptive parallel distribution feature is mainly meant for HASH and MERGE joins, parallel distribution method is evaluated at every execution phase of the same parallel sql and thus this feature differs from 'Adaptive Join' where final join method is decided only at hard parse of the sql after which subsequent sql's using the same plan due to soft parse will not have ```'STATISTICS COLLECTOR'``` in play to decide the best join method. Adaptive parallel distribution feature allows 'build table' of the HASH join to decide either BROADCAST or HASH parallel distribution method for every execution of this sql, here 'build table' is in context with HASH join where cardinality is low when compared to other joining table. It is important to emphazie that this feature works only with inner-join/equi-join since HASH/MERGE join is possible only with these types of joining methods.

Adaptive Parallel Distribution Method in action
-----------------------------------------------
Lets take ```CUSTOMER``` and ```COUNTRIES``` table from ```SH``` schema and perform join with different degree of parallelism to observe the different parallel distribution methods in real time. Few hints are in place to simulate ```HYBRID HASH``` distribution as these tables are of very small in size. Lets proceed with parallelism of 4 and see how data has been distributed among the parallel processes.

```sql
select  /*+ 
		leading(coun) 
		no_swap_join_inputs(coun) 
		pq_distribute(cust hash hash) 
		parallel(4) 
		*/ 
		count(*)
from   	sh.customers cust, sh.countries coun
where 	cust.country_id = coun.country_id
and 	cust.cust_city='Los Angeles'   
and    	cust.cust_state_province='CA'
/

select * from table(dbms_xplan.display_cursor(null,null,'typical -bytes -cost last'));

Plan hash value: 2610597691
--------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name      | Rows  | Time     |    TQ  |IN-OUT| PQ Distrib |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |           |       |          |        |      |            |
|   1 |  SORT AGGREGATE              |           |     1 |          |        |      |            |
|   2 |   PX COORDINATOR             |           |       |          |        |      |            |
|   3 |    PX SEND QC (RANDOM)       | :TQ10002  |     1 |          |  Q1,02 | P->S | QC (RAND)  |
|   4 |     SORT AGGREGATE           |           |     1 |          |  Q1,02 | PCWP |            |
|*  5 |      HASH JOIN               |           |   952 | 00:00:01 |  Q1,02 | PCWP |            |
|   6 |       PX RECEIVE             |           |    23 | 00:00:01 |  Q1,02 | PCWP |            |
|   7 |        PX SEND HYBRID HASH   | :TQ10000  |    23 | 00:00:01 |  Q1,00 | P->P | HYBRID HASH|
|   8 |         STATISTICS COLLECTOR |           |       |          |  Q1,00 | PCWC |            |
|   9 |          PX BLOCK ITERATOR   |           |    23 | 00:00:01 |  Q1,00 | PCWC |            |
|* 10 |           TABLE ACCESS FULL  | COUNTRIES |    23 | 00:00:01 |  Q1,00 | PCWP |            |
|  11 |       PX RECEIVE             |           |   952 | 00:00:01 |  Q1,02 | PCWP |            |
|  12 |        PX SEND HYBRID HASH   | :TQ10001  |   952 | 00:00:01 |  Q1,01 | P->P | HYBRID HASH|
|  13 |         PX BLOCK ITERATOR    |           |   952 | 00:00:01 |  Q1,01 | PCWC |            |
|* 14 |          TABLE ACCESS FULL   | CUSTOMERS |   952 | 00:00:01 |  Q1,01 | PCWP |            |
--------------------------------------------------------------------------------------------------
```

In above execution plan Optimizer has considered using ```HYBRID HASH``` distribution method at line Id 7 and 12, but ```STATISTICS COLLECTOR``` has been placed at lined Id 8 for table ```COUNTRIES``` which is having low cardinality and thus 'build table' of the ```HASH JOIN```. Ideally 'build table' is the smallest of the two tables (between two row sources after the application of predicates) should be listed first below the ```HASH JOI```N line in the execution plan. So ```HYBRID HASH``` operation can decide to use either BROADCAST or HASH distribution depending upon rows buffered by ```STATISTICS COLLECTOR```, if buffered rows exceeds threshold it will use HASH distribution else BROADCAST. In this case threshold is 2*4=8 and buffered rows are 23 (Lined Id 10) which exceeds threshold and thus it use HASH distribution. One of the best way to see parallel queries data distribution is by querying ```V$PQ_TQSTAT```, but this view gets populated only in private memory region of Query Co-ordinator if parallel query has succeded with any issues. Thus we can't query the details of other sessions by using ```V$PQ_TQSTAT```. For educational purpose we will query this view right after executing the parallel query in the same session to get more details about the new HYBRID HASH distribution method .

```sql
select tq_id,
       server_type,
       instance,
       process,
       num_rows
from   v$pq_tqstat
order  by dfo_number,tq_id,server_type desc,instance,process 
/
     TQ_ID SERVER_TYP   INSTANCE PROCESS    NUM_ROWS
---------- ---------- ---------- -------- ----------
         0 Producer            1 P002             23
                                 P003              0
                               2 P002              0
                                 P003              0

           Consumer            1 P000              7
                                 P001              3
                               2 P000              5
                                 P001              8

         1 Producer            1 P002            144
                                 P003            158
                               2 P002            292
                                 P003            338

           Consumer            1 P000              0
                                 P001              0
                               2 P000            932
                                 P001              0

         2 Producer            1 P000              1
                                 P001              1
                               2 P000              1
                                 P001              1

           Consumer            2 QC                4
```

TQ_ID corresponds to Table Queue in the execution plan named as ```:TQ10000```, ```:TQ10001```, ```:TQ10002```, data distribution among Producer and Consumer within each Table Queue can be viewed as shown in above output. In this example ```STATISTICS COLLECTOR``` is placed in Table Queue ```:TQ10000``` and as per above output we can easily conclude that HYBRID HASH has used HASH distribution method to distribute data from Producer to Consumer. 

Let's re-execute the same sql with higher degree of parallelism to see which data distribution method will be chosen by HYBRID HASH when threshold is exceeded.

```sql
select  /*+ 
		leading(coun) 
		no_swap_join_inputs(coun) 
		pq_distribute(cust hash hash) 
		parallel(12) 
		*/ 
		count(*)
from   	sh.customers cust, sh.countries coun
where 	cust.country_id = coun.country_id
and 	cust.cust_city='Los Angeles'   
and    	cust.cust_state_province='CA'
/

select tq_id,
       server_type,
       instance,
       process,
       num_rows
from   v$pq_tqstat
order  by dfo_number,tq_id,server_type desc,instance,process 
/
     TQ_ID SERVER_TYP   INSTANCE PROCESS    NUM_ROWS
---------- ---------- ---------- -------- ----------
         0 Producer            1 P006            276
                                 P007              0
                                 P008              0
                                 P009              0
                                 P010              0
                                 P011              0
                               2 P006              0
                                 P007              0
                                 P008              0
                                 P009              0
                                 P010              0
                                 P011              0

           Consumer            1 P000             23
                                 P001             23
                                 P002             23
                                 P003             23
                                 P004             23
                                 P005             23
                               2 P000             23
                                 P001             23
                                 P002             23
                                 P003             23
                                 P004             23
                                 P005             23

         1 Producer            1 P006            101
                                 P007             99
                                 P008             80
                                 P009             80
                                 P010             77
                                 P011            101
                               2 P006             79
                                 P007             84
                                 P008             98
                                 P009             41
                                 P010             21
                                 P011             71

           Consumer            1 P000             83
                                 P001             83
                                 P002             82
                                 P003             81
                                 P004             81
                                 P005             77
                               2 P000             77
                                 P001             76
                                 P002             74
                                 P003             73
                                 P004             73
                                 P005             72

         2 Producer            1 P000              1
                                 P001              1
                                 P002              1
                                 P003              1
                                 P004              1
                                 P005              1
                               2 P000              1
                                 P001              1
                                 P002              1
                                 P003              1
                                 P004              1
                                 P005              1

           Consumer            1 QC               12
```

By checking data distribution among Producer and Consumer for Table Queue ```:TQ10000``` we can conclude that 23 rows has been BROADCAST to all the Consumers as buffered 23 rows is less than the calculated threshold(2*12=24). 


Automatic detection of data skewness
------------------------------------
Let me copy the previous result of ```V$PQ_TQSTAT``` again here to show you the imbalance in data distribution.

```
     TQ_ID SERVER_TYP   INSTANCE PROCESS    NUM_ROWS
---------- ---------- ---------- -------- ----------
         0 Producer            1 P002             23
                                 P003              0
                               2 P002              0
                                 P003              0

           Consumer            1 P000              7
                                 P001              3
                               2 P000              5
                                 P001              8

         1 Producer            1 P002            144
                                 P003            158
                               2 P002            292
                                 P003            338

           Consumer            1 P000              0
                                 P001              0
                               2 P000            932
                                 P001              0

         2 Producer            1 P000              1
                                 P001              1
                               2 P000              1
                                 P001              1

           Consumer            2 QC                4
```

In Table Queue ```:TQ10001``` data distribution by all the Producers were done to one single Consumer which stands against fundamentals of parallelism as all other Consumers were idle without any work load balance among all the Consumers. This is due to data skewness in join column COUNTRY_ID of CUSTOMERS table. By default parallel query was not aware of this data skewness, hence the data distribution method was appropriate. This causes servere limitation for scaling parallel queries. 

In 12c parallel queries can detect this data skewness automatically when histogram exists on joining column ```COUNTRY_ID```. Histogram will be used by parallel query to check if there is any data skewness and if skewness exists then recursive queries will be fired behind the scene to gather all the popular values and store it in the cursor. When HYBRID HASH operation has to decide on data distribution method it will check with these popular values stored in the cursor with the data at run-time and performs BROADCAST distribution only for the matching popular values along with HASH distribution for all other non popular values, similarly for other join row source it will perform RANDOM/ROUND ROBIN if data matches with popular values stored in the cursor else it will use HASH for non-popular values.

This feature can also be triggred without Histogram by using hint ```PQ_SKEW```, but there is slight change in the way it works when capturing the popular values through recursive queries.

There are few pre-requisites for this feature to work, the joining column should be a table(it can't be view) and also result set of another join disables this feature due to limitation of single join condition(predicates). This feature doesn't support MERGE joins. Histogram should exists on joining column, this feature can also be triggered without histrogram by using ```PQ_SKEW``` hint.

Detection of skewness by Histogram
----------------------------------
Lets create histogram on joining column ```COUNTRY_ID``` for table ```CUSTOMERS```.

```sql
exec dbms_stats.gather_table_stats('SH','CUSTOMERS',estimate_percent=>DBMS_STATS.AUTO_SAMPLE_SIZE,no_invalidate=>FALSE,method_opt=>'for columns COUNTRY_ID  size 254',cascade=>TRUE);	 
```

Execute same sql and investigate if Adaptive parallel query has detected data skewness in column ```COUNTRY_ID``` by checking its execution plan.

```sql
select  /*+ 
		gather_plan_statistics 
		leading(coun) 
		no_swap_join_inputs(coun) 
		pq_distribute(cust hash hash) 
		parallel(4) 
		*/ 
		count(*)
from   	sh.customers cust, sh.countries coun
where 	cust.country_id = coun.country_id
and 	cust.cust_city='Los Angeles'   
and    	cust.cust_state_province='CA'
/

select * from table(dbms_xplan.display_cursor(null,null,'typical -bytes -cost last'));
------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name      | Rows  | Time     |    TQ  |IN-OUT| PQ Distrib |
------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |           |       |          |        |      |            |
|   1 |  SORT AGGREGATE                  |           |     1 |          |        |      |            |
|   2 |   PX COORDINATOR                 |           |       |          |        |      |            |
|   3 |    PX SEND QC (RANDOM)           | :TQ10002  |     1 |          |  Q1,02 | P->S | QC (RAND)  |
|   4 |     SORT AGGREGATE               |           |     1 |          |  Q1,02 | PCWP |            |
|*  5 |      HASH JOIN                   |           |   952 | 00:00:01 |  Q1,02 | PCWP |            |
|   6 |       PX RECEIVE                 |           |    23 | 00:00:01 |  Q1,02 | PCWP |            |
|   7 |        PX SEND HYBRID HASH       | :TQ10000  |    23 | 00:00:01 |  Q1,00 | P->P | HYBRID HASH|
|   8 |         STATISTICS COLLECTOR     |           |       |          |  Q1,00 | PCWC |            |
|   9 |          PX BLOCK ITERATOR       |           |    23 | 00:00:01 |  Q1,00 | PCWC |            |
|* 10 |           TABLE ACCESS FULL      | COUNTRIES |    23 | 00:00:01 |  Q1,00 | PCWP |            |
|  11 |       PX RECEIVE                 |           |   952 | 00:00:01 |  Q1,02 | PCWP |            |
|  12 |        PX SEND HYBRID HASH (SKEW)| :TQ10001  |   952 | 00:00:01 |  Q1,01 | P->P | HYBRID HASH|
|  13 |         PX BLOCK ITERATOR        |           |   952 | 00:00:01 |  Q1,01 | PCWC |            |
|* 14 |          TABLE ACCESS FULL       | CUSTOMERS |   952 | 00:00:01 |  Q1,01 | PCWP |            |
------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - access("CUST"."COUNTRY_ID"="COUN"."COUNTRY_ID")
  10 - access(:Z>=:Z AND :Z<=:Z)
  14 - access(:Z>=:Z AND :Z<=:Z)
       filter(("CUST"."CUST_CITY"='Los Angeles' AND "CUST"."CUST_STATE_PROVINCE"='CA'))
```

At line ID 12 we could see that skewness has been detected ```'HYBRID HASH (SKEW)'``` due to presence of histogram, this ensures data will get distributed uniformly accross all the parallel servers. This is acomplished by using BROADCAST distribution for popular values and HASH distribution for all other non popular values at 'build table' side and at 'probe table' by using RANDOM/ROUND ROBIN for popular values and HASH for all other non-popular values. Lets see how data has been distributed by querying ```V$PQ_TQSTAT``` view.

```sql	   
select tq_id,
       server_type,
       instance,
       process,
       num_rows
from   v$pq_tqstat
order  by dfo_number,tq_id,server_type desc,instance,process 
/	   
     TQ_ID SERVER_TYP   INSTANCE PROCESS    NUM_ROWS
---------- ---------- ---------- -------- ----------
         0 Producer            1 P002             26
                                 P003              0
                               2 P002              0
                                 P003              0

           Consumer            1 P000              8
                                 P001              5
                               2 P000              6
                                 P001              7

         1 Producer            1 P002            280
                                 P003            173
                               2 P002            279
                                 P003            200

           Consumer            1 P000            234
                                 P001            233
                               2 P000            233
                                 P001            232

         2 Producer            1 P000              1
                                 P001              1
                               2 P000              1
                                 P001              1

           Consumer            2 QC                4
```

In Table Queue 1 Producers have distributed the data uniformly for all the Consumers and hence scaling of parallel queries for skewed data is done automatically.

How does values are considered to be popular ?
----------------------------------------------
Let's now look under the hood to see how this is acomplished, by 10053 tracing we could see that Oracle calculates it as shown below.

```
skewRatio:10, skewMinFreq:30, minNDV:19, skewThreshold:0.526316
ind:0, csel:0.010757, skew count:0
ind:1, csel:0.140180, skew count:0
ind:2, csel:0.012829, skew count:0
ind:3, csel:0.036216, skew count:0
ind:4, csel:0.007261, skew count:0
ind:5, csel:0.014973, skew count:0
ind:6, csel:0.014991, skew count:0
ind:7, csel:0.147261, skew count:0
ind:8, csel:0.006901, skew count:0
ind:9, csel:0.036739, skew count:0
ind:10, csel:0.069063, skew count:0
ind:11, csel:0.011243, skew count:0
ind:12, csel:0.004396, skew count:0
ind:13, csel:0.012757, skew count:0
ind:14, csel:0.001351, skew count:0
ind:15, csel:0.001640, skew count:0
ind:16, csel:0.136162, skew count:0
ind:17, csel:0.333694, skew count:1
ind:18, csel:0.001586, skew count:1
Skewed value count:1 scaling:0 degree:4
kkopqSkewInfo: Query:SELECT /* DS_SKEW */ /*+ opt_param('parallel_execution_enabled', 'false') dynamic_sampling(0) no_sql_tune no_monitoring  */ * FROM (SELECT SYS_OP_COMBINED_HASH("COUNTRY_ID"), COUNT(*) CNT, TO_CHAR("COUNTRY_ID") FROM "SH"."CUSTOMERS" SAMPLE(99.900000) GROUP BY "COUNTRY_ID" ORDER BY CNT DESC) WHERE ROWNUM <= 1

*** 2016-01-12 07:15:10.486
skewHashVal:7795997631687905384 count:18506 to_charVal:52790
```

So there are few metrics(skewRatio,skewMinFreq,minNDV,skewThreshold) to which data will be compared to identify whether particular value can be considered as popular or not. SkewRatio is derived from parameter ```_px_join_skew_ratio``` having default value 10, skewMinFreq is derived from parameter ```_px_join_skew_minfreq``` having default value 30. This can be interpret as value should be present in atleast 10 buckets of histogram OR value has to be popular for more than 30% of total data. minNDV is total number of buckets existing in histogram. skewThreshold is calculated as 1/minNDV*skewRatio, so in our case it is 1/19*10=0.00526316 and thus it is 0.526316 %. After detecting popular values it stores them into the cursor as (skewHashVal:7795997631687905384 count:18506 to_charVal:52790) for future reference. If data is volatile enough to change the skewness then any soft parse/reuse of the same cursor will not detect this new skewness along with new popular values.

Detection of skewness by PQ_SKEW hint
-------------------------------------
```PQ_SKEW``` hint can be used to trigger this feature without using histogram provided all the pre-requisites are met. It works almost similar to the way it works when histogram is present. But when we use this hint it doesn't follow the rules of metrics(skewRatio,skewMinFreq,minNDV,skewThreshold), thus recursive queries are fired to find all the popular values as shown below in 10053 trace.

```
Skewed value count:4 scaling:1 degree:4
kkopqSkewInfo: Query:SELECT /* DS_SKEW */ /*+ opt_param('parallel_execution_enabled', 'false') dynamic_sampling(0) no_sql_tune no_monitoring  */ * FROM (SELECT SYS_OP_COMBINED_HASH("COUNTRY_ID"), COUNT(*) CNT, TO_CHAR("COUNTRY_ID") FROM "SH"."CUSTOMERS" SAMPLE(99.900000) GROUP BY "COUNTRY_ID" ORDER BY CNT DESC) WHERE ROWNUM <= 4
skewHashVal:7795997631687905384 count:18497 to_charVal:52790
skewHashVal:4497045456490979659 count:8159 to_charVal:52776
skewHashVal:5633529705774701552 count:7773 to_charVal:52770
skewHashVal:5464977823951815750 count:7554 to_charVal:52789
```

But not all the popular values have been detected as we guess, in fact popular values detection is limited to 4 in this case which is nothing but degree of parallelism for this query. So limitation popular value detection is directly proportional to degree of parallelism of the query(Skewed value count:4 scaling:1 degree:4). There is also an hint ```NO_PQ_SKEW``` which can be used to disable this feature. 

Conclusion
----------
Adaptive parallel broadcasting method is one of major improvement in parallelism area which allows to scale parallel queries at higher level by evenly distributing work load among all the parallel server processes. Also it is sophisticated enough to detect the skewness in the data and prevent it from un-evenly distribution of data among parallel processes by fluctutating the broadcasting methods meant for popular and non-popular values. There are few limitations for this feature to work but in future this feature may get more robust and mature. 