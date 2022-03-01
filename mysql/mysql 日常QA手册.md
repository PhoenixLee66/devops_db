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

##### 2. mysql 密码修改

###### 2.1 docker

    <h6>参考资料：[^1]</h6>

1. 进入容器
   
   ```bash
   docker exec -it mysql /bin/bash
   ```

2. 修改 docker.cnf 配置文件
   
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

5. docker.cnf 配置还原，并重启服务

###### 2.2 宿主机

1. 调整启动命令

2. 登录 mysql，修改密码
   
   ```sql
   SET PASSWORD FOR 'root'@'localhost' = PASSWORD('你的新密码');
   
   FLUSH PRIVILEGES; 
   ```

3. 重启 mysql 服务

##### 3. mac下修改mysql端口

1. 以 xml 格式打开 mysqld.plist 文件
   
   ```bash
   plutil -convert xml1 /Library/LaunchDaemons/com.oracle.oss.mysql.mysqld.plist
   ```

2. 在ProgramArguments的array中添加或修改 <string>--port=3306</string>即可

[^1]: [MYSQL忘记密码](https://blog.csdn.net/weixin_39816332/article/details/103746846)
