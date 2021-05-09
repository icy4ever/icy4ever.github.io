## MySQL索引失效原因总结

#### 1.范围查询

​	< 、>、<>、 != 、is null 、 is not null、or查询会使索引失效	

#### 2.使用联合索引却不符合最左前缀匹配

​	对于索引 Test (a,b,c) , (a,b,c)和(a,b)、(a)都是可以匹配的，没有问题。但是(b),(b,c)是用不着的。如果在a字段中使用范围查询也是会使索引失效的。	

​	PS:创建和销毁索引语句如下：

```mysql
alter table add index index_name(column_name_1,column_name_2...)
alter table drop index index_name;
```

#### 3.like查询在头部使用%做模糊匹配

#### 4.字符串不加单引号