# join使用方式

## join的基本原理

### 名词

​	驱动表：按照该表的内容为基准连接被驱动表

​	被驱动表：该表被连接到驱动表中



### 怎么选择驱动表？

建表，t1和t2结构相同

```mysql
CREATE TABLE `t2` (
`id` int(11) NOT NULL,
`a` int(11) DEFAULT NULL,
`b` int(11) DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `a` (`a`)
) ENGINE=InnoDB;
create table t1 like t2;	
```

#### IndexNested-Loop Join

```mysql
select * from t1 straight_join t2 on (t1.a=t2.a);
```

执行流程是这样的：

1. 从表t1中读入一行数据 R；
2. 从数据行R中，取出a字段到表t2里去查找；
3. 取出表t2中满足条件的行，跟R组成一行，作为结果集的一部分；
4. 重复执行步骤1到3，直到表t1的末尾循环结束。



​	这个过程是先遍历表t1，然后根据从表t1中取出的每行数据中的a值，去表t2中查找满足条件的记录。在形式上，这个过程和写程序时的嵌套查询类似，并且**可以用上被驱动表的索引**，所以我们称之为“IndexNested-Loop Join”，简称NLJ。

![1679389419412](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679389419412.png)

​	**在这个join语句执行过程中，驱动表是走全表扫描，而被驱动表是走树搜索。**



#### Block Nested-Loop Join

​	再看看被驱动表用不上索引的情况

```mysql
select * from t1 straight_join t2 on (t1.a=t2.b)
```

被驱动表上没有可用的索引，算法的流程：
1. 把表t1的数据读入线程内存join_buffer中，由于我们这个语句中写的是select *，因此是把整个表t1放入了内存；
2. 扫描表t2，把表t2中的每一行取出来，跟join_buffer中的数据做对比，满足join条件的，作为结果集的一部分返回。

![1679389465146](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679389465146.png)

​	join_buffer的大小是由参数join_buffer_size设定的，默认值是256k。如果放不下表t1的所有数
据话，策略很简单，就是**分段放**。

执行过程就变成了：

1. 扫描表t1，顺序读取数据行放入join_buffer中，放完第88行join_buffer满了，继续第2步；
2. 扫描表t2，把t2中的每一行取出来，跟join_buffer中的数据做对比，满足join条件的，作为结
果集的一部分返回；
3. 清空join_buffer；
4. 继续扫描表t1，顺序读取最后的12行数据放入join_buffer中，继续执行第2步。

![1679390822701](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1679390822701.png)

**join_buffer_size越大，一次可以放入的行越多，分成的段数也就越少，对被驱动表的全表扫描次数就越少。**



#### 总结

1. 如果可以使用被驱动表的索引，join语句还是有其优势的；
2. 不能使用被驱动表的索引，只能使用Block Nested-Loop Join算法，这样的语句就尽量不要
使用；
3. 在使用join的时候，应该让小表做驱动表。
4. 在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是“小表”。**(过滤后的数据量判断是否为小表)**

