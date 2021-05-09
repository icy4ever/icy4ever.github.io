### 类型

```go
type bmap struct {
	// tophash 通常包含了这个桶里所有Key哈希值的高位 如果 tophash[0] < minTopHash 代表 tophash[0] 是一个正在扩容的桶
	tophash [bucketCnt]uint8
	// bucketCnt 键值依次排列
	// 解释: 将所有的key排在一起后面接着所有的值比 key/elem/key/elem/... 这样排列更复杂，但是它帮助我们消除例如 map[int64]int8 的填充
	// 伴随着一个溢出的指针.
}
```

```go
// Go map的一个头.
type hmap struct {
	count     int 	 // 元素个数
	flags     uint8  // 标示Map当前正在执行的操作（删除或添加元素）
	B         uint8  // 可容纳2^B个元素
	noverflow uint16 // 溢出桶的数量
	hash0     uint32 // hash seed （一般哈希需要指定seed）
	buckets    unsafe.Pointer // 2^B容量的Bucket数组指针. 如果count为0，则可能为nil.
	oldbuckets unsafe.Pointer // 扩容的时候才使用，此时buckets是oldbuckets的两倍
	nevacuate  uintptr        // 扩容进度 (小于这个的buckets说明已经完成)
	extra *mapextra 					// 可选项
}

// mapextra 拥有不在所有maps里的区域
type mapextra struct {
	// 如果键值都不包含指针, 我们将桶的类型标记为无指针. 这样防止了扫描此种Map
	// 但是, bmap.overflow 是一个指针. 为了使溢出的桶活着, 我们将指针存储在所有溢出的桶的 		  hmap.extra.overflow和hmap.extra.oldoverflow字段里.
	// overflow 和 oldoverflow 只有在键值都不包含指针的时候才会存在
	// overflow 为 hmap.buckets 记录溢出的桶.
	// oldoverflow 为 hmap.oldbuckets 记录溢出的桶.
	// 这样间接的行为有助于快速地将指针存储于切片
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow 保存了下一个空的溢出桶.
	nextOverflow *bmap
}
```

```go
// 一个哈希遍历结构
type hiter struct {
	key         unsafe.Pointer // 必须在第一位.  当是最后一个元素的时候 key为nil (详情查阅 cmd/internal/gc/range.go).
	elem        unsafe.Pointer // 必须在第二位 (详情查阅 cmd/internal/gc/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // 保持hmap.buckets的溢出桶活着（做指针引用防止被GC掉？）
	oldoverflow *[]*bmap       // 保持hmap.oldbuckets的溢出桶活着
	startBucket uintptr        // 桶迭代的起始地址
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // 是否从头到尾的桶都被包裹了
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}
```

```go
// hmap 的扩展字段
type mapextra struct {
   overflow    *[]*bmap //为了防止新增的溢出bucket被回收，将它存在hmap的extra字段里
   oldoverflow *[]*bmap //和overflow一样，只是它存 oldoverflow的溢出bucket

   nextOverflow *bmap		//这个变量拥有一个指向下个空闲溢出bucket的指针
}
```

### 方法

#### tophash：

```go
func tophash(hash uintptr) uint8 {
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	if top < minTopHash {
		top += minTopHash
	}
	return top
}
```

这个方法主要是拿哈希值的前8位

#### incrnoverflow：

```go
func (h *hmap) incrnoverflow() {
   if h.B < 16 {
      h.noverflow++
      return
   }
   mask := uint32(1)<<(h.B-15) - 1
   if fastrand()&mask == 0 {
      h.noverflow++
   }
}
```

命名上来看，这个是一个增加 Map 的 noverflow 值的方法。

当小于16的时候，直接增加，当大于等于16的时候概率增加。概率为 1 / ( 2^( h.B - 15) -1 )。

#### createOverflow:

```go
func (h *hmap) createOverflow() {
   if h.extra == nil {
      h.extra = new(mapextra)
   }
   if h.extra.overflow == nil {
      h.extra.overflow = new([]*bmap)
   }
}
```

在hmap的extra字段里创建一个bucket切片存储溢出的buckets

#### newoverflow:

```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
   var ovf *bmap
   // 一般来说，如果调用了createOverflow，判断是成立的
   if h.extra != nil && h.extra.nextOverflow != nil {
      ovf = h.extra.nextOverflow
      if ovf.overflow(t) == nil {
         h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
      } else {
         ovf.setoverflow(t, nil)
         h.extra.nextOverflow = nil
      }
   } else {
      ovf = (*bmap)(newobject(t.bucket))
   }
   h.incrnoverflow()
   if t.bucket.ptrdata == 0 {
      h.createOverflow()
      *h.extra.overflow = append(*h.extra.overflow, ovf)
   }
   b.setoverflow(t, ovf)
   return ovf
}
```

从命名上和返回值上来看，这个方法是用来创建一个Bucket的