## [explain介绍](https://juejin.cn/post/6844903592776695821)

​	MySQL 提供了一个 EXPLAIN 命令, 它可以对 `SELECT` 语句进行分析, 并输出 `SELECT` 执行的详细信息, 以供开发人员针对性优化.
EXPLAIN 命令用法十分简单, 在 SELECT 语句前加上 Explain 就可以了, 例如:

```sql
EXPLAIN SELECT * from user_info WHERE  id < 300;
```



## 慢查询

1. 查看慢查询设置是否打开:

   `show variables like '%slow%';`

2. 查看是否记录未使用索引的语句:

   `show variables like "log_queries_not_using_indexes";`

为了把所有语句记录到slowlog里，先执行了 `set long_query_time=0`，将慢查询日志的时间阈值设置为0。

如果慢查询日志中记录内容很多，可以使用**mysqldumpslow工具**（MySQL客户端安装自带）来对慢查询日志进行分类汇总。



`innodb_io_capacity`：用于告诉InnoDB你的磁盘能力，影响数据库刷脏页的速度。建议设置为磁盘IOPS。磁盘的IOPS可以通过fio这个工具来测试



而要看事务具体状态的话，你可以查`information_schema库的innodb_trx`表。

`select * from information_schema.innodb.trx;`

`select * from information_schema.innodn_trx\G;`



通过查询sys.schema_table_lock_waits这张表，我们就可以直接找出造成阻塞的process id，把
这个连接用kill 命令断开即可。

`select * from sys.innodb_lock_waits`