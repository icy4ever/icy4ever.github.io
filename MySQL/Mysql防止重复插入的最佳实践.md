## **Mysql防止重复插入的最佳实践**

假设我们有一张表：

注意这里的唯一索引很重要，因为这个是判断一条数据是否重复的重要标准。

```sql
CREATE TABLE `user_action_day`
(
    `id`           int(10)     NOT NULL AUTO_INCREMENT COMMENT '用户行为ID',
    `user_id`      varchar(24) NOT NULL COMMENT '用户ID',
    `action`       varchar(40)  DEFAULT '' COMMENT '用户行为',
    `stay_time_ms` float        DEFAULT '0' COMMENT '用户行为停留的毫秒数',
    `create_time`  datetime     DEFAULT NULL COMMENT '行为的创建时间',
    `day_of_Week`  int(2)       DEFAULT '1' COMMENT '行为发生在周几',
    `desc`         varchar(100) DEFAULT '' COMMENT '描述',
    PRIMARY KEY (`id`),
    UNIQUE KEY `user_id` (`user_id`, `action`, `create_time`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  ROW_FORMAT = DYNAMIC;主流插入防重复有以下几种手段：
```

### Replace

这种方式类似于 insert 只是把insert关键字替换为replace而已。

```sql
REPLACE INTO hera_test.user_action_day (user_id, action, stay_time_ms, create_time, day_of_Week, `desc`)
values ('1', '2', 1, '2020-05-22 10:03:21', 1, ''' ''');
```

遇到相同的数据，直接将新数据更新，同时如果你的主键是自增的，主键也会被更新。

### Ignore

这种方式在数据重复时，直接忽略，这种做法会忽略新数据，可以保证主键不自增。

```sql
insert ignore into hera_test.user_action_day (user_id, action, stay_time_ms, create_time, day_of_Week, `desc`)
values ('1', '2', 1, '2020-05-22 10:03:21', 1, ''' ''');
```

