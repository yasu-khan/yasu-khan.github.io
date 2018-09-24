---
layout: post
title: Oracle SQL Plan Directive - (Part 1)
description: Oracle 12c SQL Plan Directive insights - Part 1
comments: true
keywords: SPD, Oracle, SQL Plan Directive
---

Overview
--------

Sql Plan Directives(SPD) got introduced in 12.1.0.1 and its main objective is to take corrective action or help the Optimizer whenever there is misestimation of cardinality, this help Optimizer to generate more optimal plan. SPD are created at table/column/predicate level so that it can be used by any sql matching the table/column/predicate section. SPD are stored persistently into SYSAUX tablespaces so that any cursor agening out of shared pool can use it again without going through all the learning curve once again. SPD provides temporary fix through Dynamic Sampling and also strives to provide permanent fix by instructing ```dbms_stats``` to create Extended Statistics.

Even in 11g release we have feature called Cardinality Feedback to take corrective measures for helping Optimizer to avoid misestimation of cardinality, but whatever measures Cardinality Feedback uses to take were not persistant and one more important fact is that corrective actions are taken at each cursor level which means one cursor learnings can't be leveraged to any other cursors. In 11g creation of Extended Statistics is manual process by using ```dbms_stats.seed_col_usage``` procedure to determine the appropriate the column groups.

SPD has different states which are very imporatnt to observe its behaviour and when compared to 12.1.0.1 to 12.1.0.2 SPD states have been changed as shown in below table.

{% highlight sql %}

|    12.1.0.1    |   12.1.0.2   |
|--------------------------------
|NEW             |    USABLE    |
|MISSING_STATS   |    USABLE    |
|PERMANENT       |    USABLE    |
|HAS_STATS       |  SUPERSEDED  |
{% endhighlight %}

Good thing is that if we check the base view definition of ```DBA_SQL_PLAN_DIRECTIVES``` in 12.1.0.2 both types of states are stored in ```STATE``` and ```NOTES(XML type)``` column which gives lot of details like whether SPD is redundant or not and the actual SPD text to define the purpose of SPD creation. Its always better to look at ```INTERNAL_STATE``` column instead of ```STATE``` column for an SPD as it gives more detail of each SPD state transformation, hence we can extract it by using XML functions from NOTES column of ```DBA_SQL_PLAN_DIRECTIVES```. Also if you check ```LAST_USED``` column from base view ```_BASE_OPT_DIRECTIVE``` it is updated for every 6.5 days ```(cast(d.last\_used as timestamp) - NUMTODSINTERVAL(6.5, 'day'))```. One more important fact is that as of now Dynamic Sampling is the only supported type for SPD as we see ```'decode(type, 1, 'DYNAMIC_SAMPLING', 'UNKNOWN')'``` for base view definition of ```_BASE_OPT_DIRECTIVE```.


{% highlight sql %}
SQL> select view_name, text from dba_views where view_name ='DBA_SQL_PLAN_DIRECTIVES';
VIEW_NAME                 TEXT
------------------------- --------------------------------------------------------------------------------
DBA_SQL_PLAN_DIRECTIVES   SELECT
                              d.dir_id,
                              d.type,
                              d.enabled,
                              case when d.internal_state = 'HAS_STATS' or d.redundant = 'YES'
                                     then 'SUPERSEDED'
                                   when d.internal_state in ('NEW', 'MISSING_STATS', 'PERMANENT')
                                     then 'USABLE'
                                   else 'UNKNOWN' end case,
                              d.auto_drop,
                              f.reason,
                              d.created,
                              d.last_modified,
                              d.last_used,
                              xmltype(
                                '<spd_note>' ||
                                   '<internal_state>' || d.internal_state || '</internal_state>' ||
                                   '<redundant>' || d.redundant || '</redundant>' ||
                                   '<spd_text>' || sys.dbms_spd_internal.get_spd_text(d.dir_id) ||
                                   '</spd_text>' ||
                                 '</spd_note>')  notes
                          FROM
                              sys."_BASE_OPT_DIRECTIVE" d,
                              sys."_BASE_OPT_FINDING" f
                          WHERE d.f_id = f.f_id

SQL> select view_name, text from dba_views where view_name ='_BASE_OPT_DIRECTIVE';
VIEW_NAME            TEXT
-------------------- -------------------------------------------------------------------------------------
_BASE_OPT_DIRECTIVE  SELECT
                         d.dir_own#,
                         d.dir_id,
                         d.f_id,
                         decode(type, 1, 'DYNAMIC_SAMPLING', 'UNKNOWN'),
                         decode(state, 1, 'NEW',
                                      2, 'MISSING_STATS',
                                      3, 'HAS_STATS',
                                      5, 'PERMANENT',
                                      'UNKNOWN'),
                         decode(bitand(flags, 1), 1, 'YES', 'NO'),
                         decode(bitand(flags, 2), 2, 'YES', 'NO'),
                         decode(bitand(flags, 4), 4, 'NO', 'YES'),
                         cast(d.created as timestamp),
                         cast(d.last_modified as timestamp),
                         -- Please see QOSD_DAYS_TO_UPDATE and QOSD_PLUS_SECONDS for more details
                         -- about 6.5
                         cast(d.last_used as timestamp) - NUMTODSINTERVAL(6.5, 'day')
                     FROM
                         sys.opt_directive$ d
			 
{% endhighlight %}

Demo
----

To understand the SPD in detail and purpose of each ```INTERNAL_STATE``` let me go through a simple demo by using the popular ```CUSTOMERS``` table in ```SH``` schema.

By executing below sql query with ```gather_plan_statistics``` hint we can check the difference between Estimated and Actual cardinality.

{% highlight sql %}
SQL> SELECT /*+ gather_plan_statistics */ count(*)
     FROM   sh.customers
     WHERE  cust_city='Los Angeles'
     AND    cust_state_province='CA';

  COUNT(*)
----------
       932
    
SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------------
SQL_ID  0ch70x7cfqfvc, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   sh.customers WHERE
 cust_city='Los Angeles' AND    cust_state_province='CA'

Plan hash value: 296924608

---------------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |00:00:00.02 |    1522 |      1 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |00:00:00.02 |    1522 |      1 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |      1 |     51 |    932 |00:00:01.75 |    1522 |      1 |
---------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

{% endhighlight %}

There is huge difference in Estimated(51) and Actual cardinality(932), its due to correlation between two columns ```cust_city``` and ```cust_state_province```. This misestimation of cardinality will be the cause for bad access path selection by Optimizer. By default Optimizer is not aware of this relationship between two columns as default statistics are gathered on each individual column level but not in combination of columns.

Let's check whether this sql is candidate for re-optimization or not and the reason for it.

{% highlight sql %}
SQL> select sql_id,child_number,is_reoptimizable from v$sql where sql_id='0ch70x7cfqfvc';

SQL_ID        CHILD_NUMBER I
------------- ------------ -
0ch70x7cfqfvc            0 Y

{% endhighlight %}
So this sql is the candidate for re-optimization, now lets try to find out how this reoptimization is going to help this sql.

{% highlight sql %}
SQL> select hash_value, sql_id, child_number, hint_text from V$sql_reoptimization_hints where sql_id='0ch70x7cfqfvc';

HASH_VALUE SQL_ID        CHILD_NUMBER HINT_TEXT
---------- ------------- ------------ --------------------------------------------------------------------------------
3639294828 0ch70x7cfqfvc            0 OPT_ESTIMATE (@"SEL$1" TABLE "CUSTOMERS"@"SEL$1" ROWS=932.000000 )
{% endhighlight %}

Seems like re-optimization is for adjusting the cardinality and this confirms it will go for Statistics Feedback on next parse of this sql. But since this sql is the candidate for re-optimization SPD will kick in and creates the directive aswell. By using ```dbms_xplan``` along with +report option we can see what intention Optimizer is having to generate the plan after re-optimization.

{% highlight sql %}
SQL> select * from table(dbms_xplan.display_cursor('0ch70x7cfqfvc',null,'allstats +report'));

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------------
SQL_ID  0ch70x7cfqfvc, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   sh.customers WHERE
 cust_city='Los Angeles' AND    cust_state_province='CA'

Plan hash value: 296924608

---------------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |00:00:00.02 |    1522 |      1 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |00:00:00.02 |    1522 |      1 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |      1 |     51 |    932 |00:00:01.75 |    1522 |      1 |
---------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))


Reoptimized plan:
-----------------
This cursor is marked for automatic reoptimization.  The plan that is
expected to be chosen on the next execution is displayed below.

Plan hash value: 296924608

------------------------------------------------
| Id  | Operation          | Name      | Rows  |
------------------------------------------------
|   0 | SELECT STATEMENT   |           |     1 |
|   1 |  SORT AGGREGATE    |           |     1 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |   932 |
------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - filter("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA')

{% endhighlight %}

By using ```dbms_xplan``` with ```+report``` option it provides us the Reoptimized plan with Estimated rows matching the Actual rows. Let's check what SPD has done behind the scene by querying the view ```DBA_SQL_PLAN_DIRECTIVES```. But before querying this view we need to flush the SPD information from memory to disk(```DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE```) as the default frequency for flushing this information is 15 minutes.

{% highlight sql %}
SQL> EXEC DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE;
PL/SQL procedure successfully completed.

SQL> SELECT directive_id,
            state,
            last_used,
            auto_drop,
            enabled,
            Extract( notes, '/spd_note/spd_text/text()' )       spd_text,
            Extract( notes, '/spd_note/internal_state/text()' ) internal_state
     FROM   DBA_SQL_PLAN_DIRECTIVES
     WHERE  directive_id IN
            ( SELECT directive_id
              FROM   DBA_SQL_PLAN_DIR_OBJECTS
              WHERE  owner = 'SH' )
/
         DIRECTIVE_ID STATE      LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE
--------------------- ---------- ------------------------------- --- --- -------------------------------------------------- ---------------
 12105355473441073000 USABLE                                     YES YES {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} NEW
{% endhighlight %}

So SPD has been created and is in NEW ```internal_state``` which means it's waiting for more information to be gathered and hence not in usable state. 

Re-execute the same sql and check the corrective measures taken by the Optimizer by creating new child cursor.

{% highlight sql %}
SQL> SELECT /*+ gather_plan_statistics */ count(*)
     FROM   sh.customers
     WHERE  cust_city='Los Angeles'
     AND    cust_state_province='CA';
  COUNT(*)
----------
       932

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------------
SQL_ID  0ch70x7cfqfvc, child number 1
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM   sh.customers WHERE
 cust_city='Los Angeles' AND    cust_state_province='CA'

Plan hash value: 296924608

------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |00:00:00.01 |    1522 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |00:00:00.01 |    1522 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |      1 |    932 |    932 |00:00:00.01 |    1522 |
------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

Note
-----
   - statistics feedback used for this statement

SQL> select sql_id,child_number,is_reoptimizable from v$sql where sql_id='0ch70x7cfqfvc';

SQL_ID        CHILD_NUMBER I
------------- ------------ -
0ch70x7cfqfvc            0 Y
0ch70x7cfqfvc            1 N
{% endhighlight %}


New child cursor got created along with the corrective measures taken by Statistics Feedback(Renamed from Cardinality Feedback in 11g)

Again check the ```Internal_state``` of SPD.

{% highlight sql %}
SQL> EXEC DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE;
PL/SQL procedure successfully completed.

SQL> SELECT directive_id,
            state,
            last_used,
            auto_drop,
            enabled,
            Extract( notes, '/spd_note/spd_text/text()' )       spd_text,
            Extract( notes, '/spd_note/internal_state/text()' ) internal_state
     FROM   DBA_SQL_PLAN_DIRECTIVES
     WHERE  directive_id IN
            ( SELECT directive_id
              FROM   DBA_SQL_PLAN_DIR_OBJECTS
              WHERE  owner = 'SH' )
/
         DIRECTIVE_ID STATE      LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE
--------------------- ---------- ------------------------------- --- --- -------------------------------------------------- ---------------
 12105355473441073000 USABLE     05-DEC-15 12.30.57.000000000 PM YES YES {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} MISSING_STATS
{% endhighlight %}

Now SPD has updated ```internal_state``` to ```MISSING_STATS``` and ready for use by any sql using same table/columns/predicate. If you check the column ```SPD_TEXT``` it has been prefix with EC and to more details on this we can look into definition of base view ```_BASE_OPT_FINDING_OBJ``` which is one of the base view for ```DBA_SQL_PLAN_DIR_OBJECTS``` as shown below.

{% highlight sql %}
SQL> select view_name, text from dba_views where view_name ='_BASE_OPT_FINDING_OBJ';

VIEW_NAME             TEXT
--------------------- --------------------------------------------------------------------------------
_BASE_OPT_FINDING_OBJ SELECT
                          f.f_id,
                          f.f_obj#,
                          f.obj_type,
                          f.col_list,
                          f.cvec_size,
                          xmltype(
                            '<obj_note>' ||
                               '<equality_predicates_only>' ||
                                 decode(bitand(f.flags, 1), 0, 'NO', 'YES') ||
                               '</equality_predicates_only>' ||
                               '<simple_column_predicates_only>' ||
                                 decode(bitand(f.flags, 2), 0, 'NO', 'YES') ||
                               '</simple_column_predicates_only>' ||
                               '<index_access_by_join_predicates>' ||
                                 decode(bitand(f.flags, 4), 0, 'NO', 'YES') ||
                               '</index_access_by_join_predicates>' ||
                               '<filter_on_joining_object>' ||
                                 decode(bitand(f.flags, 8), 0, 'NO', 'YES') ||
                               '</filter_on_joining_object>' ||
                             '</obj_note>')  notes
                      FROM
                          sys.opt_finding_obj$ f

{% endhighlight %}

The XMLTYPE column defines each notation meaning as follows
{% highlight sql %}
equality_predicates_only         E
simple_column_predicates_only    C
index_access_by_join_predicates  J
filter_on_joining_object         F
{% endhighlight %}

In our case ```{EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]}``` illustrate ```equality_predicates_only``` and ```simple_column_predicates_only``` on columns ```CUST_CITY``` and ```CUST_STATE_PROVINCE``` of table ```SH.CUSTOMERS```. As you see there is no information of sql's using it, this is why SPD are global in terms of usage and can be used by any sql having predicates using these columns in combination.

Let's try to run different sql having same predicates to check if SPD created previously will be used or not.

{% highlight sql %}
SELECT /*+ gather_plan_statistics */ count(cust_city)
  FROM   sh.customers
  WHERE  cust_city='Los Angeles'
  AND    cust_state_province='CA';
COUNT(CUST_CITY)
----------------
             932

SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------------
SQL_ID  gqdy2t3g2xxw8, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(cust_city)   FROM
sh.customers   WHERE  cust_city='Los Angeles'   AND
cust_state_province='CA'

Plan hash value: 296924608

------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |00:00:00.01 |    1522 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |00:00:00.01 |    1522 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |      1 |    952 |    932 |00:00:00.01 |    1522 |
------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - 1 Sql Plan Directive used for this statement

{% endhighlight %}

As per the NOTE section previously created SPD is used by this sql to perform synamic sampling due to which Actual and Estimated cardinality has been improved.

Now let's gather statistics on table ```SH.CUSTOMERS``` as SPD internal_state is informing as ```MISSING_STATS```.

{% highlight sql %}
SQL> exec dbms_stats.gather_table_stats('SH','CUSTOMERS')

PL/SQL procedure successfully completed.
{% endhighlight %}


If we check the statistics for this table we see that Extended Statistics has been created, this is due to ```dbms_stats``` deriving information of required/missing statistics from SPD.

{% highlight sql %}
SQL> select TABLE_NAME,EXTENSION_NAME,EXTENSION,CREATOR from dba_stat_extensions where TABLE_NAME='CUSTOMERS' and owner='SH';

TABLE_NAME     EXTENSION_NAME                 EXTENSION                           CREATO
-------------- ------------------------------ ----------------------------------- ------
CUSTOMERS      SYS_STSWMBUN3F$#398R7BS0YVS86R ("CUST_CITY","CUST_STATE_PROVINCE") SYSTEM
{% endhighlight %}

Let's again execute different sql having same predicates on this table and see whether SPD or Extended Statistics will be used.

{% highlight sql %}
SELECT /*+ gather_plan_statistics */ count(cust_state_province)
  FROM   sh.customers
  WHERE  cust_city='Los Angeles'
  AND    cust_state_province='CA';
COUNT(CUST_STATE_PROVINCE)
--------------------------
                       932
  
SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------
SQL_ID  7tava2xapwvm4, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(cust_state_province)   FROM
sh.customers2   WHERE  cust_city='Los Angeles'   AND
cust_state_province='CA'

Plan hash value: 2704912892

-------------------------------------------------------------------------------------------
| Id  | Operation          | Name       | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |            |      1 |        |      1 |00:00:00.01 |    1521 |
|   1 |  SORT AGGREGATE    |            |      1 |      1 |      1 |00:00:00.01 |    1521 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS2 |      1 |    958 |    932 |00:00:00.01 |    1521 |
-------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

{% endhighlight %}


So SPD has not been used instead Extended Statistics has been used as there is no NOTE section saying any usage of SPD. Now check the SPD state as we have not used it in previous run.

{% highlight sql %}
SQL> EXEC DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE;
PL/SQL procedure successfully completed.

SQL> SELECT directive_id,
            state,
            last_used,
            auto_drop,
            enabled,
            extract(notes, '/spd_note/spd_text/text()' )       spd_text,
            extract(notes, '/spd_note/internal_state/text()' ) internal_state,
			extract(notes, '/spd_note/redundant/text()')       redundant 
     FROM   DBA_SQL_PLAN_DIRECTIVES
     WHERE  directive_id IN
            ( SELECT directive_id
              FROM   DBA_SQL_PLAN_DIR_OBJECTS
              WHERE  owner = 'SH' )
/
         DIRECTIVE_ID STATE      LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE  REDUNDANT
--------------------- ---------- ------------------------------- --- --- -------------------------------------------------- --------------- ---------
 12105355473441073000 SUPERSEDED 05-DEC-15 12.30.57.000000000 PM YES YES {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} HAS_STATS       NO  
{% endhighlight %}

SPD internal_state has been changed from ```MISSING_STATS``` to ```HAS_STATS``` internal_state as we have Extended Statistics now.

As you observe SPD internal\_state has been changed from ```NEW``` to ```MISSING_STATS``` to ```HAS_STATS```, but what is ```PERMANENT``` internal\_state then? Well when SPD finds that even after creating Extended Statistics the cardinality misestimation has not been resolved then it will revert back to Dynamic Sampling method permenantly and thus SPD internal\_state will reflect as ```PERMANENT```. To simulate it I will drop previously created Extended Statistics on this table and again run the sql on this table having same predicates to check what SPD does in such case.

{% highlight sql %}
SQL> exec dbms_stats.drop_extended_stats('SH','CUSTOMERS','("CUST_CITY","CUST_STATE_PROVINCE")')
SQL> alter system flush shared_pool;

SQL> SELECT /*+ gather_plan_statistics */ count(*)
     FROM   sh.customers
     WHERE  cust_city='Los Angeles'
     AND    cust_state_province='CA';

  COUNT(
  *)
----------
       932

SQL> select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));	   
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------------
SQL_ID  1tzub0d9tw7jq, child number 0
-------------------------------------
SELECT /*+ gather_plan_statistics */ count(*)   FROM   sh.customers
WHERE  cust_city='Los Angeles'   AND    cust_state_province='CA'

Plan hash value: 296924608

------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |00:00:00.01 |    1522 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |00:00:00.01 |    1522 |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |      1 |    952 |    932 |00:00:00.01 |    1522 |
------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - filter(("CUST_CITY"='Los Angeles' AND "CUST_STATE_PROVINCE"='CA'))

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - 1 Sql Plan Directive used for this statement

{% endhighlight %}

As expected SPD has directed to use dynamic sampling, now lets check what SPD internal\_state will be for this directive.

{% highlight sql %}
SQL> EXEC DBMS_SPD.FLUSH_SQL_PLAN_DIRECTIVE;
PL/SQL procedure successfully completed.

SQL> SELECT directive_id,
            state,
            last_used,
            auto_drop,
            enabled,
            extract(notes, '/spd_note/spd_text/text()' )       spd_text,
            extract(notes, '/spd_note/internal_state/text()' ) internal_state
      extract(notes, '/spd_note/redundant/text()')       redundant 
     FROM   DBA_SQL_PLAN_DIRECTIVES
     WHERE  directive_id IN
            ( SELECT directive_id
              FROM   DBA_SQL_PLAN_DIR_OBJECTS
              WHERE  owner = 'SH' )
/
         DIRECTIVE_ID STATE      LAST_USED                       AUT ENA SPD_TEXT                                           INTERNAL_STATE  REDUNDANT
--------------------- ---------- ------------------------------- --- --- -------------------------------------------------- --------------- ---------
 12105355473441073000 USABLE     05-DEC-15 12.30.57.000000000 PM YES YES {EC(SH.CUSTOMERS)[CUST_CITY, CUST_STATE_PROVINCE]} PERMANENT       NO 
{% endhighlight %}

Its internal\_state has been changed to PERMANENT as SPD thinks Extended Statistics is of no help here, but infact we dropped the Extended Statistics on this table and SPD is not aware of it. SPD just thinks Extended Statistics did not help to resolve the misestimation.

If you observe ```REDUNDANT``` column has the value ```NO``` always for the directive as it state that there is no other directive which has already resolved the cardinality misestimation. In other way a table/column can have multiple directives on them and if any one them resolves the cardinality misestimation then all other similar directives will be redundant.

Conclusion
----------

In this article we saw how SPD gets created and transfrom from one internal\_state to another. SPD helps Optimizer by taking corrective actions for resolving misestimation of cardinality and persist the information permanentaly so that any other sql's can take advantage of it. It also helps ```DBMS_STATS``` for creating required Extended Statistics with any manual intervention from DBA. 
From maintenance perspective and controlling behaviour of SPD in real time prodcution databases would be complex, there are many things which we need to be aware of when we see full fledge use of SPD in production. Stay tuned for next part of this article describing all the important mechanism of SPD.
