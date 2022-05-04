# 浅谈Golang数据结构之chan

​	chan作为goroutine交换数据的重要数据结构之一，在并发场景下被广泛使用。今天我们就来简单了解一下它的内部实现。我们先来简单的分析下数据结构：

```go
type hchan struct {
   qcount   uint           // queue中的数据总量
   dataqsiz uint           // 环型队列的大小
   buf      unsafe.Pointer // 存储数据的指针
   elemsize uint16 // 元素大小
   closed   uint32 // 是否关闭
   elemtype *_type // 元素类型
   sendx    uint   // buf的下一个发送位置
   recvx    uint   // buf的下一个接收位置
   recvq    waitq  // sudog队列，维护因为接收导致wait的G
   sendq    waitq  // sudog队列，维护因为发送导致wait的G

   lock mutex
}
```

我们可以看到，对于chan来说，他的实现主要依赖队列。它的同步语义主要由lock和recvq、sendq实现。

​	当我们使用make来初始化chan时，本质上会将chan分配在堆上（具体实现参考`mallocgc func`）。chan传递时是指针传递，所以无论是添加元素还是读取元素都是直接发生在堆上的。

## 发送

​	当我们使用chan发送数据时（使用`c<-elem`语法），Go会尝试从因读取chan数据而阻塞的`sudog`链表里获取头部G，将对应的数据传递给该G。此时该G的状态将由wait变为runnable。此时可以思考一个问题：为什么我们此时发送的数据可以直接到达G而不是放在队尾？因为只要有G在waiting状态说明此时队列的数据已经为空，所以直接发送数据给链表头部的G即可。

​	如果此时我们没有获取到G，那么代表并没有G在等待queue，此时我们直接将数据添加到buf里面并移动sendx。注意此时我们存入chan的数据即使是指针也是将值传入而不是存储指针，即放入chan的数据不会因外部数据的变化而变化。但如果数据已经满了即`qcount >= c.dataqsiz`。则会导致发送者（即本G）进入wait状态。

## 接收

​	对于接受来说，我们通常会有使用值（当queue使用）和不使用值（当信号量使用）两种。这两种语法分配如下：

```go
select {
	case <-c:
  // do recv logic
}

select {
  case elem, ok := <-c:
  // do recv logic
}
```

第一种情况下接受到的值会被直接丢弃，第二种情况下的我们会将接收到的值赋给elem。

​	类似于发送，接收操作也会尝试获取sendq链表里的G，如果sendq里有因为没有接收G而导致阻塞，此时会获取G并读取对应的值并将其状态由waiting变为runnable。

​	如果并没有获取到阻塞的G，且此时chan里元素数量大于0，可以直接从队列里获取数据。但是如果此时chan未空，则会导致读取G阻塞。

## 关闭

​	对于chan来说，由于chan是在堆上分配，我们需要及时的释放chan的资源时，需要使用`close`语法关闭chan。当关闭chan时，所有由于此读取此chan的G都会被直接释放（即由waiting状态变为runnable）。而所有写入此chan的G则会产生panic。

## 参考

- https://developpaper.com/chan-data-structure-and-understanding/
- https://www.velotio.com/engineering-blog/understanding-golang-channels
- https://github.com/golang/go

