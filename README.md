# Bugs found in Database Management Systems

We have successfully discovered 16 bugs (4 fixed, 15 confirmed and 1 open reported) from real-world production-level DBMSs, including 2 bugs in MySQL, 1 bugs in TDSQL, 11 bugs in TiDB, and 2 bugs in Oceanbase.

We are thankful to the DBMS developers for responding to our bug reports and fixing the bugs that we found. 



| ID   | 数据库               | Issue           | Bug 描述                                                     | Bug 状态             |
| :--- | :------------------- | :-------------- | :----------------------------------------------------------- | :------------------- |
| 1    | **TiDB**             | #59562          | 注入网络分区引发断言失败，多表连接查询无法执行。             | Confirmed            |
| 2    | TiDB                 | #59567          | 在索引创建期间注入网络分区故障，恢复后索引查询返回的数据量不一致。 | Confirmed            |
| 3    | TiDB                 | #59680          | 并发事务执行同时进行数据分区合并操作，导致出现相同主键行，违反主键约束。 | Fixed                |
| 4    | TiDB                 | #59727          | 修改列类型 DDL 与事务并发执行，导致出现相同主键行，违反主键约束。 | Confirmed            |
| 5    | TiDB                 | #59781          | 并发执行插入语句和 select for update 语句，未能检测到死锁，与 Mysql 行为不一致 | Open            |
| 6    | TiDB                 | #59960          | 多客户端同时修改同一列的列类型与空值约束，导致该列索引返回结果不一致。 | Fixed                |
| 7    | TiDB                 | #59961          | 多客户端并发修改同一列的元信息并添加索引，致使索引查询结果异常。 | Fixed                |
| 8    | **MySQL**            | #117735         | 并发执行事务与在线 DDL 操作时，触发错误的死锁检测逻辑。      | Confirmed            |
| 9    | **OceanBase**/ MySQL | #2235 / #117563 | 将 BigInt 列的自增起始值更新后，允许修改列类型为 Int，导致后续数据插入报错（应禁止此类类型更改）。 | Confirmed/ Confirmed |
| 10   | OceanBase            | #2243           | 事务并发执行时添加唯一约束，最终表既存在唯一约束，又包含相同值的重复行。 | Confirmed                 |
| 11   | **TDSQL**            | /               | 并发执行添加索引与包含 update 的事务时，触发错误的死锁检测逻辑。 | Fixed                |
| 12   | TiDB                 | #60245          | 多线程并发执行事务与修改列类型的 DDL 操作，表数据项被错误置为  Null. | Confirmed            |
| 13   | TiDB                 | #57650          | Update 语句执行时发生错误的类型转换，导致数据错误截断。      | Confirmed            |
| 14   | TiDB                 | #57640          | Delete 语句在包含与运算的谓词条件时，未正确处理溢出问题，导致异常抛出。 | Confirmed            |
| 15   | TiDB                 | #60261          | 多线程并发更改列类型为 decimal，导致相同数据项被重复复制。   | Confirmed            |

