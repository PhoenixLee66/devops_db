### 1. 锁定机制

> 数据库为了保证数据的一致性，而使各种共享资源在被并发访问变得有序所设计的一种规则，是由存储引擎各自所面对的特定场景二优化设计的

| 锁定机制 | 使用存储引擎                       | 特性                                         |
| ---- | ---------------------------- | ------------------------------------------ |
| 表级锁定 | MyISAM，MEMORY，CSV等一些非事务性存储引擎 | 开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低      |
| 行级锁定 | InnoDB                       | 开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高     |
| 页级锁定 | BerkeleyDB                   | 开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般 |

从锁的角度适用来说：

    表级锁更适合于以查询为主，只有少量按索引条件更新数据的应用，如Web应用；而行级锁则更适合于有大量按索引条件并发更新少量不同数据，同时又有并发查询的应用，如一些在线事务处理（OLTP）系统

### 2. Innodb 行级锁定

#### 2.1 锁定模式

###### 1. 当前读（locking read）

    当前读会在所有扫描到的索引记录上加锁，不管它后面的where条件到底有没有命中对应的行记录。

    对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁（X)；对于普通SELECT语句，InnoDB不会加任何锁，但可通过以下方式主动加锁

```sql
# 共享锁（S）
SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE

# 排他锁（X)
SELECT * FROM table_name WHERE ... FOR UPDATE
```

###### 2. 意向锁

    让行级锁定和表级锁定共存，显示“某个事务正在某一行上持有了锁或者准备去持有锁”，且自动添加的，无需用户干预。

    如果表中的数据很多，逐行检查锁标志的开销将很大，系统的性能将会受到影。如果存在意向锁，那么假如事务Ａ在更新一条记录之前，先加意向锁，再加Ｘ锁，事务B先检查该表上是否存在意向锁，存在的意向锁是否与自己准备加的锁冲突，如果有冲突，则等待直到事务Ａ释放，而无须逐条记录去检测

|           | 共享锁（S） | 排他锁（X） | 意向共享锁（IS） | 意向排他锁（IX） |
| --------- | ------ | ------ | --------- | --------- |
| 共享锁（S）    | 兼容     | 冲突     | 兼容        | 冲突        |
| 排他锁（X）    | ---    | 冲突     | 冲突        | 冲突        |
| 意向共享锁（IS） | ---    | ---    | 兼容        | 兼容        |
| 意向排他锁（IX） | ---    | ---    | ---       | 兼容        |

    如果一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁授予该事务；反之，如果两者不兼容，该事务就要等待锁释放。

    

#### 2.2 Innodb 行锁

##### 2.2.1 Record Lock

###### 1. 定义

    通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据（存在的数据），InnoDB才使用行级锁，即使该表上没有任何索引，那么innodb会在后台创建一个隐藏的聚集主键索引，除非没有使用任何索引，才使用表锁

使用实际例子：

* 不通过索引条件查询时，Innodb使用表锁

* innodb行锁是针对索引加的锁，即如果访问不同行的数据，也存在锁冲突问题（使用相同的索引键）

* 当表有多个索引时，不同的事务可以使用不同的索引锁定不同的行，另外行锁不区分索引类型

* 即便在条件中使用了索引字段，最终还是由explain执行计划的代价来决定的

##### 2.2.2 GAP Lock

###### 1. 定义

    在可重复读模式下，为了解决幻读问题，对数据（B+数据叶子节点）之间的间隙加锁

###### 2. 使用间隙锁目的

- 防止幻读，以满足相关隔离级别的要求

- 满足其恢复和复制的需要

##### 2.2.3 间隙锁（Next-Key）

###### 1. 定义

    行锁与间隙锁的组合 

#### 2.3 行锁优化建议

###### 2.3.1 扬长避短

1. 尽可能让所有的数据检索都通过索引来完成，从而避免InnoDB因为无法通过索引键加锁而升级为表级锁定

2. 合理设计索引，让InnoDB在索引键上面加锁的时候尽可能准确，尽可能的缩小锁定范围，避免造成不必要的锁定而影响其他Query的执行

3. 尽可能减少基于范围的数据检索过滤条件，避免因为间隙锁带来的负面影响而锁定了不该锁定的记录

4. 尽量控制事务的大小，减少锁定的资源量和锁定时间长度

5. 在业务环境允许的情况下，尽量使用较低级别的事务隔离，以减少MySQL因为实现事务隔离级别所带来的附加成本

###### 2.3.2 减少发生概率

1. 在业务环境允许的情况下，尽量使用较低级别的事务隔离，以减少MySQL因为实现事务隔离级别所带来的附加成本

2. 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率

3. 对于非常容易产生死锁的业务部分，可以尝试使用升级锁定颗粒度，通过表级锁定来减少死锁产生的概率

###### 2.3.3 查看系统状态变量

```sql
mysql> show status like 'InnoDB_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| InnoDB_row_lock_current_waits | 0     |
| InnoDB_row_lock_time          | 0     |
| InnoDB_row_lock_time_avg      | 0     |
| InnoDB_row_lock_time_max      | 0     |
| InnoDB_row_lock_waits         | 0     |
+-------------------------------+-------+
```

对各个状态量的说明如下：

　　InnoDB_row_lock_current_waits：当前正在等待锁定的数量
　　InnoDB_row_lock_time：从系统启动到现在锁定总时间长度
　　InnoDB_row_lock_time_avg：每次等待所花平均时间
　　InnoDB_row_lock_time_max：从系统启动到现在等待最常的一次所花的时间
　　InnoDB_row_lock_waits：系统启动后到现在总共等待的次数

　　对于这5个状态变量，比较重要的主要是InnoDB_row_lock_time_avg（等待平均时长），InnoDB_row_lock_waits（等待总次数）以及InnoDB_row_lock_time（等待总时长）这三项

#### 2.4 表锁使用场景

##### 2.4.1 使用场景

> 当然应用中这两种事务不能太多，否则，就应该考虑使用MyISAM表

1. 事务需要更新大部分或全部数据，表又比较大

2. 事务涉及多个表，比较复杂，很可能引起死锁，造成大量事务回滚

##### 2.4.2 表锁注意事项

1. 使用LOCK TABLES虽然可以给InnoDB加表级锁，但必须说明的是，表锁不是由InnoDB存储引擎层管理的，而是由其上一层──MySQL Server负责的，仅当autocommit=0（不自动提交，默认是自动提交的）、InnoDB_table_locks=1（默认设置）时，InnoDB层才能知道MySQL加的表锁，MySQL Server也才能感知InnoDB加的行锁，这种情况下，InnoDB才能自动识别涉及表级锁的死锁，否则，InnoDB将无法自动检测并处理这种死锁。

2. 在用 LOCK TABLES对InnoDB表加锁时要注意，要将AUTOCOMMIT设为0，否则MySQL不会给表加锁；事务结束前，不要用UNLOCK TABLES释放表锁，因为UNLOCK TABLES会隐含地提交事务；COMMIT或ROLLBACK并不能释放用LOCK TABLES加的表级锁，必须用UNLOCK TABLES释放表锁。

### 3. 事务

#### 3.1 事务属性

###### 1. 原子性

事务是一个原子操作单元。在当时原子是不可分割的最小元素，其对数据的修改，要么全部成功，要么全部都不成功

###### 2. 一致性

事务开始到结束的时间段内，数据都必须保持一致状态

###### 3. 隔离性

数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的"独立"环境执行

###### 4. 持久性

事务完成后，它对于数据的修改是永久性的，即使出现系统故障也能够保持

#### 3.2 事务常见问题

###### 1. 更新丢失

    当多个事务选择同一行操作，并且都是基于最初选定的值，由于每个事务都不知道其他事务的存在，就会发生更新覆盖的问题，好比github提交冲突

###### 2. 脏读

    事务A读取了事务B已经修改但尚未提交的数据。若事务B回滚数据，事务A的数据存在不一致性的问题

###### 3. 不可重复读

    事务A第一次读取最初数据，第二次读取事务B已经提交的修改或删除数据，导致两次读取数据不一致。

###### 4. 幻读

    事务A根据相同条件第二次查询到事务B提交的更新数据，两次数据结果集不一致

### 4. 死锁

#### 4.1 定义

> 当两个事务都需要获得对方持有的排他锁才能继续完成事务

#### 4.2 锁检测机制

> Innodb 通过判断产生死锁中的两个事务，哪一个更小，并回滚较小的事务

    事务的大小是通过计算插入、更新或者删除的数据量大小来判断事务大小，但如果跨存储 引擎，则无法检查到死锁。

InnoDB_lock_wait_timeout 参数含义：

* 解决死锁问题

* 解决高并发访问下，大量事务因获取锁等待而占用大量计算机资源，影响性能

#### 4.3 避免死锁的常用方法

1. 如果不同的程序并发存取多个表时，应尽量约定以相同的顺序访问

2. 在程序批量处理数据时，应尽量先对数据排序，以保证每个线程按固定的顺序处理记录

3. 更新记录时，应申请足够级别的锁，即排他锁

4. Repeatable-Read 隔离级别下，两个线程会同时加锁成功。如果在记录不存在的情况插入一条记录，会出现死锁，这时需要使用 Read-Committed 隔离级别。

5. Read-Committed 隔离级别下，第二个线程主键重复出错时，可直接执行插入操作，并捕获主键重复异常，或者执行Rollback释放获得的拍他锁

#### 4.4 死锁排查

##### 4.4.1 mysql 8.x

    <h6> 参考资料：[^1] </h6>

###### 1. 查询是否锁表

```sql
show OPEN TABLES where In_use > 0;
```

###### 2. 记录INNODB未提交事务信息

```sql
SELECT * FROM information_schema.INNODB_TRX;
```

重要参数：

| 表名                            | 字段                  | 注解        |
| ----------------------------- | ------------------- | --------- |
| information_schema.INNODB_TRX | trx_id              | 事务ID      |
| information_schema.INNODB_TRX | trx_mysql_thread_id | MySQL线程ID |

###### 3. 查看持有锁的线程以及线程ID

```sql
SELECT * FROM sys.`innodb_lock_waits`;
```

重要参数：

| 表名                                   | 字段                | 注解      |
| ------------------------------------ | ----------------- | ------- |
| information_schema.INNODB_LOCK_WAITS | blocking_trx_id   | 阻塞的事务ID |
| information_schema.INNODB_LOCK_WAITS | requesting_trx_id | 请求的事务ID |
| information_schema.INNODB_LOCKS      | lock_mode         | 锁模式     |
| information_schema.INNODB_LOCKS      | lock_index        | 锁定的索引类型 |
| information_schema.INNODB_LOCKS      | lock_space        | 表空间位置   |
| information_schema.INNODB_LOCKS      | lock_page         | 页位置     |
| information_schema.INNODB_LOCKS      | lock_rec          | 记录位置    |

###### 4. 查看当前被锁的语句

```sql
SELECT * FROM PERFORMANCE_SCHEMA.events_statements_history 
WHERE thread_id IN ( 
  SELECT b.`THREAD_ID` FROM sys.`innodb_lock_waits` AS a, 
          PERFORMANCE_SCHEMA.threads AS b WHERE 
      a.waiting_pid = b.`PROCESSLIST_ID` ) 
ORDER BY timer_start DESC;
```

###### 5. 查看持有锁的语句

```sql
SELECT * FROM PERFORMANCE_SCHEMA.events_statements_history 
WHERE thread_id IN ( 
  SELECT b.`THREAD_ID` FROM sys.`innodb_lock_waits` AS a, 
          PERFORMANCE_SCHEMA.threads AS b WHERE 
      a.`blocking_pid` = b.`PROCESSLIST_ID` ) 
ORDER BY timer_start DESC;
```

<h6> 备注：</h6>

| 表                 | DESCRIPTION                |
| ----------------- | -------------------------- |
| INNODB_TRX        | 记录 INNODB 未提交事务信息          |
| INNODB_LOCKS      | 记录 INNODB 锁信息，当出现锁等待时才有数据  |
| INNODB_LOCK_WAITS | 记录锁等待信息，关联 INNODB_LOCKS 查询 |

##### 4.4.2 mysql 5.x

###### 1. 方法一

```sql
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    TIMESTAMPDIFF( SECOND, r.trx_wait_started, CURRENT_TIMESTAMP ) AS wait_time,
    r.trx_query AS waiting_query,
    l.lock_table AS waiting_table_lock,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    SUBSTRING( p.HOST, 1, INSTR( p.HOST, ':' ) - 1 ) AS blocking_host,
    SUBSTRING( p.HOST, INSTR( p.HOST, ':' ) + 1 ) AS block_port,
IF
    ( p.command = "Sleep", p.time, 0 ) AS idle_in_trx,
    b.trx_query AS blcoking_query 
FROM
    information_schema.innodb_lock_waits AS w
    INNER JOIN information_schema.innodb_trx AS b ON b.trx_id = w.blocking_trx_id
    INNER JOIN information_schema.innodb_trx AS r ON r.trx_id = w.requesting_trx_id
    INNER JOIN information_schema.innodb_locks AS l ON w.requested_lock_id = l.lock_id
    LEFT JOIN information_schema.PROCESSLIST AS p ON p.id = b.trx_mysql_thread_id 
ORDER BY
    wait_time DESC;
```

###### 2. 方法二

```sql
select l.*
from (
  select 'blocker' as role, p.id, p.user, left(p.host, locate(':', p.host) - 1) as host, tx.trx_id
    , tx.trx_state, tx.trx_started, timestampdiff(second, tx.trx_started, now()) as duration, lo.lock_mode, lo.lock_type
    , lo.lock_table, lo.lock_index, tx.trx_query, lw.requested_lock_id as blockee_id, lw.requesting_trx_id as blockee_trx
  from information_schema.innodb_trx tx, information_schema.innodb_lock_waits lw, information_schema.innodb_locks lo, information_schema.processlist p
  where lw.blocking_trx_id = tx.trx_id
    and p.id = tx.trx_mysql_thread_id
    and lo.lock_id = lw.blocking_lock_id
  union
  select 'blockee' as role, p.id, p.user, left(p.host, locate(':', p.host) - 1) as host, tx.trx_id
    , tx.trx_state, tx.trx_started, timestampdiff(second, tx.trx_started, now()) as duration, lo.lock_mode, lo.lock_type
    , lo.lock_table, lo.lock_index, tx.trx_query, null, null
  from information_schema.innodb_trx tx, information_schema.innodb_lock_waits lw, information_schema.innodb_locks lo, information_schema.processlist p
  where lw.requesting_trx_id = tx.trx_id
    and p.id = tx.trx_mysql_thread_id
    and lo.lock_id = lw.requested_lock_id
) l
order by role desc, trx_state desc;
```

### 5. show engine innodb status 解读

#### 5.1 mysql 5.7

##### 5.1.1 BACKGROUND THREAD

> 后台Master线程

```bash
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 20436404 srv_active, 0 srv_shutdown, 520 srv_idle
srv_master_thread log flush and writes: 20436797
```

​	srv_active：每秒循环次数；srv_idle：每10秒循环次数。正常情况下，两者比例应该是 srv_active/srv_idle = 10:1，如果每秒循环次数少，每10秒次数多，证明当前负载很低，反之亦然

##### 5.1.2 SEMAPHORES

> 信号量信息，当前等待线程的列表及事件计数器，可以评估当前负载情况

```bash
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 2709304
OS WAIT ARRAY INFO: signal count 248725576
RW-shared spins 0, rounds 138334454, OS waits 1301178
RW-excl spins 0, rounds 253631585, OS waits 653630
RW-sx spins 11734, rounds 282329, OS waits 6640
Spin rounds per wait: 138334454.00 RW-shared, 253631585.00 RW-excl, 24.06 RW-sx
```

- reservation count：表示InnoDB产生了多少次OS WAIT，是线程放弃spin-wait进入挂起状态
- signal count：表示进入OS WAIT的线程被唤醒次数，即锁被释放后会收到信号（signal）唤醒，同时也会产生系统级别的上下文切换
- spins：空转次数，通过innodb_sync_spin_loops控制，超过则转到OS waits，其利用cpu的空闲时间来检查锁状态
- rounds：spin wait进行轮询检查mutexes的次数，即所谓循环的查询“锁被释放了吗？”
- OS waits：如果一直未得到锁释放的信息，则线程放弃spin wait进入挂起状态
- Spin rounds per wait：rounds / spins 

##### 5.1.3 LATEST DETECTED DEADLOCK

> 最近一次死锁信息，只有产生过死锁才会有

```bash
TRANSACTION 735968600, ACTIVE 0 sec fetching rows
```

- TRANSACTION 735968600：trx id 735968600
- ACTIVE 0 sec：事务是否正在运行
- fetching rows：显示事务状态，情况如下：
  - starting index read 表示根据索引读取数据
  - fetching rows 表示正在查找记录
  - updating or deleting 表示事务已经真正进入了update/delete的函数逻辑（row_update_for_mysql）
  - thread declared inside innoDB 说明事务已经进入innodb层，通常而言不在innodb层的事务大部分是会被回滚的

```bash
mysql tables in use 1, locked 1
```

- 当前事务使用一个表，且涉及锁的表只有1个

```bash
LOCK WAIT 31 lock struct(s), heap size 3520, 9 row lock(s), undo log entries 4
```

- LOCK WAIT 31 lock struct(s)：等待锁，且显示trx->trx_locks锁链表（锁结构包括表锁，记录锁以及auto_inc锁等）的长度，共计多少把锁
- heap size 3520：事务分配的锁堆内存大小（字节）
- 9 row lock(s)：多少记录被锁
- undo log entries 4：事务已经修改多少条记录

```bash
MySQL thread id 310837029, OS thread handle 140678934550272, query id 3095993884 172.21.1.186 zhibo updating
```

- MySQL thread id 310837029：mysql事务的线程ID信息
- OS thread handle 140678934550272：操作系统句柄信息
- query id 3095993884 172.21.1.186 zhibo updating：事务ID、连接来源、用户

```b
RECORD LOCKS space id 2245 page no 22 n bits 176 index PRIMARY of table `action_db`.`t_guild` trx id 735968600 lock_mode X locks rec but not gap waiting
```

- RECORD LOCKS：显示锁类型，这表示行锁（记录锁）
- space id 2245 page no 22 n bits 176：表空间ID为2245、页编号为2、页位置为176
- index PRIMARY of table action_db`.`t_guild trx id 735968600 lock_mode X locks rec but not gap waiting：锁发生在表action_db.t_guild的主键上，是一个X锁，但是不是gap lock。 waiting表示正在等待锁

```bash
Record lock, heap no 94 PHYSICAL RECORD: n_fields 20; compact format; info bits 0
 0: len 8; hex 8a4b4007ef359000; asc  K@  5  ;;
 1: len 6; hex 00002bddb1f8; asc   +   ;;
 2: len 7; hex 640000011b02e3; asc d      ;;
 3: len 8; hex 80000000000001d8; asc         ;;
 4: len 13; hex e7bbbfe983bd2de696b0e5a4a7; asc       -      ;;
 5: len 5; hex 3630303032; asc 60002;;
 6: len 30; hex 616664643062346164326563313732633538366532313530373730666266; asc afdd0b4ad2ec172c586e2150770fbf; (total 32 bytes);
 7: len 4; hex 800075b6; asc   u ;;
 8: len 4; hex 80000001; asc     ;;
 9: len 2; hex 8050; asc  P;;
 10: len 4; hex 7ff3ec3c; asc    <;;
 11: len 8; hex 8000000000000000; asc         ;;
 12: len 8; hex 8000000000000000; asc         ;;
 13: len 8; hex 8000000000054fe8; asc       O ;;
 14: len 3; hex 343137; asc 417;;
 15: SQL NULL;
 16: len 4; hex 80000001; asc     ;;
 17: len 2; hex 8000; asc   ;;
 18: len 4; hex 80000055; asc    U;;
 19: len 1; hex 80; asc  ;;
```

- heap no 94 PHYSICAL RECORD: n_fields 20：表示record lock的heap no 位置，共20个字段

##### 5.1.4 TRANSACTIONS

> 事务信息

```bash
------------
TRANSACTIONS
------------
Trx id counter 933672487
Purge done for trx's n:o < 933672486 undo n:o < 0 state: running but idle
History list length 25
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 422168997508736, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 422168997507824, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 422168997506912, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
```

- Trx id counter 933672487：当前事务ID
- undo n:o < 0 state: running but idle：nnodb清理进程正在使用的撤销日志编号，为0 0时说明清理进程处于空闲状态
- Purge done for trx's n:o < 933672486：innodb清除旧MVCC行时所用的事务ID，将这个值和当前事务ID进行比较，就可以知道有多少老版本的数据未被清除
- History list length 25：历史记录的长度，即位于innodb数据文件的撤销空间里的页面的数目，如果事务执行了更新并提交，这个数字就会增加，而当清理进程移除旧版本数据时，它就会减少，清理进程也会更新Purge done for.....这行中的数值
- TRANSACTION 422168997506912, not started：表示这个事务已经提交（日志落盘）并且没有再发起影响事务的语句，可能刚好空闲

​	purge 操作：当事务提交后，那些标记了删除的记录，以及undo log中的记录并不会马上清除，这些记录信息可以被其它事务所重用或是共享。只有当没有任何事务共享这些记录的时候，这些记录才会被清除（purge）。其操作流程如下：

1. 它标记此记录为删除（通过删除标记位）
2. 存储原始的记录到UNDO log
3. 更新记录列DB_TRX_ID和DB_ROLL_PTR（这些列是Innodb在原记录列上增加的）。DB_TRX_ID记录了最后操作记录的事务ID。DB_ROLL_PTR也叫回滚指针(rollback pointer)，指向UNDO log 记录

##### 5.1.5 FILE I/O

> IO Thread信息

| 字段                 | 备注                                          |
| :------------------- | :-------------------------------------------- |
| insert buffer thread‍ | 合并插入缓冲，insert buffer维护非唯一辅助索引 |
| log thread           | 负责异步刷新事物日志                          |
| read thread          | 预读，innodb_read_io_threads 默认4            |
| write thread         | 刷新脏页缓冲，innodb_write_io_threads 默认4‍   |
| purge thread         | 回收已经使用并分配的undo页，可设置多个        |
| purge cleaner Thread | 刷新脏页                                      |

##### 5.1.6 INSERT BUFFER AND ADAPTIVE HASH INDEX

> INSERT BUFFER和自适应HASH索引

| 字段                 | 备注                                                         |
| :------------------- | :----------------------------------------------------------- |
| Ibuf: size           | 已经合并页的数量                                             |
| free list len        | 空闲列表长度                                                 |
| seg size             | Insert Buffer大小                                            |
| merges               | 合并次数                                                     |
| merged operations    | Change Buffer中每个操作次数；insert代表Insert Buffer;delete mark代表Delete Buffer；delete代表Purge Buffer; |
| discarded operations | Change Buffer中无需合并的次数                                |
| hash searches/s      | 通过hash索引查询                                             |
| non-hash searches/s  | 不能通过hash索引查询                                         |

##### 5.1.7 BUFFER POOL AND MEMORY

> BUFFER POOL和内存

| 字段                         | 备注                               |
| :--------------------------- | ---------------------------------- |
| Total large memory allocated | 为innodb 分配的总内存数(byte)      |
| Dictionary memory allocated  | 为innodb数据字典分配的内存数(byte) |
| Buffer pool size             | innodb_buffer_pool的页数量         |
| Free buffers                 | lru列表中的空闲页数量              |
| Database pages‍               | lru列表中的非空闲页数量            |
| Old database pages           | old子列表的页数量                  |
| Modified db pages            | 脏页的数量                         |
| Pending reads                | 挂起读的数量                       |

##### 5.1.8 ROW OPERATIONS‍‍

> 行操作统计信息‍‍

​	Number of rows inserted、updated、deleted、read，表示多少行被插入，更新和删除，读取及每秒信息，可通过sql查看

```sql
show global status like 'Innodb_rows_%';
```



[^1]: [MYSQL锁分析和监控](http://www.dbarun.com/mysql/mysql-lock-analysis-and-monitor/)

<h6>参考资料</h6>

1. [day 59 MySQL之锁、事务、优化](http://py3study.com/Article/details/id/3975.html)
1. [一条命令解读InnoDB存储引擎—show engine innodb status](https://cloud.tencent.com/developer/article/1424670)
1. [[mysql之show engine innodb status解读](https://www.cnblogs.com/xiaoboluo768/p/5171425.html)
