

# 事务隔离

## 事务隔离级别

SQL标准定义了四种隔离级 

读未提交(READ UNCOMMITTED)
读已提交(READ COMMITTED)
可重复读(REPEATABLE READ)
可序列化(SERIALIZABLE)

## 并发事物的四种现象

1. 脏读 
   一个事务读取了另一个并行未提交事务写入的数据
2. 不可重复读
   一个事务重新读取之前读取过的数据，发现该数据已经被另一个事务(在初始读之后提交)修改。
3. 幻读	
   一个事务重新执行一个返回符合一个搜索条件的行集合的查询， 发现满足条件的行集合 因为另一个最近提交的事务而发生了改变。
4. 序列化异常
   成功提交一组事务的结果与这些事务所有可能的串行执行结果都不一致。

| 隔离级别 | 脏读              | 不可重复读 | 幻读              | 序列化异常 |
| -------- | ----------------- | ---------- | ----------------- | ---------- |
| 读未提交 | 允许，但不在 PG中 | 可能       | 可能              | 可能       |
| 读已提交 | 不可能            | 可能       | 可能              | 可能       |
| 可重复读 | 不可能            | 不可能     | 允许，但不在 PG中 | 可能       |
| 可序列化 | 不可能            | 不可能     | 不可能            | 不可能     |

在PostgreSQL中，你可以请求四种标准事务隔离级别中的任意一种，但是内部只实现了三种 不同的隔离级别，即 PostgreSQL 的读未提交模式的行为和读已提交相同。

# 锁

PostgreSQL提供了多种锁模式用于控制对表中数据的并发访问。 这些模式可以用于在MVCC 无法给出期望行为的情境中由应用控制的锁。 同样，大多数PostgreSQL命令会自动要求恰当的锁以保证被引用的表在命令的执行过程中 不会以一种不兼容的方式删除或修改(例 如，TRUNCATE无法安全地与同一表中上的其他操作并发地执行，因此它在表上获得一个排他 锁来强制这种行为)。

## 表级锁

1. ACCESS SHARE
   只与ACCESS EXCLUSIVE锁模式冲突。
   SELECT命令在被引用的表上获得一个这种模式的锁。通常，任何只读取表而不修改它的 查询都将获得这种锁模式。

2. ROW SHARE

   与EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。

   SELECT FOR UPDATE和SELECT FOR SHARE命令在目标表上取得一个这种模式的锁 (加上 在被引用但没有选择FOR UPDATE/FOR SHARE的任何其他表上的ACCESS SHARE锁)。

3. ROW EXCLUSIVE
   与SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。
   命令UPDATE、DELETE和INSERT在目标表上取得这种锁模式(加上在任何其他被引用表上 的ACCESS SHARE锁)。通常，这种锁模式将被任何修改表中数据的命令取得。

4. SHARE UPDATE EXCLUSIVE
   与SHARE UPDATE EXCLUSIVE、SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。这种模式保护一个表不受并发模式改变和VACUUM运行的影响。

   由VACUUM(不带FULL)、ANALYZE、 CREATE INDEX CONCURRENTLY、REINDEX CONCURRENTLY、 CREATE STATISTICS以及某些ALTER INDEX和 ALTER TABLE的变体获得(完 整的详细请参考ALTER INDEX 和ALTER TABLE)。

5. SHARE

   与ROW EXCLUSIVE、SHARE UPDATE EXCLUSIVE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。这种模式保护一个表不受并发 数据改变的影响。

   由CREATE INDEX(不带CONCURRENTLY)取得。

6. SHARE ROW EXCLUSIVE

   与ROW EXCLUSIVE、SHARE UPDATE EXCLUSIVE、SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。这种模式保护一个表不受并发 数据修改所影响，并且是自排他的，这样在一个时刻只能有一个会话持有它。

   由CREATE TRIGGER和某些形式的 ALTER TABLE所获得(见 ALTER TABLE)。

7. EXCLUSIVE

   与ROW SHARE、ROW EXCLUSIVE、SHARE UPDATE EXCLUSIVE、SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE锁模式冲突。这种模式只允许并发的ACCESS SHARE锁，即只有来自于表的读操作可以与一个持有该锁模式的事务并行处理。

   由REFRESH MATERIALIZED VIEW CONCURRENTLY获得。

8. ACCESS EXCLUSIVE

   与所有模式的锁冲突(ACCESS SHARE、ROW SHARE、ROW EXCLUSIVE、SHARE UPDATE EXCLUSIVE、SHARE、SHARE ROW EXCLUSIVE、EXCLUSIVE和ACCESS EXCLUSIVE)。这种模 式保证持有者是访问该表的唯一事务。

   由ALTER TABLE、DROP TABLE、TRUNCATE、REINDEX、CLUSTER、VACUUM FULL和REFRESH MATERIALIZED VIEW(不带CONCURRENTLY)命令获取。 很多形式的ALTER INDEX和ALTER TABLE也在这个层面上获得锁(见ALTER TABLE)。这也是未显式指定模式 的LOCK TABLE命令的默认锁模式。

表锁冲突矩阵

|                        | 已           | 申        | 请            | 到                     | 的    | 锁                  | 模        | 式               |
| ---------------------- | ------------ | --------- | ------------- | ---------------------- | ----- | ------------------- | --------- | ---------------- |
| **请求的锁模式**       | ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
| ACCESS SHARE           | Y            | Y         | Y             | Y                      | Y     | Y                   | Y         | N                |
| ROW SHARE              | Y            | Y         | Y             | Y                      | Y     | Y                   | N         | N                |
| ROW EXCLUSIVE          | Y            | Y         | Y             | Y                      | N     | N                   | N         | N                |
| SHARE UPDATE EXCLUSIVE | Y            | Y         | Y             | N                      | N     | N                   | N         | N                |
| SHARE                  | Y            | Y         | N             | N                      | Y     | N                   | N         | N                |
| SHARE ROW EXCLUSIVE    | Y            | Y         | N             | N                      | N     | N                   | N         | N                |
| EXCLUSIVE              | Y            | N         | N             | N                      | N     | N                   | N         | N                |
| ACCESS EXCLUSIVE       | N            | N         | N             | N                      | N     | N                   | N         | N                |



## 行级锁

1. FOR UPDATE

   FOR UPDATE会导致由SELECT语句检索到的行被锁定，就好像它们要被更新。这可以阻止 它们被其他事务锁定、修改或者删除，一直到当前事务结束。也就是说其他尝 试UPDATE、DELETE、SELECT FOR UPDATE、SELECT FOR NO KEY UPDATE、SELECT FOR SHARE或者SELECT FOR KEY SHARE这些行的事务将被阻塞，直到当前事务结束。反过 来，SELECT FOR UPDATE将等待已经在相同行上运行以上这些命令的并发事务，并且接着 锁定并且返回被更新的行(或者没有行，因为行可能已被删除)。不过，在一 个REPEATABLE READ或SERIALIZABLE事务中，如果一个要被锁定的行在事务开始后被更 改，将会抛出一个错误。

   任何在一行上的DELETE命令也会获得FOR UPDATE锁模式，在某些列上修改值的UPDATE也 会获得该锁模式。当前UPDATE情况中被考虑的列集合是那些具有能用于外键的唯一索引 的列(所以部分索引和表达式索引不被考虑)，但是这种要求未来有可能会改变。

2. FOR NO KEY UPDATE

   行为与FOR UPDATE类似，不过获得的锁较弱:这种锁将不会阻塞尝试在相同行上获得锁 的SELECT FOR KEY SHARE命令。任何不获取FOR UPDATE锁的UPDATE也会获得这种锁模式。

3. FOR SHARE

   行为与FOR NO KEY UPDATE类似，不过它在每个检索到的行上获得一个共享锁而不是排他 锁。一个共享锁会阻塞其他事务在这些行上执行UPDATE、DELETE、SELECT FOR UPDATE或 者SELECT FOR NO KEY UPDATE，但是它不会阻止它们执行SELECT FOR SHARE或者SELECT FOR KEY SHARE。

4. FOR KEY SHARE

   行为与FOR SHARE类似，不过锁较弱:SELECT FOR UPDATE会被阻塞，但是SELECT FOR NO KEY UPDATE不会被阻塞。一个键共享锁会阻塞其他事务执行修改键值的DELETE或者UPDATE，但不会阻塞其他UPDATE，也不会阻止SELECT FOR NO KEY UPDATE、SELECT FOR SHARE或者SELECT FOR KEY SHARE。

   

行级锁冲突矩阵

|                   | 当            | 前        | 锁                | 模式       |
| ----------------- | ------------- | --------- | ----------------- | ---------- |
| **请求的锁模式**  | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
| FOR KEY SHARE     | Y             | Y         | Y                 | N          |
| FOR SHARE         | Y             | Y         | N                 | N          |
| FOR NO KEY UPDATE | Y             | N         | N                 | N          |
| FOR UPDATE        | N             | N         | N                 | N          |



## 页级锁

除了表级别和行级别的锁以外，页面级别的共享/排他锁被用来控制对共享缓冲池中表页面 的读/写。 这些锁在行被抓取或者更新后马上被释放。