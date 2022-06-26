## 说一下最近遇到的2个MySQL相关的问题

# 数据库死锁

**现象：**
1. 有两个接口：
  - 一个更新操作根据`name`，SQL：`update set xxx table where name=xxx`。
  - 一个更新操作根据`id`，SQL：`update set xxx table where id=xxx`。
  - 都开启了事务。
2. 在测试的时候是很好的。
3. 但是到了线上，经常反馈说慢。

**原因：**
1. 开启了事务
2. `where` 条件中的字段没有加索引。
3. 导致锁表了，如果加了索引就会锁住那一行。

**解决：**
1. 可以给第一个SQL中的`where`条件中加一个id？
2. 也可以直接给name建立一个索引
3. 当然也可以取消事务。


# SQL查询很慢
这是一个聊天记录查询的SQL，本来在写的时候以及针对这个场景做了一定的优化了。也确实很快，上线了几个月都没什么问题。但是在某一个放假回来之后速度从 **100~200ms** 直接到了 **17s** 

## 优化前的SQL
```sql
     select xxx
        from  a
        where (
                (a.from_id = #{req.fId} and a.to_list = #{req.toId})
                or
                (a.from_id = #{req.toId} and a.to_list = #{req.fId})
            )
        <!-- 翻页 -->
        <if test="req.nextId != null">
            <if test="req.turnPageType == 1">
                and a.id > #{req.nextId}
            </if>
            <if test="req.turnPageType == 2">
                and #{req.nextId} > a.id
            </if>
        </if>
        <if test="req.nextId != null">
            <if test="req.turnPageType == 1">
                order by msg_time, id
            </if>
            <if test="req.turnPageType == 2">
                order by msg_time desc,id desc
            </if>
        </if>
        <if test="req.nextId == null">
            order by msg_time, id
        </if>
        <if test="req.id != null">
            and a.id >= #{req.id}
        </if>
        limit 100;
```

## 优化后的SQL
```sql
SELECT  各种字段
FROM `a`
RIGHT JOIN
(
SELECT  id
FROM `a`
WHERE xxx
LIMIT 100;
) t ON t.id = a.id
```

## 主要原理
先说一下什么是**回表：**
> 可以理解为普通索引的查询，先定位主键值，再定位行记录，它的性能较扫一遍索引树更低。

这个SQL主要是利用了子查询优化
1. 前面我们没有优化的SQL其实是命中了索引的，我对`msg_time`以及`from_id `都建立了索引，然后我们取出的数据，还需要其他的字段，所以就导致多次的回表取数据。
2. 只查询中只查主键
3. 查询出来的记录就包含了所有的列的值，就不用进行回表，去取出其他数据了。
4. 在用`RIGHT JOIN`进行关联，得到我们想要的数据。