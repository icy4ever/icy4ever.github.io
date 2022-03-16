# Gin探索之旅 Part-I 路由注册

 	Gin的路由注册是我们启动框架处理Http请求的关键之一，路由注册到什么样的数据结构关系到了在Http服务启动后处理请求的响应速率。因此这是Gin Web框架设计的核心点之一，下面我们就来分析一下Gin的路由注册代码。

​	还是从最简单的demo实例出发：

```go
package main

import (
   "github.com/gin-gonic/gin"
   "net/http/httputil"
)

func main() {
   r := gin.Default()
   r.GET("/ping", func(c *gin.Context) {
      bs,_:=httputil.DumpRequest(c.Request, true)
      c.JSON(200, gin.H{
         "message": string(bs),
      })
   })
   r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

​	这里，r.GET函数将handler绑定到了/ping这个路径上。我们知道gin.Default函数的出参是Engine结构体，难道说Gin在设计上直接将GET、POST和一众路由注册函数都堆到了Engine这个结构体上了吗？显然不是，我们可以看到Engine中有个成员变量：

```go
type Engine struct {
   RouterGroup
   ...
}
```

​	这里很明显，Engine内嵌套了RouterGroup对象，因此他获得了RouterGroup的成员方法。回到demo代码，所以Engine指针可以直接使用路由注册方法注册Handler。

## RouterGroup结构体

```go
// RouterGroup is used internally to configure router, a RouterGroup is associated with
// a prefix and an array of handlers (middleware).
type RouterGroup struct {
   Handlers HandlersChain
   basePath string
   engine   *Engine
   root     bool
}
```

​	RouterGroup分别有四个成员，其各自的含义如下表所示：

| 变量名   | 变量类型      | 释义                                 |
| -------- | ------------- | ------------------------------------ |
| Handlers | HandlersChain | 该Path关联的Handlers                 |
| basePath | string        | 该路由组的路径前缀，用于组合绝对路径 |
| engine   | *Engine       | 该路由组归属的engine地址             |
| root     | bool          | 是否为根路由                         |

## POST方法的注册逻辑	

​	可能大家会问：为什么这里单挑POST方法来说明？因为POST和GET等方法底层逻辑是类似的，所以这里就拿最典型的POST来说明。

```go
// POST is a shortcut for router.Handle("POST", path, handle).
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
   return group.handle(http.MethodPost, relativePath, handlers)
}
```

​	可以明显看到，POST的底层是group.handle函数，这段不需要解释，让我们来看一下group.handle函数：

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
  	// 这里进去主要是将group结构体里的basePath和当前的相对路径组合成绝对路径
   absolutePath := group.calculateAbsolutePath(relativePath)
  // 将group所带的handler与该路径的handler组合得到该路径的《绝对handlers组》
   handlers = group.combineHandlers(handlers)
	// Mark1
   group.engine.addRoute(httpMethod, absolutePath, handlers)
   return group.returnObj()
}
```

​	上面代码的Mark1处，是比较复杂的部分。这里可以思考一个问题：

​		*为什么group不在自己这维护路由解析，而将它放到engine里呢？*

​	最浅显的一层其实是Engine维护了路由解析的前缀树。

## 前缀树与路由解析

​	按照惯例还是先贴一下上文中addRoute方法的代码：

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
   ...
		// 这里我们不细究前缀树的实现，这放到后面的章节，这里我们就知道他在拿对应方法的前缀树的根
   root := engine.trees.get(method)
   if root == nil {
     // 如果拿不到则新生成对应方法的根路径
      root = new(node)
     // 根节点Path自然为/
      root.fullPath = "/"
     // 添加到树的根节点切片中
      engine.trees = append(engine.trees, methodTree{method: method, root: root})
   }
  // 将对应的处理函数注册到树中
   root.addRoute(path, handlers)

   // 计算路径参数的个数并更新maxParams
   if paramsCount := countParams(path); paramsCount > engine.maxParams {
      engine.maxParams = paramsCount
   }
  
   // 计算路径的长度更新maxSections
   if sectionsCount := countSections(path); sectionsCount > engine.maxSections {
      engine.maxSections = sectionsCount
   }
}
```

​	到这里基本将routerGroup这个代码文件分析的差不多了，剩余一些方法如StaticFS、Static将在后续的（有空的话）去解析。🐦🐦🐦
