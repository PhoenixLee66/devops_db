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

# 

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

[^1]: [MYSQL锁分析和监控](http://www.dbarun.com/mysql/mysql-lock-analysis-and-monitor/)

<h6>参考资料</h6>

1. [day 59 MySQL之锁、事务、优化](http://py3study.com/Article/details/id/3975.html)
