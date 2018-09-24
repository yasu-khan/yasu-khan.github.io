---
layout: post
title: Oracle - Forensic investigation on SQL performance degradation
description: Forensic investigation on sql performance degradation
comments: true
keywords: Oracle, performance, dbms_stats, cardinality misestimate
---

Objective was to perform root cause analysis for a batch job which had degradation in performance. Usually it becomes more strenuous when we are assinged to work on a issue which had occurred in past. But still its going to be interesting and challening work to drill down onto the issue and try to connect the dots by collecting the proofs as much as we can. Some strict guidelines were to not perform any type of change on any object in this database while investigating this issue.

Below were the few inputs collected from Application team:
* Their batch job was hung at the table creation step for about an hour.
* It got hung twice approximately at 6:30 and 7:30 AM on August 5 2015.

So my intial task was to find the sql_id and then try to pull out the history information of this sql_id. As this sql was hung for about an hour I was sure that I can find the sql run time details from ASH using ```V$ACTIVE_SESSION_HSITORY```, if not then I can go for ASH data present on disk ```DBA_HIST_ACTIVE_SESS_HISTORY```. Easiest entry point column into ASH would be ```SQL_OPCODE``` as it stores the code for each type of sql commands which got executed, in my case it would be ```'CREATE TABLE'``` and the code is 1 according to table ```AUDIT_ACTIONS```. I have filtered the data further by using application username who was running this batch job, in this case user_id is 133 according to dba_users for username APP_USER.

```
SQL> select min(sample_time) from V$ACTIVE_SESSION_HISTORY;
  
MIN(SAMPLE_TIME)
---------------------------------------------------------------------------
11-AUG-15 11.36.30.659 PM 
```


Information in ```v$active_session_history``` is flushed for August 5th, the next hope would be on AWR table of ```ASH(DBA_HIST_ACTIVE_SESS_HISTORY)```. Keep in mind that this view contains 1 out of 10 sample of ```V$ACTIVE_SESSION_HSITORY```, due to which we may lose the precision of data. 

```sql
SQL> l
  1  select to_char(ash.sample_time,'DD-MON-YY HH24') Hour, aud.name type, count(distinct ash.sql_exec_id) times, ash.sql_id
  2  from dba_hist_active_sess_history ash,
  3       audit_actions aud
  4  where SQL_ID is not NULL
  5  and ash.sql_opcode=aud.action
  6  and ash.sql_opcode=1
  7  and ash.user_id=133
  8  and ash.sample_time between to_date('05-AUG-15 00:00:00','DD-MON-YY HH24:MI:SS') and to_date('05-AUG-15 23:00:00','DD-MON-YY HH24:MI:SS')
  9  group by to_char(ash.sample_time,'DD-MON-YY HH24'), aud.name, ash.sql_id
 10* order by 1
SQL> /

HOUR         TYPE                              TIMES SQL_ID
------------ ---------------------------- ---------- -------------
05-AUG-15 02 CREATE TABLE                          1 09auwxtvq780f

05-AUG-15 03 CREATE TABLE                          2 21xmysqafv3mb
             CREATE TABLE                          1 cx6k6fc8tg3tu

05-AUG-15 04 CREATE TABLE                          1 cx6k6fc8tg3tu

05-AUG-15 05 CREATE TABLE                          1 09auwxtvq780f
             CREATE TABLE                          1 35z8hhckmx6uv

05-AUG-15 06 CREATE TABLE                          0 059ns4nryk5k1
             CREATE TABLE                          1 1wswrbdvqqczz
             CREATE TABLE                          1 9ays2128qyxc8

05-AUG-15 07 CREATE TABLE                          1 09auwxtvq780f
             CREATE TABLE                          2 1wswrbdvqqczz

05-AUG-15 08 CREATE TABLE                          1 09auwxtvq780f
             CREATE TABLE                          1 1wswrbdvqqczz

05-AUG-15 22 CREATE TABLE                          1 02mkr7hj1v85y
             CREATE TABLE                          1 09auwxtvq780f
             CREATE TABLE                          1 1wswrbdvqqczz
```


I was able to filter out the rows as much as possible and come up with few sql_id's. Upon further examination of the sql_text of all these sql_id's I was able to get the exact sql_id. Usually most of the DBA's try to find the sql_id by searching sql text in ```V$SQL``` or ```DBA_HIST_SQLTEXT``` using like operator, but I believe its the bad practice to follow though its acceptable to do it on ```DBA_HIST_SQLTEXT``` when compared to ```V$SQL```.

I this case sql_id is '1wswrbdvqqczz' and its sql text information is as shown below.
```sql
SQL> select sql_text from DBA_HIST_SQLTEXT where sql_id='1wswrbdvqqczz';

SQL_TEXT
--------------------------------------------------------------------------------
CREATE TABLE TMP_STOCK
AS
SELECT A.ABID ABID , A.CLEADATE STK_CLEADATE,
         LN((A.PRICVALUE/ B.PRICVALUE)) RN, RANK() OVER(PARTITION BY A.ABID ORDER BY  A.CLEADATE DESC) RNK
   FROM MONTHLY_DATAF A, MONTHLY_DATAF B, YEARLY_DATA C
  WHERE A.ABID = B.ABID
    AND A.ABID = C.ABID
    AND C.STATUS='AC'
    AND B.CLEADATE = ADD_MONTHS(A.CLEADATE, -1)
    AND A.CLEADATE <= TO_DATE(20150630, 'YYYYMMDD')
    AND A.CLEADATE > ADD_MONTHS (TO_DATE(20150630, 'YYYYMMDD'), -240)
```

As the literals were clearly visible its better to check the CURSOR_SHARING parameter value, so we can keep in mind that sql_id may vary for each different literal value.	

``` 
SQL> show parameter CURSOR_SHARING

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cursor_sharing                       string      EXACT
```

Next step would be to find elapsed time of this sql on 5th August. Its more easier to find out as ```SQL_EXEC_ID``` column has been included in the ASH views which will help us to differentiate each execution of the same sql.

```sql
SQL> l
  1  select sql_id,sql_exec_id,sql_plan_hash_value,count(*) time,min(sample_time) SQL_START_TIME,max(sample_time) SQL_END_TIME
  2  from DBA_HIST_ACTIVE_SESS_HISTORY
  3  where SQL_ID='1wswrbdvqqczz'
  4  and user_id=133
  5  and sample_time between to_date('05-AUG-15 00:00:00','DD-MON-YY HH24:MI:SS') and to_date('05-AUG-15 23:00:00','DD-MON-YY HH24:MI:SS')
  6  group by sql_id,sql_exec_id,sql_plan_hash_value
  7* order by 3
SQL> /

SQL_ID        SQL_EXEC_ID SQL_PLAN_HASH_VALUE       TIME SQL_START_TIME                                SQL_END_TIME
------------- ----------- ------------------- ---------- --------------------------------------------- ---------------------------------------------
1wswrbdvqqczz    16777217          1100610021         11 05-AUG-15 08.39.12.387 AM                     05-AUG-15 08.40.53.305 AM
1wswrbdvqqczz    16777218          1100610021         13 05-AUG-15 10.56.48.388 PM                     05-AUG-15 10.58.49.063 PM
1wswrbdvqqczz    16777216          2482295346        376 05-AUG-15 07.27.45.562 AM                     05-AUG-15 08.30.37.779 AM
1wswrbdvqqczz    16777241          2482295346        470 05-AUG-15 06.00.52.313 AM                     05-AUG-15 07.19.41.672 AM
```

By grouping on ```SQL_ID,SQL_EXEC_ID and SQL_PLAN_HASH_VALUE``` we can get the elapsed time and sql plan it was using along with sql execution start and end time. As per above output we can clearly see that there was plan change from 1100610021 to 2482295346 and the worst performed plan was 2482295346. Now we have to find what has caused this plan change and where most of the response time was spent out of 470 seconds, so the best way to get the breakdown of total response time will be by checking total seconds spent on each type of wait event.

```sql
  1  select event,session_state,count(*)
  2  from dba_hist_active_sess_history
  3  where sql_id='1wswrbdvqqczz' and sql_exec_id=16777216
  4  group by event,session_state
  5* order by 3
SQL> /

EVENT                                                            SESSION   COUNT(*)
---------------------------------------------------------------- ------- ----------
gc current block 2-way                                           WAITING          1
                                                                 ON CPU         469
```

Interesting... 99% of elapsed time this sql was not waiting on any wait event but busy doing some work by consuming CPU cycles for about 470 seconds. Seems like this bad plan was doing unnecessarily lot of unwanted work due to bad choice of join method. Now its time to compare bad plan with the best plan and see what else we can derive out of it to justify the reason of plan change.

```sql
SQL> select * from table(dbms_xplan.display_cursor('1wswrbdvqqczz',null,null));

PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
SQL_ID: 1wswrbdvqqczz cannot be found
```

Cursor is not availabe in memory, but ofcourse we can get it from AWR. One of the biggest disadvantage of pulling execution plan details from AWR is missing 'Predicate Information', this is most valuable information for reading the execution plan and figuring out where the Filter and Access of data has been done. But here in this case we do not have any option rather than pulling it from AWR due to cursor being flushed from memory.

```sql
SQL> select * from table(dbms_xplan.display_awr('1wswrbdvqqczz'));

PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
SQL_ID 1wswrbdvqqczz
--------------------
CREATE TABLE TMP_STOCK AS SELECT A.ABID ABID , A.CLEADATE
STK_CLEADATE,          LN((A.PRICVALUE/ B.PRICVALUE)) RTRN, RANK()
OVER(PARTITION BY A.ABID ORDER BY  A.CLEADATE DESC) PERF_RNK    FROM
MONTHLY_DATAF A, MONTHLY_DATAF B, YEARLY_DATA C   WHERE A.ABID =
B.ABID     AND A.ABID = C.ABID     AND C.STATUS='AC'     AND B.CLEADATE
= ADD_MONTHS(A.CLEADATE, -1)     AND A.CLEADATE <= TO_DATE(20150630,
'YYYYMMDD')     AND A.CLEADATE > ADD_MONTHS (TO_DATE(20150630,
'YYYYMMDD'), -240)

Plan hash value: 1100610021

------------------------------------------------------------------------------------------------
| Id  | Operation              | Name          | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------
|   0 | CREATE TABLE STATEMENT |               |       |       |       |   147K(100)|          |
|   1 |  LOAD AS SELECT        |               |       |       |       |            |          |
|   2 |   WINDOW SORT          |               |  6333K|   477M|   537M|   135K  (1)| 00:27:02 |
|   3 |    HASH JOIN           |               |  6333K|   477M|       | 18286   (2)| 00:03:40 |
|   4 |     TABLE ACCESS FULL  | YEARLY_DATA   |  8608 | 77472 |       |    69   (2)| 00:00:01 |
|   5 |     HASH JOIN          |               |  6333K|   422M|    73M| 18147   (2)| 00:03:38 |
|   6 |      TABLE ACCESS FULL | MONTHLY_DATAF |  1642K|    54M|       |  5071   (2)| 00:01:01 |
|   7 |      TABLE ACCESS FULL | MONTHLY_DATAF |  1909K|    63M|       |  5061   (2)| 00:01:01 |
------------------------------------------------------------------------------------------------

Note
-----
   - dynamic sampling used for this statement

SQL_ID 1wswrbdvqqczz
--------------------
CREATE TABLE TMP_STOCK AS SELECT A.ABID ABID , A.CLEADATE
STK_CLEADATE,          LN((A.PRICVALUE/ B.PRICVALUE)) RTRN, RANK()
OVER(PARTITION BY A.ABID ORDER BY  A.CLEADATE DESC) PERF_RNK    FROM
MONTHLY_DATAF A, MONTHLY_DATAF B, YEARLY_DATA C   WHERE A.ABID =
B.ABID     AND A.ABID = C.ABID     AND C.STATUS='AC'     AND B.CLEADATE
= ADD_MONTHS(A.CLEADATE, -1)     AND A.CLEADATE <= TO_DATE(20150630,
'YYYYMMDD')     AND A.CLEADATE > ADD_MONTHS (TO_DATE(20150630,
'YYYYMMDD'), -240)

Plan hash value: 2482295346

---------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name                 | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------------
|   0 | CREATE TABLE STATEMENT           |                      |       |       |   477 (100)|          |
|   1 |  LOAD AS SELECT                  |                      |       |       |            |          |
|   2 |   WINDOW SORT                    |                      |   848 | 66992 |   475   (1)| 00:00:06 |
|   3 |    NESTED LOOPS                  |                      |       |       |            |          |
|   4 |     NESTED LOOPS                 |                      |   848 | 66992 |   474   (1)| 00:00:06 |
|   5 |      NESTED LOOPS                |                      |   186 |  8184 |   101   (1)| 00:00:02 |
|   6 |       TABLE ACCESS FULL          | YEARLY_DATA          |     1 |     9 |    69   (2)| 00:00:01 |
|   7 |       TABLE ACCESS BY INDEX ROWID| MONTHLY_DATAF        |   371 | 12985 |    32   (0)| 00:00:01 |
|   8 |        INDEX RANGE SCAN          | MONTHLY_DATAF_U_DATE |     2 |       |    30   (0)| 00:00:01 |
|   9 |      INDEX UNIQUE SCAN           | MONTHLY_DATAF_U_DATE |     1 |       |     1   (0)| 00:00:01 |
|  10 |     TABLE ACCESS BY INDEX ROWID  | MONTHLY_DATAF        |     5 |   175 |     2   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------------

Note
-----
   - dynamic sampling used for this statement
```

Some keys points while observing the execution plan difference between 1100610021 and 2482295346.

Both plans are using dynamic sampling, but we don't have any clue on which table dynamic sampling was done.
Good plan(1100610021) uses HASH JOIN but bad plan(2482295346) uses ```NESTED LOOP JOIN```.
In bad plan(2482295346) Optimizer has done worst estimation of the cardinality.
Cardinality estimation of table YEARLY_DATA is the worst part as per line ID 6 in bad plan(2482295346). This might be reason to go for ```NESTED LOOP JOIN``` instead of ```HASH JOIN```.

Statistics on table ```MONTHLY_DATAF``` were missing and on table ```YEARLY_DATA``` stats were gathered by Auto stats scheduler job. So dynamic sampling was done only on table ```MONTHLY_DATAF``` but not on table ```YEARLY_DATA```. I went back to Application to gather more information on these tables and I found that they drop and re-create the ```MONTHLY_DATAF``` table for every run of their batch job. But table ```YEARLY_DATA``` will not be dropped and re-create. This is why stats were missing on table ```MONTHLY_DATAF``` but gathered on YEARLY_DATA. 

Just to confirm my hypothesis that wrong estimation on table ```YEARLY_DATA``` (Only 1 row) has changed the join method from ```HASH``` to ```NESTED LOOP``` join, I tried using cardinality hint as shown below to reproduce the execution plan exactly similar to bad plan what we had on Aug 5th. Note that I have used only the select statement part of the create table statement as I am not suppose to modify or create any objects while investigation in this database.

```sql
SELECT /*+ cardinality (c 1) */ A.ABID ABID , A.CLEADATE
STK_CLEADATE,          LN((A.PRICVALUE/ B.PRICVALUE)) RTRN, RANK()
OVER(PARTITION BY A.ABID ORDER BY  A.CLEADATE DESC) PERF_RNK    
FROM MONTHLY_DATAF A, MONTHLY_DATAF B, YEARLY_DATA C   
WHERE A.ABID =B.ABID     
AND A.ABID = C.ABID     
AND C.STATUS='AC'     
AND B.CLEADATE= ADD_MONTHS(A.CLEADATE, -1)     
AND A.CLEADATE <= TO_DATE(20150630,'YYYYMMDD')     
AND A.CLEADATE > ADD_MONTHS (TO_DATE(20150630,'YYYYMMDD'), -240)		

---------------------------------------------------------------------------------------------
| Id  | Operation                       | Name                 | Rows  | Bytes | Cost (%CPU)|
---------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                      |  1300 |   100K|   765   (1)|
|   1 |  WINDOW SORT                    |                      |  1300 |   100K|   765   (1)|
|   2 |   NESTED LOOPS                  |                      |       |       |            |
|   3 |    NESTED LOOPS                 |                      |  1300 |   100K|   764   (1)|
|   4 |     NESTED LOOPS                |                      |   331 | 14564 |   101   (1)|
|*  5 |      TABLE ACCESS FULL          | YEARLY_DATA          |     1 |     9 |    69   (2)|
|   6 |      TABLE ACCESS BY INDEX ROWID| MONTHLY_DATAF        |   331 | 11585 |    32   (0)|
|*  7 |       INDEX RANGE SCAN          | MONTHLY_DATAF_U_DATE |     2 |       |    30   (0)|
|*  8 |     INDEX UNIQUE SCAN           | MONTHLY_DATAF_U_DATE |     1 |       |     1   (0)|
|   9 |    TABLE ACCESS BY INDEX ROWID  | MONTHLY_DATAF        |     4 |   140 |     2   (0)|
---------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - filter("C"."STATUS"='AC')
   7 - access("A"."CLEADATE">TO_DATE(' 1995-06-30 00:00:00', 'syyyy-mm-dd
              hh24:mi:ss') AND "A"."ABID"="C"."ABID" AND "A"."CLEADATE"<=TO_DATE(' 2015-06-30
              00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
       filter("A"."ABID"="C"."ABID")
   8 - access("B"."CLEADATE"=ADD_MONTHS(INTERNAL_FUNCTION("A"."CLEADATE"),(-1)) AND
              "A"."ABID"="B"."ABID")
```

Assumption is right on ballpark, not only the execution plan was reproduced exactly to bad plan on Aug 5th but also now I am able to get the 'Predicate Information' of the execution plan which says that ```FILTER``` opration is performed on the ```STATUS``` column while performing full table scan of ```YEARLY_DATA```. Of course this gives one more clue that stats on ```STATUS``` column might be the one to influence for ```NESTED LOOP``` join. 

Since stats influenced the Optimizer to go for plan change with ```NESTED LOOP JOIN```, its better to do comparison between stats gathered on Aug 5th with the stats present currently on the table ```YEARLY_DATA```. This can be achieved by using ```dbms_stats.diff_table_stats_in_history``` as shown below.

```sql
SQL> select * from table(dbms_stats.diff_table_stats_in_history(
                         ownname => 'APP_USER',
                         tabname => upper('&tabname'),
                         time1 => systimestamp,
                         time2 => to_timestamp('&time2','yyyy-mm-dd:hh24:mi:ss'),
                         pctthreshold => 0));   
Enter value for tabname: YEARLY_DATA
Enter value for time2: 2015-08-05:08:33:50

REPORT                                                                           MAXDIFFPCT
-------------------------------------------------------------------------------- ----------
###############################################################################          50

STATISTICS DIFFERENCE REPORT FOR:
.................................

TABLE         : YEARLY_DATA
OWNER         : APP_USER
SOURCE A      : Statistics as of 07-AUG-15 03.27.23.906055 AM -04:00
SOURCE B      : Statistics as of 05-AUG-15 08.33.50.000000 AM -04:00
PCTTHRESHOLD  : 0
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

TABLE / (SUB)PARTITION STATISTICS DIFFERENCE:
.............................................

OBJECTNAME                  TYP SRC ROWS       BLOCKS     ROWLEN     SAMPSIZE
...............................................................................

YEARLY_DATA                 T   A   17778      244        45         17778
                                B   17768      244        45         17768
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

COLUMN STATISTICS DIFFERENCE:
.............................

COLUMN_NAME     SRC NDV     DENSITY    HIST NULLS   LEN  MIN   MAX   SAMPSIZ
...............................................................................

LT              A   17760   .000056306 NO   18      10   30303 59393 17760
                B   17750   .000056338 NO   18      10   30303 59393 17750
ABID            A   17778   .000056249 NO   0       6    C20D0 C50A5 17778
                B   17768   .000056280 NO   0       6    C20D0 C50A5 17768
NAME            A   17778   .000056249 NO   0       23   312D3 78343 17778
                B   17768   .000056280 NO   0       23   312D3 78343 17768
STATUS          A   2       .000028124 YES  0       3    4143  494E  17778
                B   1       .000028140 YES  0       3    494E  494E  17768
VALUEI          A   16692   .000059908 YES  1086    5    20475 5A5A4 16692
                B   16683   .000059941 YES  1085    5    20475 5A5A4 16683
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

INDEX / (SUB)PARTITION STATISTICS DIFFERENCE:
.............................................

OBJECTNAME      TYP SRC ROWS    LEAFBLK DISTKEY LF/KY DB/KY CLF     LVL SAMPSIZ
...............................................................................


                          INDEX: YEARLY_DATA_PK_ABID
                          ............................

YEARLY_DATA_P   I   A   17778   51      17778   1     1     8234    1   17778
                    B   17768   51      17768   1     1     8233    1   17768

                          INDEX: YEARLY_DATA_U_CUSIP
                          ............................

YEARLY_DATA_U   I   A   17760   64      17760   1     1     17364   1   17760
                    B   17750   64      17750   1     1     17353   1   17750

                           INDEX: YEARLY_DATA_U_NAME
                           ...........................

YEARLY_DATA_U   I   A   17778   116     17778   1     1     17394   1   17778
                    B   17768   116     17768   1     1     17384   1   17768

                          INDEX: YEARLY_DATA_U_TICKER
                          .............................

YEARLY_DATA_U   I   A   16692   42      16692   1     1     16449   1   16692
                    B   16683   42      16683   1     1     16440   1   16683
###############################################################################			  
```

If you take a closer look at the above output you will find that there is no much difference between 5th and 7th August stats except stats realted on ```STATUS``` column. NDV (Number of DIstinct Values) on ```STATUS``` column was 1 on 5th Aug but 2 on 7th Aug. NDV plays very important role in cardinality estimation, in the same way cardinality estimation plays very important role in deciding the efficient join method. If you go back to the sql text of sql_id '1wswrbdvqqczz' you will find that it was passing literal value 'AC' for STATUS column, but according to stats present on 5th Aug table was containing only one value which was not 'AC' but some different value, this made Optimizer to guess only 1 row from the table ```YEARLY_DATA``` and to go for ```NESTED LOOP``` join. 

Now how can I find the value of only one distinct value in ```STATUS``` column which on 5th Aug stats were representing to prove my theory? 
Histogram to rescue !! Since histogram exist for ```STATUS``` column on 5th Aug(Though its interesting to see Histogram for only one distinct value), I can check its endpoint_actual_value in ```dba_tab_histogram```s view. But as these were history stats stored in few internal tables I had to check with ```WRI$_OPTSTAT_HISTHEAD_HISTORY``` for ```MI``` and ```MAX``` value as ```endpoint_actual_value``` is not available in any history tables. This means I had to convert ```MIN``` and ```MAX``` value from ```WRI$_OPTSTAT_HISTHEAD_HISTORY``` into a hex string, extracting the first six pairs of digits, converting to numeric and applying the chr() function to get a character value as shown below.

```sql
select OBJ#,INTCOL#,SAVTIME,NULL_CNT,MINIMUM,MAXIMUM,DISTCNT,DENSITY,LOWVAL,HIVAL,SAMPLE_DISTCNT,SAMPLE_SIZE
from WRI$_OPTSTAT_HISTHEAD_HISTORY
where OBJ#=925718 and INTCOL#=5
order by SAVTIME;

      OBJ#    INTCOL# SAVTIME                                    NULL_CNT    MINIMUM    MAXIMUM    DISTCNT    DENSITY LOWVAL       HIVAL        SAMPLE_DISTCNT SAMPLE_SIZE 
---------- ---------- ---------------------------------------- ---------- ---------- ---------- ---------- ---------- ------------ ------------ -------------- ----------- 
    925718          5 05-AUG-15 08.33.50.666685 AM -04:00               0 3.8062E+35 3.8062E+35          1  .00002814 494E         494E                      1       17768 
    925718          5 06-AUG-15 01.01.11.784408 AM -04:00               0 3.3886E+35 3.8062E+35          2  .00002812 4143         494E                      2       17778   
    

select
	chr(to_number(substr(hex_val, 2,2),'XX')) ||
    chr(to_number(substr(hex_val, 4,2),'XX'))
from (
select
to_char(3.3886E+35,'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX')hex_val
from dual);

CH
--
AC
	
select
	chr(to_number(substr(hex_val, 2,2),'XX')) ||
    chr(to_number(substr(hex_val, 4,2),'XX'))
from (
select
to_char(3.8062E+35,'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX')hex_val
from dual);

CH
--
IN
```

Its clear that on 5th Aug table was having only 'IN' values in ```STATUS``` column and that why stats were having NDV as only 1. Again I went back to Application team to get more details about ```STATUS``` column, and found that AC stand for ACTIVE and IN stands for INACTIVE. Finally after providing all these details to Application team we came to an conclusion. 

Conclusion
----------

* Table ```YEARLY_DATA``` was having only INACTIVE records(STATUS='IN')
* Auto stats job gathered the stats for this table.
* Application team loaded all the ACTIVE records(STATUS='AC')
* Application team started their Batch job, which inturn executed sql_id '1wswrbdvqqczz' by passing STATUS='AC', but stats were representing only 'IN'.
* Batch job was hung due to bad plan(2482295346).
	
Its always important to decide at what time/circumstance statistics has to be gathered, else your statistics may depict data which may not narrate to actual data in the tables.

