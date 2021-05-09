### 删除

```sql
alter table table_name delete where id=1
```

### 更新

```sql
alter table table_name update id=2
```

#### SAMPLE

取样关键字

```sql
SELECT
    Title,
    count() * 10 AS PageViews
FROM hits_distributed
SAMPLE 0.1
WHERE
    CounterID = 34
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```

#### SAMPLE K OFFSET M

这里 `k` 和 `m` 是从0到1的数字。 示例如下所示。

**示例1**

```
SAMPLE 1/10
```

在此示例中，示例是所有数据的十分之一:

```
[++------------]
```

**示例2**

```
SAMPLE 1/10 OFFSET 1/2
```

这里，从数据的后半部分取出10％的样本。

```
[------++------]
```

#### Array Join

这个关键字是用来将字段中的元素拆分开来组上去。

```
┌─s───────────┬─arr─────┐
│ Hello       │ [1,2]   │
│ World       │ [3,4,5] │
│ Goodbye     │ []      │
└─────────────┴─────────┘
```

下面的例子使用 `ARRAY JOIN` 条款:

```sql
SELECT s, arr
FROM arrays_test
ARRAY JOIN arr;

┌─s─────┬─arr─┐
│ Hello │   1 │
│ Hello │   2 │
│ World │   3 │
│ World │   4 │
│ World │   5 │
└───────┴─────┘
```

这个相当于左边s字段和右边arr字段做了个inner join

别名：

```sql
SELECT s, arr, a
FROM arrays_test
ARRAY JOIN arr AS a;
```

既然有inner join 也就有 left join：

```sql
SELECT s, arr
FROM arrays_test
LEFT ARRAY JOIN arr;
┌─s───────────┬─arr─┐
│ Hello       │   1 │
│ Hello       │   2 │
│ World       │   3 │
│ World       │   4 │
│ World       │   5 │
│ Goodbye     │   0 │
└─────────────┴─────┘
```

使用别名，您可以执行 `ARRAY JOIN` 与外部阵列。 例如:

```sql
SELECT s, arr_external
FROM arrays_test
ARRAY JOIN [1, 2, 3] AS arr_external;
┌─s───────────┬─arr_external─┐
│ Hello       │            1 │
│ Hello       │            2 │
│ Hello       │            3 │
│ World       │            1 │
│ World       │            2 │
│ World       │            3 │
│ Goodbye     │            1 │
│ Goodbye     │            2 │
│ Goodbye     │            3 │
└─────────────┴──────────────┘
```

下面的例子使用 [arrayEnumerate](https://clickhouse.tech/docs/zh/sql-reference/functions/array-functions/#array_functions-arrayenumerate) 功能:

```
SELECT s, arr, a, num, arrayEnumerate(arr)
FROM arrays_test
ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num;
┌─s─────┬─arr─────┬─a─┬─num─┬─arrayEnumerate(arr)─┐
│ Hello │ [1,2]   │ 1 │   1 │ [1,2]               │
│ Hello │ [1,2]   │ 2 │   2 │ [1,2]               │
│ World │ [3,4,5] │ 3 │   1 │ [1,2,3]             │
│ World │ [3,4,5] │ 4 │   2 │ [1,2,3]             │
│ World │ [3,4,5] │ 5 │   3 │ [1,2,3]             │
└───────┴─────────┴───┴─────┴─────────────────────┘
```

#### 支持的类型 `JOIN`

- `INNER JOIN` （或 `JOIN`)
- `LEFT JOIN` （或 `LEFT OUTER JOIN`)
- `RIGHT JOIN` （或 `RIGHT OUTER JOIN`)
- `FULL JOIN` （或 `FULL OUTER JOIN`)
- `CROSS JOIN` （或 `,` )

查看标准 [SQL JOIN](https://en.wikipedia.org/wiki/Join_(SQL)) 描述。

#### 多联接

执行查询时，ClickHouse将多表联接重写为双表联接的序列。 例如，如果有四个连接表ClickHouse连接第一个和第二个，然后将结果与第三个表连接，并在最后一步，它连接第四个表。

如果查询包含 `WHERE` 子句，ClickHouse尝试通过中间联接从此子句推下过滤器。 如果无法将筛选器应用于每个中间联接，ClickHouse将在所有联接完成后应用筛选器。

我们建议 `JOIN ON` 或 `JOIN USING` 用于创建查询的语法。 例如:

```sql
SELECT * FROM t1 JOIN t2 ON t1.a = t2.a JOIN t3 ON t1.a = t3.a
```

**关键字**

**ALL** —如果右表具有多个匹配的行，则ClickHouse将根据匹配的行创建笛卡尔乘积（即集合 X * 集合 Y）。 这是SQL中的标准JOIN行为。
**ANY** —如果右表具有多个匹配的行，则仅连接找到的第一个行。 如果右表只有一个匹配行，则使用ANY和ALL关键字的查询结果是相同的。
**ASOF** —用于加入不完全匹配的序列。 ASOF JOIN用法如下所述。

