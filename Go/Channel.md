# Channel

是一个名称为hchan的结构体的实例化指针，简单要记住的是循环链表buf，两个下表sendx/recvx和互斥锁lock

## 结构

```
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

## 原理

 https://studygolang.com/articles/20714 

### 接收数据/发送数据

①加锁

②复制数据到队列buf种/从队列种复制数据到goroutine中

③释放锁

### 缓存满情况

①G加锁后操作主动调用Go的调度器让出M

②把这个G抽象成sudog保存在sendq（send_queue）中

③直到另一个G取出数据时，加锁，channel会把sendq中的G推出，并把数据放入到buf中，释放锁

④调用Go调度器，唤醒G并放入可执行队列中

### 缓存空情况

①G加锁后操作主动调用Go的调度器让出M

②把这个G抽象成sudog保存在recvq（recv_queue）中

③直到另一个G发送数据时，channel会把recvq中的G推出，直接把数据复制到另一个Goroutine中

**（不加锁）**

④调用Go调度器，唤醒G并放入可执行队列中