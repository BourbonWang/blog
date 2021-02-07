# MySQL 事务

当处理操作量大、复杂度高的数据时，需要很多条SQL语句一起执行，当某条操作出现错误时，已经完成的操作又可以撤销。这些SQL语句就构成了事务。MySQL中，只有Innodb引擎才支持事务。

## 特性

- **原子性（Atomicity）**：一个事务中的所有操作，要么全部完成，要么全部不完成，不会在中间某个环节结束。当执行过程中发生错误时，数据库会回滚(Rollback)到事务开始之前的状态。
- **一致性（Consistency）**：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。比如A向B转账，不可能A扣了钱，B却没收到。
- **隔离性（Isolation）：**数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离详见下文。
- **持久性（Durability）：**事务完成后，对数据的修改就是永久保存的，不能回滚。

## 事务控制

在 MySQL 命令行的默认设置下，事务都是自动提交的，即执行 SQL 语句后就会马上执行 COMMIT 操作。因此要显式地开启一个事务务须使用命令 BEGIN 或 START TRANSACTION，或者执行命令 SET AUTOCOMMIT=0，用来禁止使用当前会话的自动提交。

- BEGIN 或 START TRANSACTION：显式地开启一个事务；
- COMMIT：提交事务，事务对数据的修改会永久保存；
- ROLLBACK：回滚，会结束用户的事务，并撤销正在进行的所有未提交的修改；
- SAVEPOINT identifier：在事务中创建一个保存点 identifier，一个事务中可以有多个 SAVEPOINT；
- RELEASE SAVEPOINT identifier：删除一个事务的保存点 identifier，当没有指定的保存点时，执行该语句会抛出一个异常；
- ROLLBACK TO identifier：把事务回滚到标记点 identifier；
- SET TRANSACTION：设置事务的隔离级别。

## 并发问题

一个事务中有很多条SQL语句，当事务并发执行时，由于这些语句被交叉执行，很可能会出现以下问题。

### 脏读

读取了另一事务未提交的数据。比如：

事务B对数据进行了更新操作，但还未提交。另一客户端A此时查询到了B更新后的数据。但如果B进行回滚，B的更新操作就会被撤销，数据会恢复到更新之前，那么，A读到的数据就是脏数据。

### 不可重复读

在一个事务过程中，读取到了另一事务已提交的数据。比如：

事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时（A事务未提交），结果不一致。

### 幻读

比如：目前工资为5000的员工有10人，事务A读取所有工资为5000的人数为10人。此时，事务B插入一条工资也为5000的记录，并提交。这时，事务A再次读取工资为5000的员工，记录为11人。此时产生了幻读。

与不可重复读相比，不可重复读的场景往往是更新，需要行级锁；幻读更多是增加和删除，需要表级锁。

## 事务隔离

MySQL有4种隔离级别：读未提交、读已提交、可重复读、串行化，用以解决上面的并发问题。隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。MySQL默认的隔离级别是可重复读。

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 读已提交（read-committed）   | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |

### 读未提交 

所有事务都可以看到没有提交事务的数据。3种并发问题都可能会发生。

### 读已提交

也叫提交读。事务成功提交后才可以被查询到，所以解决了脏读的问题。但是只对事务的过程进行了隔离，没有对数据进行隔离，依然存在不可重复读和幻读的问题。

### 可重复读

MySQL默认的隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行，解决了不可重复读的问题。但对于插入删除操作，仍然会出现幻读。

### 串行化

要想解决幻读的问题，只有把事务排序使之不可能冲突，把事务的并发执行变成串行，这种隔离级别可能导致大量的超时现象和锁竞争，并发性能非常差。
