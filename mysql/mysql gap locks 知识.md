#### 1. 目的

    防止在`可重复读`的事务级别下，出现幻读问题

#### 2. 锁区间原则

##### 2.1 概述

    gap 锁的关键就是按照B+索引树排序规则，计算好叶子节点插入位置，锁住索引树的`叶子节点之间的间隙`，不让新的记录插入到间隙之中，即数据之间的间隔加锁。

    索引结构分为主索引树和辅助索引树，辅助索引树的叶子节点中包含了主键数据，主键数据影响着叶子节点的排序

![索引树](http://img-blog.csdnimg.cn/201902122353375.png)

##### 2.2 非唯一索引gap锁

> 边界都是开区间

    session1

```sql
select * from gap_table where num = 6 for update
```

    session2 

* 情况一，成功

```sql
INSERT INTO gap_table (letter, num) VALUES ('a', 3);
```

![](http://img-blog.csdnimg.cn/20190212235441833.png)

* 情况二，失败

```sql
INSERT INTO gap_table (letter, num) VALUES ('e', 3);
```

![情况2](http://img-blog.csdnimg.cn/20190212235453304.png)

- 情况三，失败

```sql
INSERT INTO gap_table (letter, num) VALUES ('h', 6);
```

![情况3](http://img-blog.csdnimg.cn/20190212235504455.png)

- 情况四，失败

```sql
INSERT INTO gap_table (letter, num) VALUES ('h', 7);
```

![情况4](http://img-blog.csdnimg.cn/20190212235514403.png)

- 情况五，成功

```sql
INSERT INTO gap_table (letter, num) VALUES ('h', 9);
```

![情况5](http://img-blog.csdnimg.cn/20190212235526380.png)

##### 2.3 唯一索引或者非唯一索引范围检索gap锁

> 唯一索引gap锁区间是左开右闭

| session1                                                   | session2                             | session2 结果 | 分析                 |
| ---------------------------------------------------------- | ------------------------------------ | ----------- | ------------------ |
| select * from gap_tbz where id > 5 for update;             | insert into gap_tbz values(6,'cc');  | 失败          | gap锁的区间是id在（5，+无限） |
| select * from gap_tbz where id > 5 and id < 11 for update; | insert into gap_tbz values(11,'cc'); | 失败          | gap锁的区间是id在（5，11]  |
|                                                            | insert into gap_tbz values(5,'cc');  | 主键重复        | gap锁的区间是id在（5，11]  |

#### 3. gap  加锁规则

| 索引规则  | 查询条件 | 锁规则                                          |
| ----- | ---- | -------------------------------------------- |
| 唯一索引  | 等值查询 | Next-Key Locks就退化为记录锁，不会加 gap 锁              |
|       | 范围查询 | 会锁住where条件中相应的范围，范围中的记录以及间隙，换言之就是加上记录锁和gap 锁 |
| 非唯一索引 | 等值查询 | Next-Key Locks会对间隙加gap锁，以及对应检索到的记录加记录锁       |
|       | 范围查询 | 会锁住where条件中相应的范围，范围中的记录以及间隙，换言之就是加上记录锁和gap 锁 |
| 非索引检索 |      | 全表间隙gap lock，全表记录record lock                 |

<h6>参考资料：</h6>

1. [深入了解mysql--gap locks,Next-Key Locks - aizhen - 博客园](https://www.cnblogs.com/chongaizhen/p/11168442.html)
