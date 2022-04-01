# MySQL随笔 - 锁

​	InnoDB普遍采用了行级锁，它的行级锁主要分为两种：**共享锁**和**排他锁**。持有共享锁的事务可以对记录做读取操作，而持有排他锁的事务则可以对记录做更新和删除操作。下表展示了这两个锁的排他性：

| 事务1\事务2 | 共享锁 | 排他锁 |
| ----------- | ------ | ------ |
| **共享锁**  | Y      | F      |
| **排他锁**  | F      | F      |

​	上表可以看出，只有在两个事务都需要共享锁时，两事务才能同时获得对应操作。

​	当我们尝试获取二级索引的锁时，由于二级索引不存完整数据，所以它需要去聚集索引那获取数据，此时也会导致聚集索引被上锁。

​	典型的排他锁有Update、Delete和 select...for update。

## 死锁

​	InnoDB在两事务间存在循环依赖时会产生死锁报错：

```
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

​	举个🌰：

​	事务1执行了如下操作：

```mysql
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;
```

​	此时事务1明显持有i=1这条记录的**共享锁**。 	

​	事务2执行了如下操作：

```mysql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
```

​	在此时事务2尝试删除i=1的记录，这样它将尝试获得i=1这条记录的排他锁。但是由于事务1持有共享锁，所以这里不会获得排他锁，它将放入获得该记录锁的队列。

```mysql
mysql> DELETE FROM t WHERE i = 1;
```

​	在此时事务1将尝试删除i=1的记录，由于队列中已经存在了事务2的请求，所以这将导致**死锁**。

### 如何避免死锁

​	尽量减少长事务，这样能大大减少事务冲突的可能性。

## 总结

​	其实还有很多其他的锁，比如间隙锁等，但是由于时间有限，简要的拿了主要的锁做介绍。锁的粒度决定了InnoDB的查询、修改性能。尽量减少长事务可以减少锁冲突的可能性，并提高性能。

