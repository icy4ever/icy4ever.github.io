# Gin框架总结

 	从之前几章的分析，基本走过了Gin的一个请求开始到一个请求返回的过程，本章让我们来回顾一下这段过程。

## 1.Engine结构体实例创建

	第一步就是Engine结构体的创建，这里过程比较简单，不作赘述。

## 2.注册路由handler

	第二步将用户对应的处理函数注册到engine维护的前缀树中，这个树因为之前已经对其做了拆分，这里将列出他的整体结构：


![gin方法树](https://user-images.githubusercontent.com/38686456/159926279-bb5f1095-3281-4f43-8835-ee9cb9142246.png)


	每个节点都有可能存在handler，这和前缀树中的isEnd标志位相似。这当中的indices是最容易误解的东西，之前的文章也有讲，它其实是一个快速索引，每个indices都是唯一的，因为如果重复将会被分裂成新的节点。（所以前面的文章图会产生误解🧎‍♂️）同时⚠️通配符的路径会放在末尾，作最低优先级匹配。

## 3.请求生命周期

### 3.1 Context分配

	Gin为了更好的性能，分配和回收Context对象使用了sync.Pool做池子，因为池子就是为长生命周期对象的频繁分配用的。所以这里每次收到一个请求时都会从池子里拿一个Context对象，用完后再放回去。

### 3.2 前缀树遍历

  前缀树遍历时会先取对应的方法下的树的根节点，然后向下开始遍历，这里有一个注意点，比方说如下这种情况：

![gin方法树遍历](https://user-images.githubusercontent.com/38686456/159930677-9624e358-ea53-46fc-aab9-eac08ef96049.png)

​	假设我们的请求为 GET /hello/world/go，按照逻辑，我们会先走12这条路径，但是我们遍历后会失配。这个时候怎么处理的呢，其实我们在遍历有通配符的节点时，会存储一个skippedNodes切片（并且indices被置为空），如果发生失配就会回到上一级节点，由于没了indices此次只会直接访问通配符对应的节点。

### 3.3 c.Next的魔力

​	我们常常需要c.Next来控制handler的执行顺序，但是这是怎么做到的呢？比如如下代码：

```go
func TestMiddlewareGeneralCase(t *testing.T) {
	signature := ""
	router := New()
	router.Use(func(c *Context) {
		signature += "A"
		c.Next()
		signature += "B"
	})
	router.Use(func(c *Context) {
		signature += "C"
	})
	router.GET("/", func(c *Context) {
		signature += "D"
	})
	// RUN
	w := PerformRequest(router, "GET", "/")

	// TEST
	assert.Equal(t, "ACDB", signature)
}
```

​	为什么这里会有一个嵌套的效果，明明c.Next是按照顺序执行的。因为在signature被添加A后，c.Next将会拿到下一个需要执行的middleware。其实由于Gin调用也是c.Next来执行middleware的，所以这是一种递归调用。

## 结语

​	好了，整体的Gin代码的主要流程都跑了一遍，主要了解了Gin的核心-前缀树及Context分配，handler的执行逻辑等。后续有空会继续钻研Gin相关的设计～🍻

