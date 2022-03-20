# Gin探索之旅 Part-II 前缀树

 	接着上一章路由解析，本章我们继续看Gin的核心模块 - 前缀树的实现。由于这块参杂着通配符 ':' 和 '*' ，所以整体的代码复杂度会比较高。所以减少整篇的代码解析。

## 树的构造

​	Gin不是整体用一颗树，而是根据方法来生成不同的树，如下图所示：
![gin路由](https://user-images.githubusercontent.com/38686456/159127106-3efd1171-5bf0-40e8-be37-4c6efc983626.png)

​	每个Http请求方法都会维护一颗树的根节点。所以这里读者以为Gin构造自己的前缀树时，会根据 / 来分割路由来构造树。但现实情况是Gin并不会按照 / 来直接分割，而是采取一种Lazy Load的方式，举个🌰：

​	假设这是一颗空的树，我们在添加 /hello/world时会产生如下情况：

![gin路由初始化](https://user-images.githubusercontent.com/38686456/159127126-c85d3ac0-05a7-4d1f-b15c-9082287e0227.png)

​	但是，当你添加另一个具有公共前缀/hello的时候才会将该节点做进一步拆分，当我们添加/hello/michael时，会产生如下效果：

![gin路由初始化 (1)](https://user-images.githubusercontent.com/38686456/159127134-01b1021d-de3d-4ce1-8ad0-ba731c5eecd0.png)

​	那么，对应处理链路handlers是如何注册到对应的节点上呢？

​	答案是：所有的叶子节点都会维护他的一份处理函数切片，如下图所示：
![gin路由初始化 (2)](https://user-images.githubusercontent.com/38686456/159127139-9d6cf5cd-c39a-4e34-ad80-057868fd7487.png)


## 代码片段解析

​	其实tree.go主要的函数只有两个：

- addRoute
- insertChild

​	下面分别简单解析一下这两个方法：

### insertChild

```go
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
   for {
      // 这里解析Path里是否有通配符
      wildcard, i, valid := findWildcard(path)
      if i < 0 { // No wildcard found
         break
      }

      // 如果找到并且是非法的path则直接panic
      if !valid {
         panic("only one wildcard per path segment is allowed, has: '" +
            wildcard + "' in path '" + fullPath + "'")
      }

      // 如果只有通配符也会panic
      if len(wildcard) < 2 {
         panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
      }
      // 如果是:类型的通配符
      if wildcard[0] == ':' { // param
         // 拆分掉path，将通配符后缀拿出
         if i > 0 {
            // Insert prefix before the current wildcard
            n.path = path[:i]
            path = path[i:]
         }
				 // 类型是Param的节点
         child := &node{
            nType:    param,
            path:     wildcard,
            fullPath: fullPath,
         }
         n.addChild(child)
         n.wildChild = true
         n = child
         n.priority++

         // 如果此时path还未全部解析完毕
         if len(wildcard) < len(path) {
            path = path[len(wildcard):]

            child := &node{
               priority: 1,
               fullPath: fullPath,
            }
            n.addChild(child)
            n = child
            continue
         }

         // 解析完毕直接将handlers添加到对应叶子节点中
         n.handlers = handlers
         return
      }

      // 如果*不在最后，则报错
      if i+len(wildcard) != len(path) {
         panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
      }

      if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
         pathSeg := strings.SplitN(n.children[0].path, "/", 2)[0]
         panic("catch-all wildcard '" + path +
            "' in new path '" + fullPath +
            "' conflicts with existing path segment '" + pathSeg +
            "' in existing prefix '" + n.path + pathSeg +
            "'")
      }

      // currently fixed width 1 for '/'
      i--
      if path[i] != '/' {
         panic("no / before catch-all in path '" + fullPath + "'")
      }

      n.path = path[:i]

      // First node: catchAll node with empty path
      child := &node{
         wildChild: true,
         nType:     catchAll,
         fullPath:  fullPath,
      }

      n.addChild(child)
      n.indices = string('/')
      n = child
      n.priority++

      // second node: node holding the variable
      child = &node{
         path:     path[i:],
         nType:    catchAll,
         handlers: handlers,
         priority: 1,
         fullPath: fullPath,
      }
      n.children = []*node{child}

      return
   }

   // 这里处理没有wildcard的情况
   n.path = path
   n.handlers = handlers
   n.fullPath = fullPath
}
```

### addRoute

```go
// addRoute adds a node with the given handle to the path.
// Not concurrency-safe!
func (n *node) addRoute(path string, handlers HandlersChain) {
   fullPath := path
   n.priority++

   // 空树则直接插入
   if len(n.path) == 0 && len(n.children) == 0 {
      n.insertChild(path, fullPath, handlers)
      n.nType = root
      return
   }

   parentFullPathIndex := 0

walk:
   for {
      // 这里将拿到插入的path和被插入的节点的公共前缀下标
      i := longestCommonPrefix(path, n.path)

      // 这里拆分旧的前缀节点
      if i < len(n.path) {
         ...
      }

      // 这里插入path的非公共前缀节点
      if i < len(path) {
        ...
      }
   }
}
```

​	总体来说，tree代码由于通配符的原因，增加了代码的复杂度，其中也有很多被抱怨的点，可以在Gin的issue中找到，如https://github.com/gin-gonic/gin/pull/2663 有兴趣的可以去研究一下～
