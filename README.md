# Bugs found in Database Management Systems

We have successfully discovered 33 bugs (16 fixed, 24 confirmed and 9 open reported) from real-world production-level DBMSs, including 5 bugs in MySQL, 2 bugs in PostgreSQL, 20 bugs in TiDB, 3 bugs in OpenGauss, and 3 bugs in Oceanbase.

We are thankful to the DBMS developers for responding to our bug reports and fixing the bugs that we found. Because the nondeterministic interleavings among operations challenges the reproducibility of the isolation-related bugs, there are 9 bugs can not be reproduced but open reported. In the future, we will aim to the research question about reproducing an isolation-related bug.

We have anonymized the corresponding issue and other personal information due to the double-blind reviewing constraint.

[TOC]



## Confirmed bugs

## TiDB

### Isolation-related Bugs

#### Bug#1 Dirty write in pessimistic transaction mode

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**     |
| ----------------------- | ------------------------------------------------------------ | ------------- |
| Mode Setting            | Set global tidb\_txn\_mode = 'pessimistic';                  | Success       |
| Schema Creation         | Create table table\_7\_2(a int primary key, b int, c double); | Success       |
| Database Initialization | Insert into table\_7\_2 values(676,-5012153, 2240641.4);     | Success       |
| 739                     | Begin transaction;                                           | Success       |
| 739                     | Update table\_7\_2 set b=-5012153, c=2240641.4 where a=676   | Success       |
| 723                     | Begin transaction;                                           | Success       |
| **723**                 | **Update table\_7\_2 set b=852150 where a=676**              | **Success X** |
| 739                     | Commit                                                       | Success       |

**Bug Description**

It first set transaction mode as pessimistic. Transaction 739 updates a record 676 in table\_ 7\_2 and holds an exclusive lock on the record 676. Before transaction 739 releases the exclusive lock on record 676, transaction 723 also successfully acquires an exclusive lock on record 676 and updates it, which results dirty write anomalies should be avoided in pessimistic transaction mode as described in TiDB official website.

#### Bug#2 Read inconsistency in snapshot isolation

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                        |
| ----------------------- | ------------------------------------------------------------ | -------------------------------- |
| Schema Creation         | Create table table\_7\_2(primarykey int primary key, attribute1 double,attribute6 double); | Success                          |
| Database Initialization | Insert into table\_7\_2 values(3873, 0.213, 0.234);          | Success                          |
| 904                     | Begin transaction;                                           | Success                          |
| 904                     | Update table\_8\_2 set attribute6=-0.386 where primarykey=3873 | Success                          |
| 904                     | Commit                                                       | Success                          |
| 914                     | Set @@global.tx\_isolation='REPEATABLE-READ';                | Success                          |
| 907                     | Begin transaction;                                           | Success                          |
| 907                     | Update table\_8\_2 set attribute6=0.484 where primarykey=3873 | Success                          |
| 907                     | Commit                                                       | Success                          |
| **914**                 | **Select attribute6 from table\_8\_2 where primarykey=3873** | **Success(attribute6=-0.368) X** |

**Bug Description**

There are two historical versions on attribute6 of record 3873 in table\_ 8\_2. The first one is -0.386 created by transaction 904; the second one is 0.484 created by transaction 907. Since the select operation of transaction 914 happens after the committing of transaction 907, transaction 914 should see the second one, i.e., 0.484, instead of the first one, i.e., -0.368. However, the select operation of transaction 914 returns the first version -0.368, which indicates there is a defect about the implementation of consistent read in TiDB.

#### Bug#3 Violating mutual exclusion

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Schema Creation         | create table t1(a int primary key, b  int); <br>create table t2(a int primary key, b int,  constraint fk1 foreign key(b) references t1(a));  <br/>create view view0(t2_a,t2_b,t1_b) as  select t2.a,t2.b,t1.b from t2,t1 where t2.b=t1.a; | Success   |
| Database Initialization | insert into t1 values(1,2);  <br/>insert into t1 values(2,3);  <br/>insert into t1 values(3,4);  <br/>insert into t1 values(4,5);  <br/>insert into t1 values(5,6);     <br/>insert into t2 values(1,2);  <br/>insert into t2 values(2,3);  <br/>insert into t2 values(3,4);  <br/>insert into t2 values(4,5);  <br/>insert into t2 values(5,1); | Success   |

| Operation ID | **Session1**                  | **Session2**                                        | **State**                                 |
| ------------ | ----------------------------- | --------------------------------------------------- | ----------------------------------------- |
| 1            | Begin transaction;            |                                                     | Success                                   |
| 2            |                               | Begin transaction;                                  | Success                                   |
| 3            | update t1 set b=12 where a=1; |                                                     | Success                                   |
| **4**        |                               | **select \* from view0 where t2\_a\>3 for update;** | **Success<br>Query Result(5,1,2)(4,5,6)** |
| 5            |                               | Commit;(Success)                                    | Success                                   |
| 6            | Commit;(Success)              |                                                     | Success                                   |

**Bug Description**

Operation 3 holds an exclusive lock on a record 1 in table t1 until operation 6 releases the lock. Due to the nature of exclusive locks, operation 4 attempts to acquire an exclusive lock on record 1 in table t1, which should be blocked. However, TiDB grants operation 4 an exclusive lock on record 1 in table t1, which indicates a locking violation.

#### Bug#4 Unnecessary locking a non-existing record

**Test Case**

| **Transaction ID**      | **Operation Detail**                      | **State** |
| ----------------------- | ----------------------------------------- | --------- |
| Schema Creation         | Create table t(a int primary key, b int); | Success   |
| Database Initialization |                                           | Success   |

| Operation ID | **Session1**                  | **Session2**                   | **State**                  |
| ------------ | ----------------------------- | ------------------------------ | -------------------------- |
| 1            | Begin transaction;            |                                | Success                    |
| 2            |                               | Begin transaction;             | Success                    |
| 3            | Update t set b=314 where a=1; |                                | Success with row count = 0 |
| **4**        |                               | **Insert into t values(1,3);** | **blocking X**             |
| 5            | Commit;                       |                                | Success                    |
| 6            |                               | Insert into t values(1,3);     | Success with row count = 1 |
| 7            |                               | Commit;                        | Success                    |


**Bug Description**

After investigating TiDB official website, the write operations of TiDB only locks the record that satisfies the conditions. Notice that the read operation of TiDB can avoid the phantom by the way of MVCC.

However, as shown in above test case, the update operation (Operation ID=3) locks a non-existing record that dose not satisfy its where condition. Additionally, the update operation (Operation ID=3) blocks the insertion operation of another transaction (Operation ID=6), which may lead to some performance issues about locking.

#### Bug#5 Query in transaction may return rows with same unique index column value

See [Query in transaction may return rows with same unique index column value · Issue #24195 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/24195) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Mode Setting            | Set global tidb\_txn\_mode = 'pessimistic';                  | Success   |
| Schema Creation         | create table t (c1 varchar(10), c2 int, c3 char(20), primary key (c1, c2)); | Success   |
| Database Initialization | insert into t values ('tag', 10, 't'), ('cat', 20, 'c');     | Success   |
| 2                       | Begin transaction;                                           | Success   |
| 2                       | update t set c1=reverse(c1) where c1='tag';                  | Success   |
| 4                       | Begin transaction;                                           | Success   |
| 4                       | insert into t values('dress',40,'d'),('tag', 10, 't');       | Success  |
| 2                       | Commit                                                       | Success   |
| 4                       | select \* from t use index(primary) order by c1,c2;          |           |

When primary key is clustered index, query returns

| C1    | C2   | C3   |
| ----- | ---- | ---- |
| cat   | 20   | c    |
| dress | 40   | d    |
| tag   | 10   | t    |

When primary key is nonclustered index, query returns

| C1    | C2   | C3   |
| ----- | ---- | ---- |
| cat   | 20   | c    |
| dress | 40   | d    |
| tag   | 10   | t    |
| tag   | 10   | t    |

**Bug Description**

The above test case runs on pessimistic transaction mode in TiDB. Session 1 initializes a table with two record. One of them is ('tag','10',t). Session 2 acquires a exclusive lock to update the record ('tag','10',t) as ('gat','10',t). Before update committed, session 4 successfully acquires a exclusive lock to insert a new record ('tag','10',t), which violates the principle of long duration lock.

After that, session 4 issues a query to see all record in table t. When primary key is clustered index, query return the newly inserted record ('tag','10',t). However, when primary key is nonclustered index, query return two identical records ('tag','10',t), which indicates a bug hidden in TiDB. We have reported such a test case in TiDB community. TiDB developers have acknowledged it and fixed it.

#### Bug#6 Select under repeatable read isolation level returns stale version of data

 [SELECT under repeatable read isolation level returns stale version of data · Issue #36718 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/36718) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                      |
| ----------------------- | ------------------------------------------------------------ | ------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2)); | Success                        |
| Database Initialization | insert into t values (412,409,258,17702);                    | Success                        |
| 281                     | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                        |
| 282                     | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE         | Success                        |
| 282                     | Begin transaction;                                           | Success                        |
| 282                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ); | Success                        |
| 282                     | commit;                                                      | Success                        |
| 281                     | Begin transaction;                                           | Success                        |
| 281                     | select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 89665 ) | Success                        |
| 281                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ) | Success                        |
| **281**                 | **select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 17702 )** | **Return (412,409,258,17702)** |
| 281                     | commit                                                       | Success                        |

**Bug Description**

Though Session-282 update the only row into (412,409,258, 40569). The last select query still returns a stale version of data for Key\<412,409,258\>, i.e. (412,409,258,17702).

#### Bug#7 Transaction can't read its own update in repeatable read.

See  [Transaction can't read its own update in repeatable read. · Issue #42487 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/42487) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                     |
| ----------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| Schema Creation         | create table table1 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2) NONCLUSTERED); | Success                                       |
| Database Initialization | insert into table1 values (1,1,1,1,1);                       | Success                                       |
| 1                       | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                                       |
| 2                       | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                                       |
| 2                       | Begin transaction;                                           | Success                                       |
| 1                       | Begin transaction;                                           | Success                                       |
| 2                       | update table1 set coAttr0\_0 = 2 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1; | Success                                       |
| 2                       | commit                                                       | Success                                       |
| 1                       | update table1 set coAttr0\_0 = 2 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1; | Success                                       |
| **1**                   | **select \* from table1 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1;** | **Return (1,1,1,1,2) instead of (1,1,1,1,1)** |

**Bug Description**

The last select query of session1 cannot get its own update.

When we update a different value in session2, the last query can get correct value, i.e., when we execute the transaction below:

It seems that some opt(skipping update) when two transactions update a same value may go wrong because the select and update hold different snapshot.

#### Bug#8 DROP DATABASE is not blocked while executing transaction.

See  [DROP DATABASE is not blocked while executing transaction · Issue #46943 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/46943) 

**Test Case**

| **Transaction ID** | **Operation Detail**     | **State** |
| ------------------ | ------------------------ | --------- |
| Schema Creation    | create table t (a int);  | Success   |
| 2                  | Begin transaction;       | Success   |
| 2                  | insert into t values(1); | Success   |
| 1                  | drop database test;      | No block  |

**Bug Description**

DROP DATABASE can be executed without blocking. And the idling transaction can also be committed without warning/error.

We tested DROP TABLE in TiDB and found this DDL will be blocked. And we also test DROP DATABASE in MySQL, which is also be blocked. So it seems obviously that DROP DATABASE should also be blocked.

Maybe locking all tables while altering database can help.

#### Bug#9 Transaction cannot be committed/rollbacked via binary protocol after TTL manager timed out

See  [Transaction cannot be committed/rollbacked via binary protocol after TTL manager timed out · Issue #49151 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/49151) 
**Test Case**

```js
  const c = await mysql.createConnection({ host: TIDB_HOST, port: 4000, user: "root", database: 'test' });
  try {
    await c.query("set @@tidb_general_log=1");
    await c.query("drop table if exists t1");
    await c.query("create table t1 (a int, b int)");
    await c.query("insert into t1 values (3, 2), (2, 3)");
    log("show config", await pp(c.query("show config where name like '%max-txn-ttl'")));
    await c.query("begin");
    await c.query("update t1 set a=4 where a=3");
    log("select(1):", await pp(c.query("select * from t1")));
    log("wait 45 seconds");
    await sleep(45*1000);
    log("continue");
    log("select(2):", await pp(c.query("select * from t1")));
    log("rollback(bin):", await pp(c.execute("rollback")));
    log("commit(bin):  ", await pp(c.execute("commit")));
    log("commit(txt):  ", await pp(c.query("commit")));
  } finally {
    await close(c);
  }
```

**Bug Description**

Both commit and rollback fail.

```bash
2023-12-04T10:49:43.457Z show config [{"Type":"tidb","Instance":"127.0.0.1:4000","Name":"performance.max-txn-ttl","Value":"30000"}]
2023-12-04T10:49:43.467Z select(1): [{"a":4,"b":2},{"a":2,"b":3}]
2023-12-04T10:49:43.467Z wait 45 seconds
2023-12-04T10:50:28.476Z continue
2023-12-04T10:50:28.478Z select(2): TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.481Z rollback(bin): TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.482Z commit(bin):   TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.486Z commit(txt):   {"fieldCount":0,"affectedRows":0,"insertId":0,"info":"","serverStatus":2,"warningStatus":0}
```

It's because binary protocal cannot finish an aborted transaction.



### Other types of bugs

#### Bug#10 Schema version check error

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**                                     |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------- |
|                    | Drop db0.table\_1\_2                                         | Success                                       |
| **723**            | **Update db1.table\_5\_1 set attribute2=8132130 where primarykey=6123** | **Exception:Information schema is changed X** |

**Bug Description**

During verifying the read committed isolation level recently developed by TiDB team, we find there is a checking schema problem. Specifically, the first line in above table modifies db0's schema information, while the second line in transaction 723 modifies db1's table with exception "information schema is changed". However, there is no modification on db1's schema information, which indicates a bug hidden in checking schema version.

#### Bug#11 Timestamp acquisition mechanism error in read committed

**Test Case**

| **Transaction ID** | **Operation Detail**                                 | **State**                |
| ------------------ | ---------------------------------------------------- | ------------------------ |
| **232**            | **Select \* from table\_2\_1 where primarykey=4323** | **Stall(never respond)** |

**Bug Description**

As definition, read committed isolation level sees a consistent view of database as of the beginning of an operation. Thus, each read operation in read committed isolation level should acquire a timestamp. Under the read committed isolation level recently developed by TiDB team, in order to optimize the performance of timestamp acquisition, asynchronous timestamp acquisition mechanism is adopted, but there are internal problems in this mechanism, as shown in the above table. Specifically, the read operation in above table never be responded from timestamp sever. We have help their developers fix this bugs.

#### Bug#12 Update BLOB data error

**Test Case**

| **Operation ID**        | **Operation Detail**                                         | **State**                             |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------- |
| Schema Creation         | create table t(pk0 int, pk1 int, pk2 int, attr3 blob);       | Success                               |
| Database Initialization | insert into t values(1,1,1,null);                            |                                       |
| 1                       | Update t set attr3=FILE("./data\_case/obj/12obj\_file.obj") where pk0=1 and pk1=1 and pk2= 1 | Success                               |
| 2                       | Update t set attr3=FILE("./data\_case/obj/12obj\_file.obj") and other column where pk0 = 1 and pk1 = 1 and pk2 = 1 | Success                               |
| **3**                   | **Select attr3 from t where pk0 = 1 and pk1 = 1 and pk2 = 1 for update** | **Success <br>Return attr3 = NULL X** |

**Bug Description**

For BLOB data type, when the new value and the old value written by the update operation are for the same binary file, the value actually written is null and success is returned, which indicates a BLOB-related bug hidden in TiDB.

#### Bug#13 Bug in Start Transaction

**Test Case**

| **Transaction ID** | **Operation Detail**                                     | **State** |
| ------------------ | -------------------------------------------------------- | --------- |
| **1**              | **Start transaction read only with consistent snapshot** | **fail**  |

**Bug Description**

As for the start transaction statement, the TiDB official websiate shows that it supports the keywords "with consistent snapshot" and "read only". However, in fact, we found that TiDB cannot support these two keywords at the same time.

#### Bug#14 JDBC ResultSetMetaData.getColumnName for view query returns the attribute name defined in the table instead of the one defined in the view

See  [JDBC ResultSetMetaData.getColumnName for view query returns the attribute name defined in the table instead of the one defined in the view · Issue #24227 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/24227) 

**Test Case**

 Just query a view with renamed attribute and print `ResultSetMetaData.getColumnName`. 

```java
import lombok.Cleanup;

import java.sql.*;

public class Main {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306?serverTimezone=UTC&useServerPrepStmts=true&cachePrepStmts=true";
        String username = "root";
        String password = "root";

        String dropDatabase = "drop database if exists RSMDTestDB;";
        String createDatabase = "create database RSMDTestDB;";
        String useDatabase = "use RSMDTestDB;";
        String createTable0 = "create table table0(t0c0 int, t0c1 int, t0c2 int);";
        String createView0 = "create view view0(v0c0, v0c1, v0c2) as select t0c0, t0c1, t0c2 from table0;";
        String selectView0 = "select * from view0;";
        String selectView0WithAlias = "select v0c0 as alias_v0c0, v0c1 as alias_v0c1, v0c2 as alias_v0c2 from view0;";

        @Cleanup
        Connection conn = DriverManager.getConnection(url, username, password);
        Statement stat = conn.createStatement();
        stat.execute(dropDatabase);
        stat.execute(createDatabase);
        stat.execute(useDatabase);
        stat.execute(createTable0);
        stat.execute(createView0);

        // print DBMS info
        DatabaseMetaData dbMD = conn.getMetaData();
        System.out.println(dbMD.getDatabaseProductVersion());

        // select from view without alias
        ResultSet rs = stat.executeQuery(selectView0);
        ResultSetMetaData md = rs.getMetaData();
        System.out.println("SELECT WITHOUT ALIAS");
        System.out.printf("%-10s\t%-10s\t%-10s\n", "Index", "ColName", "ColLabel");
        for(int ind=1; ind <= md.getColumnCount(); ind++) {
            System.out.printf("%-10d\t%-10s\t%-10s\n", ind, md.getColumnName(ind), md.getColumnLabel(ind));
        }
        rs.close();

        // select from view with alias
        rs = stat.executeQuery(selectView0WithAlias);
        md = rs.getMetaData();
        System.out.println("SELECT WITH ALIAS");
        System.out.printf("%-10s\t%-10s\t%-10s\n", "Index", "ColName", "ColLabel");
        for(int ind=1; ind <= md.getColumnCount(); ind++) {
            System.out.printf("%-10d\t%-10s\t%-10s\n", ind, md.getColumnName(ind), md.getColumnLabel(ind));
        }
        rs.close();

        // drop database
        stat.execute(dropDatabase);
    }
}
```

**Bug Description**

For MySQL, ResultSetMetaData.getColumnName returns the name of a column defined in the view, and ResultSetMetaData.getColumnLabel returns the alias given in the query.

However, in TiDB, ResultSetMetaData.getColumnName returns the name of a column defined in the table instead of in the view, while ResultSetMetaData.getColumnLabel returns the alias given in the query.

However, considering that when users query a view, they treat it as an abstract table, and do not care whether it is a table or a view. Perhaps it is more reasonable to return the name of the attribute listed in the view definition. This is the behavior of MySQL.

Moreover, if the user uses the as keyword to alias a attribute when querying the view, it seems that the attribute name defined in the view cannot be obtained from ResultSetMetaData.

#### Bug#15 Query Error in information\_schema.slow\_query

See  [Query Error in information_schema.slow_query · Issue #28069 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/28069) 

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**              |
| ------------------ | ------------------------------------------------------------ | ---------------------- |
| Schema Creation    | Use information\_schema                                      | Success                |
| 1                  | select Txn\_start\_ts,time from slow\_query where time \> '2021-08-13 16:18:37.313976' limit 10; | Success returns 1 rows |
| **1**              | **select Txn\_start\_ts,time from slow\_query where time \> '2021-08-10 16:18:37.313976' limit 10;** | **Empty set**          |

**Bug Description**

Since the last query select a larger scale(\>2021-08-10) tha the first one (2021-08-13), it should not return less result.

#### Bug#16 Min/Max function applied on partition key causes more than 5000x performance degradation

See  [Min/Max function applied on partition key causes more than 5000x performance degradation · Issue #41462 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/41462) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                |
| ----------------------- | ------------------------------------------------------------ | ------------------------ |
| Schema Creation         | CREATE TABLE IF NOT EXISTS test\_table ( a INT NOT NULL, b INT NOT NULL, c INT NOT NULL, PRIMARY KEY (a,b,c)) PARTITION BY HASH(a) PARTITIONS 5; | Success                  |
| Database Initialization | Insert 10k tuples.For any tuple, column a and b have same value, generated from [1-10], and column c is increasing by 1. | Success                  |
| **1**                   | **select min(a) from test\_table;**                          | **Success(1min 54 sec)** |
| **1**                   | **select max(a) from test\_table;**                          | **Success(2min 3 sec)**  |
| 1                       | select distinct(a) from test\_table order by a limit 1;      | Success(21ms)            |
| 1                       | select min(b) from test\_table;                              | Success(14ms)            |

**Bug Description**

Expect to see that four queries should take a similar amount of time.

But first two queries took a very long time to execute(1 min 54.51 sec, 2 min 3.46 sec). While the last two act quickly(21.3ms, 14.4ms). There is a 5000x performance degradation between them.

And we found that the first two queries used Limit but no TopN on tikv, which may lead to a large data transfer from tikv to root. This may make query slow(even on the same machine).

#### Bug#17 Create index is blocked when no /tmp/tidb/tmp\_ddl-4000.

See  [Create index is blocked when no /tmp/tidb/tmp_ddl-4000 · Issue #45624 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/45624) 

**Test Case**

| **Transaction ID**  | **Operation Detail**       | **State** |
| ------------------- | -------------------------- | --------- |
| Schema Creation     | Create table t(a int)      | Success   |
| **Schema Creation** | **Create index t1 on (a)** | **block** |

**Bug Description**

The ddl is blocked. When kill this ddl, return error mes cannot get disk capacity at /tmp/tidb/tmp\_ddl-4000: no such file or directory. And there is no tidb under /tmp. Index can be created after making dir /tmp/tidb/tmp\_ddl-4000 manually.

It is because TiDB does not build such dir automatically after checking.

#### Bug#18 Join between blob type with matching returns incorrect result.

See  [Join between blob type with matching returns incorrect result. · Issue #50393 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/50393) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Schema Creation         | create table t2(a blob);<br> create table t3(a blob);        | Success   |
| Database Initialization | insert into t2 values(0xC2A0);<br> insert into t3 values(0xC2); | block     |
| **1**                   | **select * from t2,t3 where t2.a like concat("%",t3.a,"%");** |           |

In MySQL(v8.0.33), the last query returns one row.

```sql
mysql> select * from t2,t3 where t2.a like concat("%",t3.a,"%");
+------------+------------+
| a          | a          |
+------------+------------+
| 0xC2A0     | 0xC2       |
+------------+------------+
1 row in set (0.01 sec)
```

However, TiDB returns an empty set.

```sql
> select * from t2,t3 where t2.a like concat("%",t3.a,"%");
Empty set (0.00 sec)
```

 It looks weird since in the case below, TiDB works well. 

```sql
create table t2(a blob);
create table t3(a blob);
insert into t2 values(0xC2A020);
insert into t3 values(0xC2A0);
select * from t2,t3 where t2.a like concat("%",t3.a,"%");
```

In this case, two DBMSs both returns one row. The charsets of TiDB and MySQL are the same(utf8mb4). 

This bug may be relative to how match operation(like) compare two objects. When taking two bytes(as java's char) as a compare unit, the second object in the first case are different from the first one instead of contained by the first object. However, in the second case, the second object is contained by the first one. Moreover, when taking one byte as a compare unit(as c's char), the second object is contained by the first one in both cases.

## MySQL

### Isolation-related Bugs

#### Bug#19 Select under repeatable read isolation level returns stale version of data

See  [MySQL Bugs: #108015: select under repeatable read isolation level returns stale version of data](https://bugs.mysql.com/bug.php?id=108015) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                      |
| ----------------------- | ------------------------------------------------------------ | ------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2)); | Success                        |
| Database Initialization | insert into t values (412,409,258,17702);                    | Success                        |
| 281                     | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                        |
| 282                     | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE         | Success                        |
| 282                     | Begin transaction;                                           | Success                        |
| 282                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ); | Success                        |
| 282                     | commit;                                                      | Success                        |
| 281                     | Begin transaction;                                           | Success                        |
| 281                     | select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 89665 ) | Success                        |
| 281                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ) | Success                        |
| **281**                 | **select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 17702 )** | **Return (412,409,258,17702)** |
| 281                     | commit                                                       | Success                        |

**Bug Description**

Though Session-282 update the only row into (412,409,258, 40569). The last select query still returns a stale version of data for Key\<412,409,258\>, i.e. (412,409,258,17702).

#### Bug#20 Two parallel threads trigger error code '1032 Can't find record in 'table'

See  [MySQL Bugs: #103891: Two parallel threads trigger error code '1032 Can't find record in 'table'](https://bugs.mysql.com/bug.php?id=103891) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                        |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| Schema Creation         | create table t(pkattr0 integer, coattr1 varchar(100), coattr2 varchar(100), coattr3 varchar(100), coattr4 integer, coattr5 varchar(100), coattr6 varchar(100), primary key(pkattr0)); | Success                                          |
| Database Initialization | delimiter \$<br>create procedure p()<br/>begin<br/>declare done int default false;<br/>declare v\_pkattr0 integer;<br/>declare v\_coattr1 varchar(100);<br/>declare v\_coattr2 varchar(100);<br/>declare v\_coattr3 varchar(100);<br/>declare v\_coattr4 varchar(100);<br/>declare v\_coattr5 varchar(100);<br/>declare v\_coattr6 varchar(100);<br/>declare cur1 cursor for select pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6 from t order by pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6;<br/>-- declare continue handler for sqlexception begin end;<br/>declare continue handler for not found set done = true;<br/>repeat<br/>    set session transaction isolation level read uncommitted;<br/>    if rand()\<0.5 <br/>          then start transaction; <br/>    end if;<br/>open cur1;<br/>     read\_loop: loop<br/>         fetch cur1 into v\_pkattr0, v\_coattr1, v\_coattr2, v\_coattr3, v\_coattr4, v\_coattr5, v\_coattr6;<br/>         if done then<br/>             leave read\_loop;<br/>         end if;<br/>      end loop;<br/>close cur1;<br/>if rand()\<0.5 then commit; end if;<br/>if rand()\<0.5 then start transaction; end if;<br/>if rand()\<0.5 then delete from t where pkattr0 = 1; end if;<br/>if rand()\<0.5 then commit; end if;set session transaction isolation level read uncommitted;<br/>if rand()\<0.5 then start transaction; end if;<br/>if rand()\<0.5 then replace into t(pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6) values("1", "varchar1", "varchar2", "varchar3", "2021", "varchar5", "varchar6"); end if;<br/>if rand()\<0.5 then commit; end if;<br/>until 1=2 end repeat; <br/>end ​\$ <br/>delimiter ; | Success                                          |
| 1                       | call p();                                                    | Success                                          |
| 2                       | call p();                                                    | Success                                          |
| **3**                   | **call p();**                                                | **ERROR 1032 (HY000): Can't find record in 't'** |

**Bug Description**

The expected result should not throw '1032' error.

## Oceanbase

### Other types of bugs

#### Bug#21 Create View Error

**Test Case**

| **Transaction ID**  | **Operation Detail**                                         | **State**                        |
| ------------------- | ------------------------------------------------------------ | -------------------------------- |
| Schema Creation     | create table table0(pkId integer,pkAttr0 integer,pkAttr1 integer,pkAttr2 integer,pkAttr3 integer,pkAttr4 integer,coAttr0\_0 integer,coAttr0\_1 decimal(10, 0),coAttr0\_2 varchar(100),primary key (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4)); | Success                          |
| Schema Creation     | alter table table0 add index index\_pk(pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4); | Success                          |
| Schema Creation     | create table table1(pkId integer,pkAttr0 integer,pkAttr1 integer,coAttr0\_0 varchar(100),coAttr0\_1 integer,coAttr0\_2 varchar(100),primary key (pkAttr0, pkAttr1)); | Success                          |
| Schema Creation     | alter table table1 add index index\_pk(pkAttr0, pkAttr1);    | Success                          |
| Schema Creation     | alter table table1 add index index\_commAttr0(coAttr0\_0, coAttr0\_1, coAttr0\_2); | Success                          |
| Schema Creation     | create table table2(pkId integer,pkAttr0 integer,pkAttr1 integer,pkAttr2 integer,pkAttr3 integer,pkAttr4 integer,pkAttr5 integer,pkAttr6 integer,coAttr0\_0 decimal(10, 0),coAttr0\_1 varchar(100),coAttr0\_2 varchar(100),fkAttr0\_0 integer,fkAttr0\_1 integer,fkAttr0\_2 integer,fkAttr0\_3 integer,fkAttr0\_4 integer,fkAttr1\_0 integer,fkAttr1\_1 integer,primary key (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6),foreign key (fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4) references table0 (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4),foreign key (fkAttr1\_0, fkAttr1\_1) references table1 (pkAttr0, pkAttr1)); | Success                          |
| Schema Creation     | alter table table2 add index index\_pk(pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6); | Success                          |
| Schema Creation     | alter table table2 add index index\_fk0(fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4); | Success                          |
| Schema Creation     | alter table table2 add index index\_fk1(fkAttr1\_0, fkAttr1\_1); | Success                          |
| **Schema Creation** | **create view view2 (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6, fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4, fkAttr1\_0, fkAttr1\_1, coAttr0\_0, coAttr0\_1, coAttr0\_2, coAttr1\_0, coAttr1\_1, coAttr1\_2)asselect table2.pkAttr0,table2.pkAttr1,table2.pkAttr2,table2.pkAttr3,table2.pkAttr4,table2.pkAttr5,table2.pkAttr6,table2.fkAttr0\_0,table2.fkAttr0\_1,table2.fkAttr0\_2,table2.fkAttr0\_3,table2.fkAttr0\_4,table2.fkAttr1\_0,table2.fkAttr1\_1,table2.coAttr0\_0,table2.coAttr0\_1,table2.coAttr0\_2,table0.coAttr0\_0,table0.coAttr0\_1,table0.coAttr0\_2from table2,table0where table2.fkAttr0\_0 = table0.pkAttr0and table2.fkAttr0\_1 = table0.pkAttr1and table2.fkAttr0\_2 = table0.pkAttr2and table2.fkAttr0\_3 = table0.pkAttr3and table2.fkAttr0\_4 = table0.pkAttr4** | **Error: duplicate column name** |

**Bug Description**

When creating a correct view defined above, an error that cannot be imported may occur, and the error message "duplicate column name" will be reported. However, after careful inspection, there are no duplicate column names in the statement, and the same DDL statement can run normally on MySQL 5.7, which indicates a schema-related bug hidden in Oceanbase.

#### Bug#22 Different join results when change calculate order.

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**              |
| ----------------------- | ------------------------------------------------------------ | ---------------------- |
| Schema Creation         | create table t0 (a decimal(5,3));create table t1 (a double); | Success                |
| Database Initialization | insert into t0 values(0.01);insert into t1 values(0.1);      | Success                |
| **281**                 | **select \* from t0,t1 where t0.a = t1.a - t1.a + t0.a;**    | **Returns (0.01,0.1)** |
| **282**                 | **select \* from t0,t1 where t0.a = t0.a + t1.a - t1.a;**    | **Returns Empty set**  |

**Bug Description**

The two queries are expected to have the same result. The different results are caused by the float deviation in different calculation order.

## OpenGauss

### Isolation-related bugs

#### Bug#23 JDBC Get A Wrong Snapshot When Start

**Test Case**


| **Transaction ID**      | **Operation Detail**    | **State**       |
| ----------------------- | ----------------------- | --------------- |
| Schema Creation         | create table t (a int); | Success         |
| Database Initialization | insert into t values(1) | Success         |
| 281(jdbc)               | begin;                  | Success         |
| 282(jdbc)               | update t set a = 2;     | Success         |
| **281(jdbc)**           | **select \* from t;**   | **Returns (1)** |

**Bug Description**

The last query is expected to return (2), as that does in CLI, but it returns (1).

It is because OpenGauss is designed to get snapshot at the first non-transaction-control statement of each transaction, but jdbc requires a snapshot at begin.

## Open reported bugs

## TiDB

### Isolation-related bugs

#### Bug#24 Update with sub query uses incorrect snapshot.

See  [Update with sub query uses incorrect snapshot in RR isolation level · Issue #45677 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/45677) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                      | **State**                          |
| ----------------------- | --------------------------------------------------------- | ---------------------------------- |
| Schema Creation         | create table t(a int, b int);                             | Success                            |
| Database Initialization | insert into t values(1,1);                                | Success                            |
| 1                       | begin;                                                    | Success                            |
| 2                       | update t set b=2 where a=1;                               | Success                            |
| **1**                   | **update t set b=3 where b=(select b from t where b=2);** | **Success, affects 0 row**         |
| 1                       | commit                                                    | Success                            |
| **1**                   | **select \* from t;**                                     | **Returns (1,2) instead of (1,3)** |

**Bug Description**

Update is designed to read the last committed data. Thus the update in session 1 should read the data version (1,2). However, the sub query in update is executed as a single query and uses snapshot read, which read a earlier version(1,1).The snapshot read by sub query is different from that read by update and breaks the consistency in this single update SQL(with sub query), because a single SQL reads two snapshots.We test the same case in MySQL(v8.0.33), whose sub query read consistent snapshot as update.

Thus this execution is incompatible with MySQL.

## MySQL

### Isolation-related bugs

#### Bug#25 Predicate Lock ERROR

See  [MySQL Bugs: #105988: The problem about predicate lock in Serializable isolation level](https://bugs.mysql.com/bug.php?id=105988) 

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**     |
| ------------------ | ------------------------------------------------------------ | ------------- |
| Schema Creation    | Create table table0 (pkId integer, pkAttr0 integer, coAttr0\_0 integer, primary key(pkAttr0)); | Success       |
| Mode Setting       | Set autocommit = 0;                                          | Success       |
| 69264              | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;        | Success       |
| 69269              | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;        | Success       |
| 69264              | START TRANSACTION;                                           | Success       |
| 69264              | Insert into `table0`(`pkId`, `pkAttr0`, `coAttr0_0`) values(225, 225, 35704); | Success       |
| 69269              | START TRANSACTION;                                           | Success       |
| **69269**          | **Select `pkAttr0`, `coAttr0_0` from `table0` where ( `pkAttr0` = 225 );** | **Success X** |
| 69264              | rollback                                                     | Success       |

**Bug Description**

We disable autocomit and set both the above two transactions as serializable isolation level, as shown in the second to third lines.

As definition, InnoDB implicitly converts all plain SELECT statements to SELECT ... FOR SHARE if autocommit is disabled. Therefore, the seventh line should acquire a range-level shared lock to protect all records whose pkAttr0 is 225.

However, the fifth line has acquired a record-level exclusive lock on a record whose pkAttr0 is 225. The exclusive lock acquired by the fifth line is imcompatible with the shared lock acquired by the seventh line, which indicates a bug hidden in range-level lock.

#### Bug#26 Read uncommitted transaction reads the result of a failed write operation

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                          |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------- |
| Schema Creation         | set global innodb\_deadlock\_detect=off;create table t(a int primary key, b int); | Success                            |
| Database Initialization | insert into t values(1,2);insert into t values(2,4);         | Success                            |
| 1                       | begin;                                                       | Success                            |
| 2                       | begin;                                                       | Success                            |
| 2                       | set session transaction isolation level read uncommitted;    | Success                            |
| 3                       | begin;                                                       | Success                            |
| 3                       | set session transaction isolation level read uncommitted;    | Success                            |
| 2                       | delete from t where a=1;                                     | Success                            |
| 3                       | update t set b=321 where a=2;                                | Success                            |
| 2                       | update t set b=1421 where a=2;                               | Success                            |
| 3                       | insert into t value(1,1231);                                 | Rollback                           |
| 1                       | select \* from t where a=1;                                  | Returns (1,2) instead of empty set |

**Bug Description**

Transaction 2 writes new versions on records 1 and 2 successively, while Transaction 3 writes new versions on records 2 and 1 successively. So there is a deadlock situation between transaction 2 and 3. Before the deadlock between transaction 2 and 3 timeouts, another read uncommitted transaction 1 launch a query to read the record 1 that has been modified by transaction 2 and 3 successively. Since the second write operation of transaction 3 are failed due to deadlock, we should not see its write results. Therefore, as expected, the query result of transaction 1 should be the write result of transaction 2. However, the query result of transaction 1 is the write result before transaction 2, which is weird. We think there may be a subtle bug hidden in the current version of MySQL.

### Other types of Bugs

#### Bug#27 Update BLOB data error

**Test Case**

| **Operation ID**        | **Operation Detail**                                         | **State**                          |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------- |
| Schema Creation         | create table t(pk0 int, pk1 int, pk2 int, attr3 blob);       | Success                            |
| Database Initialization | insert into t values(1,1,1,null);                            |                                    |
| 1                       | Update t set attr3=FILE("./data_case/obj/12obj_file.obj") where pk0=1 and pk1=1 and pk2= 1 | Success                            |
| 2                       | Update t set attr3=FILE("./data_case/obj/12obj_file.obj") and other column where pk0 = 1 and pk1 = 1 and pk2 = 1 | Success                            |
| **3**                   | **Select attr3 from t where pk0 = 1 and pk1 = 1 and pk2 = 1 for update** | **Success  Return attr3 = NULL X** |

**Bug Description**

For BLOB data type, when the new value and the old value written by the update operation are for the same binary file, the value actually written is null and success is returned, which indicates a BLOB-related bug hidden in MySQL.

## PostgreSQL

### Isolation-related bugs

#### Bug#28 Write skew in SSI

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**   |
| ------------------ | ------------------------------------------------------------ | ----------- |
| 206                | Select attribute1 from table\_7\_1 where primarykey= 832     | Success     |
| 204                | Select attribute1 from table\_7\_4 where primarykey= 1460    | Success     |
| 206                | Update table\_7\_4 set attribute where primarykey=1460       | Success     |
| 204                | Update table\_7\_1 set attribute1 = -635092 where primarykey= 832 | Success     |
| 204                | Commit                                                       | Success     |
| **206**            | **Commit**                                                   | **Success** |

**Bug Description**

Transaction 206 reads a record 832 in table\_ 7\_ 1，then transaction 204 writes a new record to cover it, so transactions 206 to 204 have a RW dependency. Similarly, transaction 204 reads the record 1460 in table\_ 7\_ 4, then transaction 206 writes a new record to cover it, so transactions 204 to 206 have a RW dependency. Finally, transactions 204 to 206 generate a circular dependency, that is, write skew anomalies that should be avoided in Snapshot Isolation Level of PostgreSQL.

#### Bug#29 Two different versions of the same row of records are returned in one query

See  [PostgreSQL: BUG #17017: Two versions of the same row of records are returned in one query](https://www.postgresql.org/message-id/17017-c37dbbadb77cfde9%40postgresql.org) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                 | **State**                               |
| ----------------------- | ---------------------------------------------------- | --------------------------------------- |
| Schema Creation         | Create Table t(a int primary key, b int);            | Success                                 |
| Database Initialization | Insert into t values(1,2);Insert into t values(2,3); | Success                                 |
| 1                       | begin;                                               | Success                                 |
| 1                       | set transaction isolation level repeatable read;     | Success                                 |
| 1                       | Select \* from t where a=1;                          | Success                                 |
| 2                       | Begin;                                               | Success                                 |
| 2                       | set transaction isolation level read committed;      | Success                                 |
| 2                       | Delete from t where a=2;                             | Success                                 |
| 2                       | Commit;                                              | Success                                 |
| 1                       | Insert into t values(2,4);                           | Success                                 |
| **1**                   | **Select \* from t where a=2;**                      | **Returns (2,4)(2,3) instead of (2,4)** |

**Bug Description**

According to the definition of snapshot isolation, a query in a transaction should always see a consistent view of the database. That is, it should see a consistent version for each record in the database. However, bug#20 shows that two different versions of the same record are returned to a query under the snapshot isolation of PostgreSQL. This certainly violates the definition of snapshot isolation. Thus, we report it to the PostgreSQL community.

## OpenGauss

### Isolation-related Bugs

#### Bug#30 Violating First-Updater-Wins

**Test Case**

| Transaction ID | Session1                                                     | Session2                                                     | State                     |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------- |
| 2              |                                                              | Begin;                                                       | Success                   |
| 2              |                                                              | set session transaction isolation level repeatable read;     | Success                   |
| 2              |                                                              | update "table0" set "coAttr31_0" = 1048.0 where ( "pkAttr0" = 280 ) and ( "pkAttr1"  = 241 ) and ( "pkAttr2" = ‘vc204’ ) and ( "pkAttr3" =  ‘vc361’ ) and ( "pkAttr4" = 363 );--row count=1 | Success                   |
| 1              | Begin;                                                       |                                                              | Success                   |
| 1              | set session transaction isolation level repeatable read;     |                                                              | Success                   |
| 1              | select "pkAttr0", "pkAttr1", "pkAttr2", "pkAttr3", "pkAttr4", "pkAttr5", "pkAttr6", "pkAttr7", "fkAttr0\_0", "fkAttr0\_1", "fkAttr0\_2", "fkAttr0\_3", "fkAttr0\_4" from "view0" where ( "fkAttr0\_0" = 94 ) and ( "fkAttr0\_1" = 239 ) or ( "fkAttr0\_2" \< 'vc119' ) and ( "fkAttr0\_3" \> 'vc81u' ) and ( "fkAttr0\_4" = 278 ) ; |                                                              | Success                   |
| 2              |                                                              | commit                                                       | Success                   |
| **1**          | **delete from "table0" where ( "pkAttr0" = 280 ) and ( "pkAttr1" = 241 ) and ( "pkAttr2" = 'vc204' ) and ( "pkAttr3" = 'vc361' ) and ( "pkAttr4" = 363 );** |                                                              | **Success --row count=1** |

**Bug Description**

Transaction 1 starts before transaction 2 commit, and both transaction 1 and 2 write a new version on a record (280, 241,'vc204' , 'vc361' ,363 ). Therefore, transaction 1 and 2 are a pair of concurrent transaction, which should be avoided by first updater wins mechanism in OpenGauss.

#### Bug#31 Violating Read-Consistency

**Test Case**

Create table table2 (primarykey int primary key, coAttr25\_0 int);

Insert into table2 values(6,0);

Insert into table2 values(7,0);

| **Transaction ID** | **Session1**                                                 | **Session2**                                                 | **State**                                                |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------------- |
| 1                  | Begin;                                                       |                                                              | Success                                                  |
| 1                  | set session transaction isolation level repeatable read;     |                                                              | Success                                                  |
| 1                  | update "table2" set "coAttr25\_0" = 78354, where "primaryKey" = 7; |                                                              | Success                                                  |
| 2                  |                                                              | Begin;                                                       | Success                                                  |
| 2                  |                                                              | set session transaction isolation level repeatable read;     | Success                                                  |
| 2                  |                                                              | "update "table2" set " coAttr25\_0" = 14 where "primaryKey" = 6; | Success                                                  |
| 2                  |                                                              | Commit                                                       | Success                                                  |
| **1**              | **Select "primaryKey", "fkAttr0\_0", "coAttr25\_0" from "table2";** |                                                              | **Success returns"primaryKey":"6", "coAttr25\_0": "14"** |

**Bug Description**

Transaction 1 launch a update operation while fetches a consistent snapshot. According to the rule of repeatable read isolation level, any operation in transaction 1 should sees a same snapshot., so transaction 1 should not see the write result created by transaction 2. However, transaction 1 sees the write result created by transaction 2, which indicates a consistency read violation.

## Oceanbase

### Isolation-related bugs

#### Bug#32 Read inconsistency

**Test Case**

| Transaction ID | Session1                                                     | Session2                                                     | State                                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1              | set session transaction isolation level repeatable read;     |                                                              | Success                                                      |
| 2              |                                                              | set session transaction isolation level repeatable read;     | Success                                                      |
| 1              | START TRANSACTION READ ONLY,WITH CONSISTENT SNAPSHOT;        |                                                              | Success                                                      |
| 2              |                                                              | START TRANSACTION;                                           | Success                                                      |
| 2              |                                                              | update table0 set coAttr17 = 19635, coAttr18 = 1244, coAttr19 = 92947 where ( pkAttr0 = 'vc239' ) and ( pkAttr1 = 'vc234' ) and ( pkAttr2 = 'vc233' ); | Success, affects 1 row                                       |
| 2              |                                                              | COMMIT                                                       | Success                                                      |
| **1**          | **select pkAttr0, pkAttr1, pkAttr2, coAttr17, coAttr18, coAttr19 from table0 order by pkAttr0 ;** |                                                              | **Success returns (vc239, vc234, vc233, 19635, 1244, 92947)** |

**Bug Description**

After confirmation with developer, the repeatable read isolation level of Oceanbase is consistent with snapshot isolation as defined in the paper "A Critique of ANSI SQL Isolation Levels". Specifically, snapshot isolation is define as:

1. A read operation sees a snapshot as of the start of the transaction.
2. No transaction modify the record that has been modified by another concurrent transaction.

However, Oceanbase violates the first rule of snapshot isolation. Specifically, after transaction 1 obtains the consistency snapshot, another parallel transaction 2 issues a write operation. Transaction 1 should not see the write result created by transaction 2. In practice, transaction 1 sees it.
