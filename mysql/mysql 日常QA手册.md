##### 1. MDL 锁导致无法操作数据库

```sql
select *
from information_schema.innodb_trx i,
  (select id, time from information_schema.processlist
   where time =
       (select max(time)
        from information_schema.processlist
        where state = 'Waiting for table metadata lock'
          and substring(info, 1, 5) in ('alter', 'optim', 'repai',
                                        'lock', 'drop', 'creat'))) p
where timestampdiff(second, i.trx_started, now()) > p.time
  and i.trx_mysql_thread_id not in (connection_id(), p.id);
```

##### 2. mysq 锁查询

###### 2.1 mysql 8.x

    <h6>参考资料：[^1]</h6>

1. 查询是否锁表
   
   ```sql
   show OPEN TABLES where In_use > 0;
   ```

2. 记录INNODB未提交事务信息
   
   ```sql
   SELECT * FROM information_schema.INNODB_TRX;
   ```
   
   重要参数：
   
   | 表名                            | 字段                  | 注解        |
   | ----------------------------- | ------------------- | --------- |
   | information_schema.INNODB_TRX | trx_id              | 事务ID      |
   | information_schema.INNODB_TRX | trx_mysql_thread_id | MySQL线程ID |

3. 查看持有锁的线程以及线程ID
   
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

4. 查看当前被锁的语句
   
   ```sql
   SELECT * FROM PERFORMANCE_SCHEMA.events_statements_history 
   WHERE thread_id IN ( 
       SELECT b.`THREAD_ID` FROM sys.`innodb_lock_waits` AS a, 
               PERFORMANCE_SCHEMA.threads AS b WHERE 
           a.waiting_pid = b.`PROCESSLIST_ID` ) 
   ORDER BY timer_start DESC;
   ```

5. 查看持有锁的语句
   
   ```sql
   SELECT * FROM PERFORMANCE_SCHEMA.events_statements_history 
   WHERE thread_id IN ( 
       SELECT b.`THREAD_ID` FROM sys.`innodb_lock_waits` AS a, 
               PERFORMANCE_SCHEMA.threads AS b WHERE 
           a.`blocking_pid` = b.`PROCESSLIST_ID` ) 
   ORDER BY timer_start DESC;
   ```

<h6>备注：</h6>

| 表                 | DESCRIPTION                |
|:----------------- |:-------------------------- |
| INNODB_TRX        | 记录 INNODB 未提交事务信息          |
| INNODB_LOCKS      | 记录 INNODB 锁信息，当出现锁等待时才有数据  |
| INNODB_LOCK_WAITS | 记录锁等待信息，关联 INNODB_LOCKS 查询 |

###### 2.2 mysql 5.x

> 方法一

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

> 方法二

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

##### 3. mysql 密码修改

###### 3.1 docker

    <h6>参考资料：[^2]</h6>

1. 进入容器
   
   ```bash
   docker exec -it mysql /bin/bash
   ```

2. 修改docker.cnf配置文件
   
   > 在 /etc/mysql/conf.d/docker.cnf 文件中追加 skip-grant-tables

3. 重启容器
   
   ```bash
   docker restart mysql
   ```

4. 登录mysql，修改密码
   
   ```bash
   mysql --socket /var/lib/mysql/mysql.socket -A
   
   # 修改密码
   mysql> update mysql.user set authentication_string=password('a123456') where user='root'; 
   
   # 刷新权限
   mysql> flush privileges;
   ```

5. docker.cnf配置还原，并重启服务

###### 3.2 宿主机

1. 调整启动命令

2. 登录 mysql，修改密码
   
   ```bash
   SET PASSWORD FOR 'root'@'localhost' = PASSWORD('你的新密码');
   
   FLUSH PRIVILEGES; 
   ```

3. 重启 mysql 服务

##### 4. mac下修改mysql端口

1. 以 xml 格式打开 mysqld.plist 文件
   
   ```bash
   plutil -convert xml1 /Library/LaunchDaemons/com.oracle.oss.mysql.mysqld.plist
   ```

2. 在ProgramArguments的array中添加或修改 <string>--port=3306</string>即可

[^1]: [MYSQL锁分析和监控](http://www.dbarun.com/mysql/mysql-lock-analysis-and-monitor/)

[^2]: [MYSQL忘记密码](https://blog.csdn.net/weixin_39816332/article/details/103746846)
