# 日志文件

## binlog（server实现）

​	二进制日志（binary log）**记录了**对mysql数据库**执行更改的所有操作**，但是不包括select和show这类操作。如果操作本身并没有导致数据库发生变化，该操作也能被写入二进制日志。

### 作用

1. **MySQL主从复制**：MySQL Replication在Master端开启binlog，Master把它的二进制日志传递给slaves来达到master-slave数据一致的目的
2. **数据恢复**：通过使用 mysqlbinlog工具来使恢复数据

### 格式

​	`binlog_format`影响了记录二进制日志的格式，可选值有statement、row和mixed。

记录在二进制日志中的事件的格式取决于二进制记录格式。支持三种格式类型：

- STATEMENT：基于SQL语句的复制（statement-based replication, SBR）
- ROW：基于行的复制（row-based replication, RBR）
- MIXED：混合模式复制（mixed-based replication, MBR）



### 查看binlog文件

#### mysql实例中查看

mysql实例中与binlog相关的语句：

1. 是否启用了日志

   ```mysql
   mysql>show variables like 'log_bin';
   ```

2. 获取binlog文件列表；

   ```mysql
   mysql>show binary logs;
   ```

3. 查看当前正在写入的binlog文件

   ```mysql
   mysql>show master status\G
   ```

4. 查看指定binlog文件的内容

   ```mysql
   mysql>show binlog events in 'mysql-bin.000006';
   ```



#### 用mysqlbinlog工具查看

1. 不要查看当前正在写入的binlog文件
2. 不要加--force参数强制访问
3. 如果binlog格式是行模式的,请加 -vv参数

基于开始/结束时间

```shell
mysqlbinlog --start-datetime='2013-09-10 00:00:00' --stop-datetime='2013-09-10 01:01:01' -d 库名 二进制文件
```

基于pos值(写入位置)

```
mysqlbinlog --start-postion=107 --stop-position=1000 -d 库名 二进制文件
```



## redolog（innodb实现）

​	Write Ahead Log策略：即当事务提交时，先写重做日志，再修改页。

​	redolog就是基于WAL策略出现的

## Mysql异常重启如何保证数据完整性的

​	日志文件提交的两阶段图：

![1679040064306](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679040064306.png)

### 崩溃恢复规则

#### 时刻A

​	写入redo log 处于prepare阶段之后、写binlog之前中时刻A的地方，也就是写入redo log 处于prepare阶段之后、写binlog之前，发生了崩溃（crash），由于此时binlog还没写，redo log也还没提交，所以崩溃恢复的时候，这个**事务会回滚**。此时binlog还没写，所以也不会传到备库。
​	

#### 时刻B

​	在时刻B对应于binlog写完，redo log还没commit前发生crash，崩溃恢复逻辑：

1. 如果redo log里面的事务是完整的，也就是已经有了commit标识，则直接提交；
2. 如果redo log里面的事务只有完整的prepare，则判断对应的事务binlog是否存在并完整：
   a. 如果是，则提交事务；
   b. 否则，回滚事务。

时刻B发生crash对应的就是2(a)的情况，崩溃恢复过程中事务会被提交。



### redo log 和 binlog是怎么关联起来的?

​	它们有一个**共同的数据字段，叫XID**。崩溃恢复的时候，会按顺序扫描redo log：如果碰到既有prepare、又有commit的redo log，就直接提交；如果碰到只有parepare、而没有commit的redo log，就拿着XID去binlog找对应的事务



### 处于prepare阶段的redo log加上完整binlog，重启就能恢复，MySQL为什么要这么设计?

​	主要是为了数据与备份的一致性有关。在时刻B，也就是binlog写完以后MySQL发生崩溃，这时候binlog已经写入了，之后就会被从库（或者用这个binlog恢复出来的库）使用。
​	所以，在主库上也要提交这个事务。采用这个策略，主库和备库的数据就保证了一致性。



## mysql如何写入日志

### binlog 的写入机制

​	事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。**一个事务的 binlog 是不能被拆开的**，因此不论这个事务多大，也**要确保一次性写入**。这就涉及到了 binlog cache 的保存问题。

![1679059617067](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679059617067.png)

​	可以看到，每个线程有自己 binlog cache，但是共用同一份 binlog 文件。

- 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。
- 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。



​	对支持事务的引擎如**InnoDB而言，必须要提交了事务才会记录binlog**。binlog 什么时候刷新到磁盘跟参数 sync_binlog 相关。write 和 fsync 的时机，是由参数 `sync_binlog` 控制的：

1. `sync_binlog=0` 的时候，表示每次提交事务都只 write，不 fsync；
2. `sync_binlog=1` 的时候，表示每次提交事务都会执行 fsync；
3. `sync_binlog=N(N>1)` 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

### redo log的写入机制

redo log存在这三种状态分别是：

1. 存在 redo log buffer 中，**物理上是在 MySQL 进程内存中**，就是图中的红色部分；
2. 写到磁盘 (write)，但是没有持久化（fsync)，**物理上是在文件系统的 page cache 里面**，也就是图中的黄色部分；【**mysql异常宕机不影响**】
3. 持久化到磁盘，对应的是 hard disk，也就是图中的绿色部分。

![1679059805078](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679059805078.png)

​	日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了。

为了控制 redo log 的写入策略，InnoDB 提供了` innodb_flush_log_at_trx_commit` 参数，它有三种可能取值：

1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；（同步）
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache（异步）



​	InnoDB 有一个后台线程（主线程，主要负责数据的刷入），每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。

​	注意，事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，**一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的**。



​	如果把` innodb_flush_log_at_trx_commit`设置成 1，那么 **redo log 在 prepare 阶段就要持久化一次**，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的。每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，**InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了**。



### 异常场景下的一致性分析

#### mysql异常可能发生的位置

1. redo log在prepare阶段持久化到磁盘（可能失败）
2. 紧接着binlog持久化（可能失败） 
3. 最后redolog commit（可能失败） 



#### 异常一致性分析（重点）

情况一：在1处mysql异常重启，redo log没有fsync，内存丢失，直接回滚，不影响数据一致性；

情况二：redolog fsync成功，但是binlog写入错误，此时mysql异常重启，现在有redo log的磁盘数据没有binlog的数据，此时检测redolog处于prepare阶段，但是没有binlog，回滚（虽然刚刚redolog fsync了，但是不影响数据一致性，因为redolog的操作并没有写入mysql，也永远不会写入mysql）； 

情况三：binlog完整但未commit，此时检测redolog处于prepare阶段，且**binlog完整**但未提交，默认添加commit标记，进而提交，**写入mysql**，满足数据一致性； 

情况四：binlog完整且提交，写入mysql，满足一致性；



#### 总结

​	binlog完整落盘了，数据一定会写入mysql中。即使后续发生了异常，也能根据redo log恢复出来。



### 组提交（降低IOPS）

​	通常我们说 MySQL 的**“双 1”配置（推荐配置），指的就是 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1**。也就是说，一个事务完整提交前，需要等待**两次刷盘**，一次是 redo log（prepare 阶段），一次是 binlog。

​	组提交的概念就是在fsync持久化到磁盘的时候，可以根据日志逻辑序列号（log sequence number，LSN）把其他事务的日志一起落盘。

​	在并发更新场景下，第一个事务写完redo log buffer以后，接下来这个fsync越晚调用，组员可能

越多，节约IOPS的效果就越好。



​	如果你想提升 binlog 组提交的效果，可以通过设置 `binlog_group_commit_sync_delay` 和`binlog_group_commit_sync_no_delay_count` 来实现。

1. `binlog_group_commit_sync_delay` 参数，表示延迟多少微秒后才调用 fsync;
2. `binlog_group_commit_sync_no_delay_count` 参数，表示累积多少次以后才调用 fsync。这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。



**如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？**

针对这个问题，可以考虑以下三种方法：

1. 设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。（还没执行fsync持久化，就断电了）
3. 将 innodb_flush_log_at_trx_commit 设置为 2（写入os的是page cace）。这样做的风险是，由于没有落盘，**主机掉电**的时候会丢数据。



## 问题

Q：为什么 binlog cache 是每个线程自己维护的，而 redo log buffer 是全局共用的？

A：MySQL 这么设计的主要原因是，binlog 是不能“被打断的”。一个事务的 binlog 必须连续写，因此要整个事务完成后，再一起写到文件里。

而 redo log 并没有这个要求，中间有生成的日志可以写到 redo log buffer 中。redo log buffer 中的内容还能“搭便车”，其他事务提交的时候可以被一起写到磁盘中。
