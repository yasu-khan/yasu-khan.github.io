---
layout: post
title: How 'Consistent Read' processing is dealt by Oracle Exadata cell server ?
description: How 'Consistent Read' processing is dealt by Oracle Exadata cell server
comments: true
keywords: Oracle, Exadata, consistent read, cell blocks helped by commit cache, Commit Cache, MinSCN
---

Concept
-------
Consistent read is one of the main concept among the ACID properties provided by Oracle. The one which supports this Consistency is UNDO in Oracle. But according to concepts of Exadata, cell/storage servers doesn't communicate each other and thus constructing the consistent image of the database blocks by using UNDO data is not possible from storage layer as UNDO data itself is striped accross multiple storage servers which can't be shared/pass among the cell servers to construct the consistent data. This has direct impact on Smart Scan efficiency as those entire blocks has to be shipped back to database servers for processing of consistent read with the help UNDO data. Oracle has implemented new optimizations in Exadata to minimize the impact on Smart Scan by reducing the back and forth communication between cell server and database server whenever Smart Scan finds the blocks having active lock byte set(ITL).

When Smart Scan reaches a row which has lock byte set it has to ship entire block back to database layer for constructing consistent blocks by taking advantage of UNDO data. But when lock byte is set and not cleared for the rows by the transaction which has already been committed then Exadata will not ship those blocks to database layer everytime due to implementation of new enhancements 'commit cache' and 'minscn' optimization. So Oracle has taken few steps to avoid slow down of Smart Scan when lock byte is set for the rows by transaction which is committed already, but when Smart Scan encounters the blocks where lock byte is set and transaction is not yet committed then it has no way of dealing it in cell server level instead it has to transfer all such blocks back to the database layer for consistent read processing.

Whenever query starts it usually compares it's query start SCN with the cleanout scn of the blocks where the lock byte has been cleared, if query scn is greater than the cleanout scn found in the block then cell server will come to know that there is no need of rolling back that block for consistent read purpose. But if query scn is less than the cleanout scn found in the block, it has to be shipped back to database layer for consistent read processing which will have huge impact on Smart Scan as it have to ship back many such blocks to database layer where lock byte is set and at places where lock byte is set and has not been cleaned out. This ensures that whenever query starts using Smart Scan it sends its query scn to cell servers through iDB protocol. Even in these types of scenarios Exadata optimization like ```'commit cache'``` and ```'minscn'``` at cell server level will help to reduce the amount of blocks needs to be shipped back to database layer for consistent read processing.

Investigating it with session level statistics
----------------------------------------------
Let's create a table and set the storage property to not cache blocks of this table into flash cache for having stable performance metrics. 

```sql
CREATE TABLE DEMO
  (
     id   NUMBER,
     pad  VARCHAR2(500)
  ) TABLESPACE users STORAGE (flash_cache NONE)
/  
```

Load the table such that we have about 10,000 blocks for easier conputation of various metrics. 
```sql
INSERT INTO DEMO
SELECT rownum,
       rpad( '*', 500, '*' )
FROM   DUAL
CONNECT BY LEVEL <= 1.4e5
/

COMMIT
/

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'DEMO')
/
```

Before moving ahead with the setup let's check how many blocks have been allocated for this table and how many of them are in use and free, as this information will be helpful while interpreting Smart Scan related statistics. The best way to get such details is by using ```dbms_space``` package as shown below.

```sql
declare
  UNFOR_BLK number;
  UNFOR_BYT number;
  FS1_BLK   number;
  FS1_BYT   number;
  FS2_BLK   number;
  FS2_BYT   number;
  FS3_BLK   number;
  FS3_BYT   number;
  FS4_BLK   number;
  FS4_BYT   number;
  FULL_BLK  number;
  FULL_BYT  number;
begin
  dbms_space.space_usage('TEST','DEMO','TABLE',UNFOR_BLK,UNFOR_BYT,FS1_BLK,FS1_BYT,FS2_BLK,FS2_BYT,FS3_BLK,FS3_BYT,FS4_BLK,FS4_BYT,FULL_BLK,FULL_BYT);
  dbms_output.put_line('Unformatted Blocks                  ==> '||UNFOR_BLK);
  dbms_output.put_line('Blocks having 0  to 25%  free space ==> '||FS1_BLK  );
  dbms_output.put_line('Blocks having 25 to 50%  free space ==> '||FS2_BLK  );
  dbms_output.put_line('Blocks having 50 to 75%  free space ==> '||FS3_BLK  );
  dbms_output.put_line('Blocks having 75 to 100% free space ==> '||FS4_BLK  );
  dbms_output.put_line('Blocks which are full               ==> '||FULL_BLK );
end;
/

Unformatted Blocks                  ==> 0
Blocks having 0  to 25%  free space ==> 1
Blocks having 25 to 50%  free space ==> 0
Blocks having 50 to 75%  free space ==> 0
Blocks having 75 to 100% free space ==> 97
Blocks which are full               ==> 9999

PL/SQL procedure successfully completed.
```

With above output we could say that this table has 9999+1=10000 blocks which are fully or atleast 75% used and has 97 free blocks which doesn't contain any data.

Now setup a base table to capture all the session level staistics by using ```V$SESSTAT``` for a target session simulating different cases so that we can perform our analysis on these metrics later. This base table will have unique captured identifier to represent each snapshot of ```V$SESSTAT```.

```sql
CREATE TABLE CAP_SES_STAT AS 
SELECT 0 AS ID, name, value
FROM v$statname NATURAL JOIN v$sesstat
WHERE 1=2
/
```

From session 1
* Execute the UPDATE statement - [ ```update demo set id=id*50``` ]
* Flush buffer cache - [ ```alter system flush buffer_cache``` ]
* Commit the transaction - [ ```commit``` ]
* Perform conventional read of the table to clear the transaction lock byte set in the blocks. - [ ```alter session set "_serial_direct_read"=FALSE; select count(*) from demo ```]
	
From session 2 set the ```"_serial_direct_read"``` to ```ALWAYS``` to force direct path of the table regardless of table size and other factors, this ensures Smart Scan would happen on this table. After enabling this parameter this session has to perform ```"SELECT count(*) FROM DEMO"``` after each activity of session 1. 

From session 3 capture the session level statistics of session 2 for each select statement execution with unique ID as shown below.
```sql
INSERT INTO CAP_SES_STAT
SELECT 1, name, value
FROM v$statname NATURAL JOIN v$sesstat
WHERE SID=2
/
```

When all the three sessions are done with their tasks we will have snapshots of session statistics for each different simulated case stored in table ```CAP_SES_STAT```. With this information we can derive the result for each different case and do our analysis of Smart Scan dealing with Consistent Read of the blocks. Below sql is selecting only few statistics which are deemed to be worthful and is reporting the result of each case in columnar format using PIVOT so that results of each case can be compared easily.

```sql
SELECT   *
FROM (
      SELECT   id,
               name,
               value - Lag ( value ) over ( ORDER BY name, id ) AS value
      FROM     cap_ses_stat
      WHERE    name IN ('CPU used by this session',
                        'active txn count during cleanout',
                        'cell blocks helped by minscn optimization',
                        'cell blocks processed by cache layer',
                        'cell blocks processed by data layer',
                        'cell blocks processed by txn layer',
                        'cell commit cache queries',
                        'cell physical IO bytes eligible for predicate offload',
                        'cell physical IO interconnect bytes',
                        'cell physical IO interconnect bytes returned by smart scan',
                        'cell scans',
                        'cell transactions found in commit cache',
                        'cleanouts and rollbacks - consistent read gets',
                        'consistent gets',
                        'data blocks consistent reads - undo records applied',
                        'physical read total IO requests',
                        'physical read total multi block requests',
                        'cleanouts only - consistent read gets',
                        'session logical reads',
                        'physical reads',
                        'physical reads direct',
                        'cell blocks helped by commit cache',
                        'table scan blocks gotten',
                        'table scans (direct read)',
                        'table scan rows gotten' )
     ) 
	 pivot ( SUM ( value ) FOR id IN ( 1 AS before_commit,
                                       2 AS after_flush1,
                                       3 AS after_flush2,
                                       4 AS after_commit,
                                       5 AS after_scan ) )
/

NAME                                                       BEFORE_COMMIT AFTER_FLUSH1 AFTER_FLUSH2 AFTER_COMMIT AFTER_SCAN
---------------------------------------------------------- ------------- ------------ ------------ ------------ ----------
CPU used by this session                                              48           89           55            4          4
active txn count during cleanout                                   10000        10000        10000            0          0
cell scans                                                             1            1            1            1          1
cell physical IO bytes eligible for predicate offload           82714624     82714624     82714624     82714624   82714624
cell physical IO interconnect bytes                             81937784    105792280     81937632      2581416    2573224
cell physical IO interconnect bytes returned by smart scan      81937784     81937176     81937632      2573224    2573224
cell blocks processed by cache layer                               10127        10123        10126        10097      10097
cell blocks processed by data layer                                   97           97           97        10097      10097
cell blocks processed by txn layer                                    97           97           97        10097      10097
cell commit cache queries                                          10030        10026        10029        10000          0
cell transactions found in commit cache                                0            0            0        10000          0
cell blocks helped by commit cache                                     0            0            0        10000          0
cell blocks helped by minscn optimization                             97           97           97           97         97
cleanouts and rollbacks - consistent read gets                     10000        10000        10000            0          0
cleanouts only - consistent read gets                                  0            0            0            0          0
consistent gets                                                   300003       300003       300003        10100      10100
data blocks consistent reads - undo records applied               279903       279903       279903            0          0
physical read total IO requests                                      117         3025          116           88         87
physical read total multi block requests                              88           84           85           79         79
physical reads                                                     10097        13009        10097        10098      10097
physical reads direct                                              10097        10097        10097        10097      10097
session logical reads                                             300003       300003       300003        10100      10100
table scan blocks gotten                                           10000        10000        10000        10000      10000
table scan rows gotten                                            140000       140000       140000       140000     140000
table scans (direct read)                                              1            1            1            1          1
```

With above output it will be easy to conclude the concepts of Smart Scan dealing with consistent reads of the blocks. Before we proceed it's important to understand the concepts of ```'commit cache'``` and ```'minscn'``` optimization of cell server for consistent read.

Commit Cache
------------
It is an memory area in cell servers which keeps track of recently committed transaction. When smart scan hits the blocks having lock byte set then it will check commit cache area to find information of the transaction found in ITL section of the block, this check/query of commit cache area will increase the statistic ```'cell commit cache queries'``` and if it finds the information of this transaction then it will increase the statistic ```'cell blocks helped by commit cache'``` or ```'cell transactions found in commit cache'```. Without this optimzation it would overhead for cell server to ship all the blocks back to database server for consistent read processing which infact is very slow due to single block I/O.

Minscn
------
This is one more kind of optimization used by cell servers which keeps track of oldest open transaction SCN. This information will be helpful when smart scan hits the blocks having lock byte set, it can get the SCN from ITL entry and compare it to the oldest open transaction SCN and conclude whether this transaction has been committed or not. If smart scan is optimized with this optimization then statistic ```'cell blocks helped by minscn optimization'``` will be increased. Before smart scan operation database server will send this minscn information to the cell servers to take advantage of it and avoid interaction with database layer for finding whether particular transaction has been committed or not. This minscn is maintained by MMON and is global in RAC, it can be queried from ```x$ktumascn``` table.

Let's walk through each case and see how consistent read requests were handled intelligently. 

#### Before Commit

In this case it did smart scan of the table ```DEMO``` after updating all the rows without committing the transaction. By looking at statistics ```'cell scans'``` and ```'table scans (direct read)'``` we can say for sure that smart scan has occurred but there was no reduction in bytes returned by the smart scan as statistics ```'cell physical IO bytes eligible for predicate offload'``` , ```'cell physical IO interconnect bytes'``` and ```'cell physical IO interconnect bytes returned by smart scan'``` are all having same amount of processed bytes. Reason for poor performing smart scan can be found by looking at statistics related to cell blocks processed at three different layers(cache, data, txn) in the cell servers respectively. So in this case all the blocks(10127) were passed through cache layer but only 97 blocks out of them survived through data and txn layer due to minnscn optimization ```'cell blocks helped by minscn optimization'``` and these 97 blocks are the one which doesn't contain any data. Further we could see that consistent read were performed by constructing the consistent blocks using undo records, related statistic is ```'data blocks consistent reads - undo records applied'``` because we found lock byte set for all the 10000 blocks according to statistic ```'active txn count during cleanout'```. There was no optimization done for this case by the cell server and it did tried to optimize it by querying the commit cache ```'cell commit cache queries'``` but failed to get the details from commit cache as statistic ```'cell transactions found in commit cache'``` is zero. Impact of this can be measured directly with the single block I/O operations, in this case there were 117 total I/O requests out of which 88 were multi block reads and remaining 117-88=29 were single block reads as per the staistics ```'physical read total IO requests'``` and ```'physical read total multi block requests'```. 

#### After buffer cache flush 1 and 2 run:
In this case it did smart scan after updating the table and flushing the buffer cache. As per the statistics impact on smart scan is most worst in this case when compared with other cases. This case has used highest CPU due to 3025 physical I/O (physical read total IO requests) out of which there were just 84 multi block reads(physical read total multi block requests) and thus there were whopping 3025-84=2968 single block reads. All other behavioural statistics are similar to previous case. So if most of the blocks needed by smart scan are on disk having open transaction lock byte set then impact will be worst as it has to perform single block physical reads for these blocks. But if smart scan is performed immediately after first smart scan then impact will not be huge as consistent blocks already exists in cache because of first smart scan. As per the statisticcs for second run it's CPU usage less compared to first run but still high when considered in general with other cases. Even in this case there was no optimization done for this case by the cell server and it did tried to optimize it by querying the commit cache 'cell commit cache queries' but failed to get the details from commit cache as statistic ```'cell transactions found in commit cache'``` is zero.

#### After commit of the transaction:
This case is the most interesting one due to optimzation trick used by the cell server and it also demonstrate the internal working of smart scan when dealing with ```'delayed block cleanout'```. Smart scan was performed after updating the table, flushing buffer cache and then commiting the transaction. This ensures that blocks on the disk will have lock byte set though the transaction has been committed. Since Exadata leverages the commit cache optimization for this committed transaction, smart scan was very efficient by returning just 2573224 bytes(cell physical IO interconnect bytes returned by smart scan) out of 82714624 bytes (cell physical IO bytes eligible for predicate offload). Commit cache was queried for each block having lock byte set(cell commit cache queries) and was success in getting the details of the committed transaction(cell blocks helped by commit cache) and thus back and forth trips to database layer for consistent read were avoided. The staistic ```'active txn count during cleanout'``` says no active transaction found in all of the blocks though the lock byte is set, this clearly states that cell server has the detail of this committed transation in its commit cache. Due to this optimization there are no consistent block construction(data blocks consistent reads - undo records applied) and the 97 empty blocks were optimized due to minscn. This optimization is not present in non-exadata environment, but greatly used and helpful in Exadata environment while smart scan detects that block consist of lock byte set though transaction has been committed.

After clearing all the lock byte set in the blocks by performing conventional read of the table DEMO:
This case is nothing special, we update the table, flush buffer cache, commit the transaction and then perform cleanout of the lock byte set in the blocks residing on the disk by querying the entire table through conventional path. This is the reason why sometimes select statement generates redo as it has to clear lock byte set which inturn generates the redo for the activity. In this case minscn optimization was used for 97 empty blocks and smart scan has returned just 2573224 bytes out of 82714624 bytes similar to previous case. Also in this case we don't see any queries on commit cache(cell commit cache queries) as none of the blocks has lock byte set. This is a prefect demonstration of a case where there are no hurdles for smart scan unlike previous cases.

Conclusion
----------
Exadata cell server optimizations like Commit cache and minscn are very helpful when smart scan encounters active transaction in the blocks. Minscn is the intial approach where it compares the SCN found in the ITL of the block with the oldest active transaction scn(minscn) and decides whether it can read the block as it is or it has to construct the consistent block. When minscn is not helpful it will try to optimize smart scan by using commit cache consisting details of all the recent committed transaction. 

This article shows the impact of consistent read processing on smart scan and few optimization used by the cell server to overcome the overhead of communicating with database layer for consistent read processing. All these details will be very helpful when dealing with Smart Scan efficiency issues, most prominent indication of these kind of issues will be single block I/O along with Smart Scan. So whenever we see high contribution of ```'cell single block physical reads'``` along with ```'smart scan'``` we need to drill down further for finding the reason behind it.

