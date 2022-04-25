# 浅谈Golang内存分配

​	对于很多高级语言来说，内存的分配和回收都是由语言本身实现的。而像c、c++这种则需要通过本身的构造和析构函数等去手动管理内存。虽然Golang底层也替开发者做了内存分配和垃圾回收操作，开发者本身还是需要对内存的分配有一定的了解才能在使用上发挥go的优势。下面我们就来简单分析一下go的内存设计与实现。

## 栈和堆
<img width="652" alt="企业微信截图_675aaf58-2bb5-4a9a-89f6-3703640d1d18" src="https://user-images.githubusercontent.com/38686456/165136299-f5362489-a267-4478-b923-5cf6eaa5df23.png">


​	内存中的数据往往存在于两个地方：堆和栈。简单的图如下所示：

![堆栈概览](https://user-images.githubusercontent.com/38686456/165136427-66d71fd6-4931-4014-8988-84698994a25d.png)


​	对于栈来说，它存储的数据是固定的，通常是我们在方法里声明的变量，在方法执行完毕后这些变量将不可达。而对于堆来说，它存储的数据通常是线程间共享的。其次，一些内存占用较大的数据也将在栈上分配。

​	下面我举一个Go程序的简单例子帮助理解栈的变量：

![栈的变量分配](https://user-images.githubusercontent.com/38686456/165136559-ea112351-09cc-4e8a-8b86-46e9193a859c.png)

​	程序执行main方法，声明a，此时会在栈中存储一个变量。然后b也会声明一个变量，该变量的值是double(a)的返回值。此时执行到double函数，如下所示：

![栈内存分配2](https://user-images.githubusercontent.com/38686456/165136702-b63dd503-5c9c-4115-82db-7d022536e69e.png)

​	当double执行完毕后，该方法的栈帧将不再被需要，此时b被赋值，double的栈帧将被标为未使用，相当于被pop出，此时执行fmt.Println方法，会覆盖掉原先double所占用的栈帧，如下图所示：

![栈内存分配3](https://user-images.githubusercontent.com/38686456/165136657-0035c1ce-0709-4443-873e-2e00d58788d3.png)

​	执行完毕后，栈帧将被回收，这里main方法其实运行在main的goroutine上。

​	其实对于Go来说，变量分配在堆和栈上其实是由编译器（complier）决定的，通常来说对于方法的返回值，如果返回指针，将可能导致内存逃逸。即本应该在栈中的变量被分配到堆中，比如下面这段代码：
![image-20220423220638627](https://user-images.githubusercontent.com/38686456/165136796-6e07f6c3-a163-4a1e-bafc-209c369b4229.png)

​	由于a需要引用pointer方法返回的num值的地址，但当pointer执行完毕后栈帧会被标记为未使用，所以该变量会被放到堆内存上。当然，如果编译器选择内联函数，即直接将pointer方法拷贝到main的栈帧中，该变量将被分配到栈上。可以用 `//go:noinline`来禁用内联函数。

​	相比于栈，堆的设计更加复杂。对于Go的堆来说一共有三个主要结构体：mcache、mcentral以及mheap。

### mcache

​	对于mcache来说，它主要是负责goroutine中的一些小对象的内存分配。由于它是和P（processor）关联的，所以不要加锁，因为一个P只会同时执行一个goroutine。

​	mcache的基础单位是mspan，同时mspan也是Go内存管理的基础单位。mspan一共有67个种类，每一个种类都被分为一个个大小均匀的块，存储于这些块里的对象大小都是固定的，具体类型和大小（byte为单位）如下：

```go
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 24, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

​	mspan是基于内存页来存储的，在Go中，一个内存页的大小默认为8kb。对于这些类型的mspan，具体一个mspan占用的内存页数量如下：

```go
var class_to_allocnpages = [_NumSizeClasses]uint8{0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 2, 1, 2, 1, 3, 2, 3, 1, 3, 2, 3, 4, 5, 6, 1, 7, 6, 5, 4, 3, 5, 7, 2, 9, 7, 5, 8, 3, 10, 7, 4}
```

​	mcache的结构具体如下图所示：

![mcache设计](https://user-images.githubusercontent.com/38686456/165136834-598712ad-95f4-403a-a228-1819ef93a669.png)

​	当mcache需要扩展时，则会从mheap对应的spanClass中拿对应类型的mspan。这里会优先拿被被清扫掉的mspan。

## mcentral

​	当我们的P分配掉单个mcache的同大小的mspan双向链表时，此时mcache需要补充同大小的mspan，此时就会向mcentral拿对应的mspan。mcentral就像是大型超市，给mcache（小商店）进货。它的结构体如下所示：

```go
type mcentral struct {
   spanclass spanClass

   partial [2]spanSet // list of spans with a free object
   full    [2]spanSet // list of spans with no free objects
}
```

​	可以看到，mcentral的spanClass代表了它所持有的mspan类型，partial代表至少还有一个空的mspan集合，full代表一个空闲空间都没有的span集合。

​	mcentral维护在mheap对象上，如下图所示：

![mcentral设计](https://user-images.githubusercontent.com/38686456/165136856-ca3018b2-b610-4d26-8fdd-ce7a9ae6373f.png)


​	当mcentral需要扩展时，则会从mheap里分配。

## mheap

​	mheap也就是堆对象，它由arena组成，单个arena的大小在64位架构的操作系统中位64M，而其他则为4M。当mheap需要内存时，它则会从操作系统中申请。整体如下图所示：

![mheap设计](https://user-images.githubusercontent.com/38686456/165136901-7fe1724a-19db-441c-bd1c-bbba2f764123.png)

## 内存分配过程

对于一个对象，它的分配会经历以下步骤：

1. 如果它满足小于16byte且不包含指针且子成员也不包含指针则会优先分配到tinyoffset所在的位置（相当于和其他对象拼凑在一个块里），如果tiny放不下，则在mspan中申请一个spanClass的大小。
2. 如果它不小于16byte或含有指针对象，那么它将会被分配在与其大小对应向上取整大小的mspan链表里（和上面的区别是不会和其他对象挤在一起）
3. 如果mcache里没有空闲，则去mcentral里拿partical span set里的mspan来补充到mcache该mspan类型的链表末尾。
4. 如果它大于32768byte（32kb），那么则会直接在mheap上分配，这里会直接分配对应需要大小的页数取整作为一个mspan存储。
5. 如果mheap内存不够，则会向操作系统申请。同时有可能触发GC（默认GC percent为100，也就是内存翻倍时会产生GC，这步刚好可能产生内存翻倍）

## 一些想法💡

​	内存分配这一块需要大量的时间去看源码，看博客，但是这能让我们对Go有更深层次的理解，同时对内存分配没有好的理解就无法搞透GC的原理，这两者之间是关系非常紧密的。所以，🤷‍♂️，没有什么特别好的捷径，只有好好搞透这些基础啦～

## 参考

- https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed
- https://medium.com/a-journey-with-go/go-memory-management-and-allocation-a7396d430f44
- https://github.com/golang/go

