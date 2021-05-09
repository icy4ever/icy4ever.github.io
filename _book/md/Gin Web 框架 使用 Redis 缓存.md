## Gin Web 框架 使用 Redis 缓存 

首先安装redis

```
wget http://download.redis.io/releases/redis-5.0.7.tar.gz
tar -C /redis-5.0.7.tar.gz -xvf /usr/local/
/usr/local/redis-5.0.7/
sudo make
redis-cli				//开启客户端
redis-server		//开启服务端
```

安装go-redis

```bash
go get github.com/go-redis/redis
```

在框架中使用

```go
var client *redis.Client //设置为类变量 以防每个函数都要初始化一遍

//初始化client
func InitRedisClient() {
	client = redis.NewClient(&redis.Options{
		Addr:     "127.0.0.1:6379",
		Password: "",
		DB:       0,
	})
}
//Redis操作 使用页码和topActicle作为key存储对象
func setTopArticleCache(page int, arts []Article) bool {
	result, covError := json.Marshal(arts)
	err := client.Set("topActicle"+strconv.Itoa(page), string(result), 0)
	if err == nil && covError == nil {
		return true
	}
	return false
}
//去redis中拿缓存
func getTopArticleCache(page int) []Article {
	result, err := client.Get("topActicle" + strconv.Itoa(page)).Result()
	var arts []Article
  //注意这个& 必须传入地址，不然arts拿不到值的
	covError := json.Unmarshal([]byte(result), &arts)
	if err == nil && covError == nil {
		return arts
	}
	return nil
}

//数据库操作层 如果缓存中有则从缓存中拿，没有的话在数据库中查出来后缓存到redis
func SelectTopArticle(page int) []Article {
	if getTopArticleCache(page) != nil {
		return getTopArticleCache(page)
	}
	start := (page - 1) * 6
	end := page * 6
	results, err := initDB.Db.Query("select * from supercool.article order by rate limit ?,?;", start, end)
	if err != nil {
		log.Print(err.Error())
	}
	defer results.Close()
	var artSlice []Article
	for results.Next() {
		var a Article
		err = results.Scan(&a.ArticleId, &a.Path, &a.ClassId, &a.Title, &a.SubTitle, &a.Date, &a.Rate, &a.Author)
		if err != nil {
			log.Print(err.Error())
		}
		artSlice = append(artSlice, a)
	}
	setTopArticleCache(page, artSlice)
	return artSlice
}
```

这边使用完之后，postman请求打过去发现居然要 10s ，我和我的小伙伴都惊呆了，debug后发现是初始化redis的时候 套接字 写成远端阿里云机器的端口了，改成本地的就正常了，第一次请求大概是 80ms 之后稳定低于这个值。

### 性能

使用JMeter压测接口，具体教程网上有，模拟100个用户，每个请求100次得到如下性能数据：

![image-20191130222349507](img/image-20191130222349507.png)

加了redis缓存之后性能数据如下：

![image-20191130224559120](img/image-20191130224559120.png)

综合对比发现平均时间大大下降，巅峰时期吞吐量达到80~100每秒。但是后期JMX卡顿可能是网络加载问题~

如果你发现性能效果不明显或者明显升高，那么

1.可能是set失败，redis-cli 去get以下看有没有这个值。

2.debug一下看get的值是不是nil