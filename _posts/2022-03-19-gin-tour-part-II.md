# Gin探索之旅 Part-II 前缀树

```
接着上一章路由解析，本章我们继续看Gin的核心模块 - 前缀树的实现。由于这块参杂着通配符 ':' 和 '*' ，所以整体的代码复杂度会比较高。所以减少整篇的代码解析。
```

## 树的构造

 Gin不是整体用一颗树，而是根据方法来生成不同的树，如下图所示： [![gin路由](https://user-images.githubusercontent.com/38686456/159127106-3efd1171-5bf0-40e8-be37-4c6efc983626.png)](https://user-images.githubusercontent.com/38686456/159127106-3efd1171-5bf0-40e8-be37-4c6efc983626.png)

 每个Http请求方法都会维护一颗树的根节点。所以这里读者以为Gin构造自己的前缀树时，会根据 / 来分割路由来构造树。但现实情况是Gin并不会按照 / 来直接分割，而是采取一种Lazy Load的方式，举个🌰：

### 插入 

假设这是一颗空的树，我们在添加 /hello/world时会产生如下情况：

[![gin路由初始化](https://user-images.githubusercontent.com/38686456/159127126-c85d3ac0-05a7-4d1f-b15c-9082287e0227.png)](https://user-images.githubusercontent.com/38686456/159127126-c85d3ac0-05a7-4d1f-b15c-9082287e0227.png)

 但是，当你添加另一个具有公共前缀/hello的时候才会将该节点做进一步拆分，当我们添加/hello/michael时，会产生如下效果：

[![gin路由初始化 (1)](https://user-images.githubusercontent.com/38686456/159127134-01b1021d-de3d-4ce1-8ad0-ba731c5eecd0.png)](https://user-images.githubusercontent.com/38686456/159127134-01b1021d-de3d-4ce1-8ad0-ba731c5eecd0.png)

### 路由注册

 那么，对应处理链路handlers是如何注册到对应的节点上呢？

 答案是：所有的叶子节点都会维护他的一份处理函数切片，如下图所示： 
 ![gin前缀树插入节点](https://user-images.githubusercontent.com/38686456/159504701-b95aac73-3443-405b-9989-59d381103e99.png)

也就是说所有的节点（不一定是叶子节点）都会维护他的一份handlers。

## 查找

查找这块比较复杂，代码风格也是承袭了添加路由和插入路由的风格，查找会从根开始向下遍历。这边看过代码的同学很有可能会看到其数据结构中有一个indices字段，如下所示：

```go
type node struct {
   ...
   indices   string
	 ...
}
```

这个其实是子节点的第一个字母列表。维护这个是为了在访问子节点时直接通过这个字母来判断是否访问，减少访问次数。这里可能读者会疑问，那么路由字母重复怎么办？答案是既然重复，那必然节点还可以再拆分成子节点～

![gin节点失配](https://user-images.githubusercontent.com/38686456/159505198-cfe34139-cd17-49f2-a85d-4be447e7fff7.png)


但是由于通配符的存在，且通配符节点常常位于子节点的末尾，所以会存在需要回溯的情况，那么这是怎么解决的呢？ 如下图所示：

![gin节点通配](https://user-images.githubusercontent.com/38686456/159505344-7c998952-e23b-42de-871d-a97b1aafbfc5.png)


## 代码片段解析

 其实tree.go主要的函数只有两个：

- addRoute
- insertChild
- getValue

 下面分别简单解析一下这两个方法：

### insertChild

```
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

```
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

## getValue

```go
func (n *node) getValue(path string, params *Params, skippedNodes *[]skippedNode, unescape bool) (value nodeValue) {
   var globalParamsCount int16

walk: // Outer loop for walking the tree
   for {
     	// 当前的节点的路径
      prefix := n.path
     	// 当前获取的路径长度大于该节点的前缀长度
      if len(path) > len(prefix) {
         // 如果该节点的路径和path的前缀符合
         if path[:len(prefix)] == prefix {
            // 截取剩余的路径作为path
            path = path[len(prefix):]

            // 先尝试匹配所有的非通配符的路径
            idxc := path[0]
            for i, c := range []byte(n.indices) {
               if c == idxc {
                  // 如果是通配符子节点
                  if n.wildChild {
                     // 性能型扩容
                     index := len(*skippedNodes)
                     *skippedNodes = (*skippedNodes)[:index+1]
                     // 加入跳转节点
                     (*skippedNodes)[index] = skippedNode{
                        path: prefix + path,
                        node: &node{
                           path:      n.path,
                           wildChild: n.wildChild,
                           nType:     n.nType,
                           priority:  n.priority,
                           children:  n.children,
                           handlers:  n.handlers,
                           fullPath:  n.fullPath,
                        },
                        paramsCount: globalParamsCount,
                     }
                  }
									// 到对应的匹配indices的子节点去
                  n = n.children[i]
                  continue walk
               }
            }

            // 如果该节点没有通配符的子节点
            if !n.wildChild {
               // 如果最后剩余匹配的path不是/
               // 则需要回滚到上一个skipNodes
               if path != "/" {
                  for l := len(*skippedNodes); l > 0; {
                     // 将最后一个元素弹出
                     skippedNode := (*skippedNodes)[l-1]
                     *skippedNodes = (*skippedNodes)[:l-1]
                     // 如果匹配到该节点的后缀（因为加入时已经验证过前缀）
                     if strings.HasSuffix(skippedNode.path, path) {
                        // 回到那个节点
                        path = skippedNode.path
                        n = skippedNode.node
                        if value.params != nil {
                           *value.params = (*value.params)[:skippedNode.paramsCount]
                        }
                        globalParamsCount = skippedNode.paramsCount
                        continue walk
                     }
                  }
               }

              // 如果还是进入 len(path)>len(prefix) 的情况
              // 则尝试去掉path尾部的/
               value.tsr = path == "/" && n.handlers != nil
               return
            }

            // 获取尾部的通配符子节点
            n = n.children[len(n.children)-1]
            globalParamsCount++

            switch n.nType {
            case param:
               // 获取下一个/的下标或者字符串的尾
               end := 0
               for end < len(path) && path[end] != '/' {
                  end++
               }

               // Save param value
               if params != nil && cap(*params) > 0 {
                  if value.params == nil {
                     value.params = params
                  }
                  // 效率型扩容
                  i := len(*value.params)
                  *value.params = (*value.params)[:i+1]
                  val := path[:end]
                  // 如果标记了unescape则将/前面的元素解码
                  if unescape {
                     if v, err := url.QueryUnescape(val); err == nil {
                        val = v
                     }
                  }
                  // 将对应的通配符参数加入
                  (*value.params)[i] = Param{
                     Key:   n.path[1:],
                     Value: val,
                  }
               }

               // 如果没到path的最后
               if end < len(path) {
                  // 如果依然有children则继续遍历
                  if len(n.children) > 0 {
                     path = path[end:]
                     n = n.children[0]
                     continue walk
                  }

                  // ... but we can't
                  value.tsr = len(path) == end+1
                  return
               }
							 // 走到这代表已经匹配到path，返回值的handlers
               if value.handlers = n.handlers; value.handlers != nil {
                  value.fullPath = n.fullPath
                  return
               }
               // 如果该节点只有一个子节点
               if len(n.children) == 1 {
                  // No handle found. Check if a handle for this path + a
                  // trailing slash exists for TSR recommendation
                  n = n.children[0]
                  value.tsr = (n.path == "/" && n.handlers != nil) || (n.path == "" && n.indices == "/")
               }
               return
						// catchAll通配符
            case catchAll:
               // Save param value
               if params != nil {
                  if value.params == nil {
                     value.params = params
                  }
                  // Expand slice within preallocated capacity
                  i := len(*value.params)
                  *value.params = (*value.params)[:i+1]
                  val := path
                  if unescape {
                     if v, err := url.QueryUnescape(path); err == nil {
                        val = v
                     }
                  }
                  (*value.params)[i] = Param{
                     Key:   n.path[2:],
                     Value: val,
                  }
               }

               value.handlers = n.handlers
               value.fullPath = n.fullPath
               return

            default:
               panic("invalid node type")
            }
         }
      }

      if path == prefix {
         // 即使匹配到了，但是当前的path不为/并且该节点没有已注册的处理函数链，但最近匹配到的节点有其他子节点，则需要回滚到上次跳过的节点
         if n.handlers == nil && path != "/" {
            for l := len(*skippedNodes); l > 0; {
               skippedNode := (*skippedNodes)[l-1]
               *skippedNodes = (*skippedNodes)[:l-1]
               if strings.HasSuffix(skippedNode.path, path) {
                  path = skippedNode.path
                  n = skippedNode.node
                  if value.params != nil {
                     *value.params = (*value.params)[:skippedNode.paramsCount]
                  }
                  globalParamsCount = skippedNode.paramsCount
                  continue walk
               }
            }
            // n = latestNode.children[len(latestNode.children)-1]
         }
         // We should have reached the node containing the handle.
         // Check if this node has a handle registered.
         if value.handlers = n.handlers; value.handlers != nil {
            value.fullPath = n.fullPath
            return
         }

         // If there is no handle for this route, but this route has a
         // wildcard child, there must be a handle for this path with an
         // additional trailing slash
         if path == "/" && n.wildChild && n.nType != root {
            value.tsr = true
            return
         }

         // No handle found. Check if a handle for this path + a
         // trailing slash exists for trailing slash recommendation
         for i, c := range []byte(n.indices) {
            if c == '/' {
               n = n.children[i]
               value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
                  (n.nType == catchAll && n.children[0].handlers != nil)
               return
            }
         }

         return
      }

      // Nothing found. We can recommend to redirect to the same URL with an
      // extra trailing slash if a leaf exists for that path
      value.tsr = path == "/" ||
         (len(prefix) == len(path)+1 && prefix[len(path)] == '/' &&
            path == prefix[:len(prefix)-1] && n.handlers != nil)

      // roll back to last valid skippedNode
      if !value.tsr && path != "/" {
         for l := len(*skippedNodes); l > 0; {
            skippedNode := (*skippedNodes)[l-1]
            *skippedNodes = (*skippedNodes)[:l-1]
            if strings.HasSuffix(skippedNode.path, path) {
               path = skippedNode.path
               n = skippedNode.node
               if value.params != nil {
                  *value.params = (*value.params)[:skippedNode.paramsCount]
               }
               globalParamsCount = skippedNode.paramsCount
               continue walk
            }
         }
      }

      return
   }
}
```

 总体来说，tree代码由于通配符的原因，增加了代码的复杂度，其中也有很多被抱怨的点，可以在Gin的issue中找到，如[gin-gonic/gin#2663](https://github.com/gin-gonic/gin/pull/2663) 有兴趣的可以去研究一下～



