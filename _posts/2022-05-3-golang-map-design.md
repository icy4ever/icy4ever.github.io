# 浅谈Golang数据结构之Map

![image](https://user-images.githubusercontent.com/38686456/166622575-6fe97764-db82-4388-81b0-7fc9e991234c.png)

​	Go语言在runtime里实现了hashmap的数据结构，也就是我们常用的map。虽然经常使用，但是很少有机会看它的底层实现，所以今天有空就简单分析一下Go的map实现。

## hmap

​	hmap作为map的基础结构体承载了map的基本信息。它的结构体和释义如下：

```go
// A header for a Go map.
type hmap struct {
   count     int // 元素数量，当我们使用len时可以直接读取，所以读取长度属于O(1)复杂度
   flags     uint8 // map处于的状态
   B         uint8  // 持有Bucket的数量的log_2，比如：B为2时，数量即为1<<2
   noverflow uint16 // 溢出的bucket数量（过多的hash冲突可能会导致bucket溢出）
   hash0     uint32 // hash种子，和rand.seed类似

   buckets    unsafe.Pointer // bucket数组
   oldbuckets unsafe.Pointer // 扩容前的数组
   nevacuate  uintptr        // 迁移的bucket数量

   extra *mapextra // optional fields
}
```

​	可以看到，hmap维护了包括map长度、bucket大小、bucket数组地址和扩容前bucket地址等信息。但是到这里我们还是一头雾水，因为我们并不清楚bucket是什么。

## bmap

​	bmap本质就是bucket，但是它的结构体看上去让人困惑，因为我们无法从它的结构体看出它的数据存放。

```go
// A bucket for a Go map.
type bmap struct {
   tophash [bucketCnt]uint8
}
```
`tophash`是一个大小为8的uint8的数组（意味着kv数也是8），它并不用来存放数据的指针（uint8也存不了指针对吧～），而是存放对应数据的状态的（如`4 evacuatedEmpty`代表数据已经迁移、`1 emptyOne`代表此块是空的）。它的数据将存放在tophash随后的内存块中，Go通常使用

```go
dataOffset = unsafe.Offsetof(struct {
   b bmap
   v int64
}{}.v)
```

来获取bmap随后的数据。

​	那么，它的数据存储格式是怎么样的呢？Go为了减少数据补齐，将map的key放在一起，value放在一起，举个例子，当key为int64类型，而值为bool类型时，与每个kv一组形式对比如下图所示：

![bucket键值变化](https://user-images.githubusercontent.com/38686456/166621208-bfd3c35b-3e37-4a7b-a3b1-b1e351b23c1c.png)

很明显，对于这种方式来说减少了内存padding，增加了内存利用率。以上图为例，左边的实现方式比右边的实现方式增加了24Byte的内存消耗。

## map初始化

​	通常，我们会使用`make`做map的初始化操作，对于make来说，我们可以选择是否传入map的size，Go并不会直接生成与size相同的bucket数量。而是通过传入的元素数量来计算`hmap.B`的大小，这和**负载因子**有关，在Go 1.16.5 版本中，负载因子为6.5，这意味着如果我们传入size为8时，B的值为1（即会产生2个bucket）。至此我们可以得到一个大致的map的形状：
![map设计](https://user-images.githubusercontent.com/38686456/166621218-6b046604-59df-4877-a8f0-5209da5e834d.png)

*tips：颜色代表内存块被使用，白色代表未使用*

下表显示了不同负载因子对map性能及cpu负载的影响：

```go
a simple program to check some stats for different loads:
(64-bit, 8 byte keys and elems)
 loadFactor    %overflow  bytes/entry     hitprobe    missprobe
       4.00         2.13        20.77         3.00         4.00
       4.50         4.05        17.30         3.25         4.50
       5.00         6.85        14.77         3.50         5.00
       5.50        10.55        12.94         3.75         5.50
       6.00        15.27        11.67         4.00         6.00
       6.50        20.90        10.79         4.25         6.50
       7.00        27.14        10.15         4.50         7.00
       7.50        34.03         9.73         4.75         7.50
       8.00        41.10         9.40         5.00         8.00

%overflow   = 含有溢出bucket占总bucket的比例
bytes/entry = 每个kv多用的字节
hitprobe    = 查找一个存在的key需要的操作数
missprobe   = 查找一个不存在的key需要的操作数
```

## map赋值

​	相比于map初始化，map赋值会稍微复杂一些。通常对于一个hashmap来说，它需要采用的hash算法必须能使hash后的key均匀分布。对于Go来说，它首先会产生一个seed保存在hmap的hash0值中。当需要hash时，他会使用seed和hasher来对key进行hash。我们得到hash值后，会把hash值的低位与`1<<B-1`做与运算，获得对应的数字即为数据存放的bucket，如果此时bucket还有空位，everything is ok，我们将kv存放到对应位置。

​	那么如果该bucket没有位置了呢？此时，该bucket会产生一次overflow，也就是将对应的bucket扩容一倍，并标记此bucket和记录overflow的次数。看起来如下图所示：
![bmap的overflow](https://user-images.githubusercontent.com/38686456/166621228-0f1f4cd0-ca50-4635-b6c1-7abd46a4c795.png)

但是这样扩容下去，map在hash碰撞频繁的情况下将产生极大的性能损耗。Go会在每次赋值后检查是否内部元素已经超过了当前容量的6.5 / 8，如果超过了将会产生一次扩容。

​	对于Java来说，它会使用红黑树减少元素查找次数。而对于Go来说，他将产生一个二倍于现在的bucket数组来代替原先的数组，但是并不会全部一次性将所有元素迁移过去（这会产生赋值时的性能不平滑）。相反的，它会在每次读取时迁移对应hash到的bucket里的元素。当然，对于原bucket数组里的元素，会被分散到新bucket数组里，如下图所示：
![bucket迁移](https://user-images.githubusercontent.com/38686456/166621327-6b25c8ce-2a1f-4cc7-81ca-46061377e770.png)

具体元素迁移到`low`（低位）还是`high`高位，是由一系列因素决定的（原键hash后的值、原bucket的状态等）。

​	Go通过这样的扩容手段保证了map的查找依旧可以在O(1)的复杂度内完成。剩余的赋值操作就比较简单，找到对应的bucket并顺序找到内存块赋值即可。

## map读取

​	map读取和赋值类似，它遵循以下步骤：

1. 计算key对应的hash值，并与`1<<B`做与运算计算出key所属的bucket。
2. 如果此时`oldbuckets`不为nil，代表处于搬离状态，此时我们读值依然从hmap的`oldbuckets`里面读。此时会和`1<<(B-1)`做与计算（即双倍扩容前的B）。更新获取到的bucket。
3. 拿到bucket后，每次我们通过overflow来获取bucket下一个扩容的bucket，直到为nil。对比里面的每个key，直到获取到键值。如下图所示：
![bucket寻找键值](https://user-images.githubusercontent.com/38686456/166621492-0bc18578-1407-4cab-8749-9c032654c3f4.png)

## 一些感想

​	其实不难看出，map的设计还是比较简单的，相对于GC来说没有那么复杂。对于map来说，无非就是找到bucket然后遍历即可。对于赋值来说，判断是否超过负载因子然后做双倍扩容以及数据的迁移～

## 参考

- [Go: Map Design by Example — Part I](https://medium.com/a-journey-with-go/go-map-design-by-example-part-i-3f78a064a352)
- [golang souce code 1.16.5](https://github.com/golang/go)

