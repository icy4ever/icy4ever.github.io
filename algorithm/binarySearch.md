## 如何写好一个二分查找

​	在Go语言中，当需要在切片中寻找一个元素时，我们可以利用sort.Search函数。但是我认为这个函数并不应该暴露出来。从语义上来讲，search代表搜索，当它提供函数参数的时候，我认为我应该传入一个函数来判断是否是我要找的数。举个例子：

```go
// 在[0,10)中找7
sort.Search(11, func(n int) bool {
    return n==7
})
```

但是这样将永远找不到7，看一下sort.Search的源码：（Go 1.16）

```go
func Search(n int, f func(int) bool) int {
   i, j := 0, n
   for i < j {
      h := int(uint(i+j) >> 1)  // 防止溢出
      if !f(h) {
          i = h + 1 // 不满足就在[h+1,n)区间找
      } else {
          j = h // 如果满足给定条件，就在[0,h)区间查找
      }
   }
   return i
}
```

所以我们的f应该定义为选择左区间的情况。