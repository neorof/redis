> **redis服务器需要处理2类事件：**
>
> 1. 文件事件：文件事件是服务器对套接字的抽象，服务器与客户端的通信会产生相应的文件事件，服务器通过监听并处理这些事件完成通信操作。
> 2. 时间事件： redis服务器有些操作需要定期执行，就是时间事件。

## 文件事件

> redis使用io多路复用来监听多个套接字，并根据套接字目前执行的任务来关联不同的事件处理器，当被监听的套接字准备好执行连接应答、读取、写入、关闭等操作，对应的文件事件就会产生。
>
> 虽然文件事件以单线程运行，但通过io多路复用提高了性能，又可以很好的与redis服务器中其他同样以单线程方式运行的模块对接，保持了单线程设计的简单性

#### 文件事件处理器的构成

![](https://upload-images.jianshu.io/upload_images/15199742-1c852f9eab98cbfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



服务器同时连接多个套接字，所以文件事件可能并发的出现，但io多路复用总是会将事件都放到一个队列里，然后通过这个队列以**有序，同步，每次一个套接字** 方式向事件分派器传送套接字。

![](https://upload-images.jianshu.io/upload_images/15199742-4a191aee3d5ffee3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### io多路复用程序的实现

>  io多路复用通过包装常见的select， epoll，evport，kqueue函数库实现，每个函数库都对应一个单独的文件如ae_select,ae_epoll等。redis为每个库都实现了相同的API，所以底层是可以互换的。程序在编译时自动选择系统中性能最高的底层实现。

#### 事件的类型

1. 可读 || 可应答（客户端connect）产生AE_READABLE
2. 可写产生AE_WRITEABLE

当某个套接字同时产生这2个事件，先处理读事件。

#### 通信过程

![](https://upload-images.jianshu.io/upload_images/15199742-ff75a4f022010dc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 时间事件

> 时间事件分为2类：
>
> 1. 定时事件：比如让程序在x时间后的30s执行
> 2. 周期性时间：让程序每隔30s执行一次

#### 时间事件组成

1. id。递增，新比旧大
2. when，记录事件到达时间
3. timeProc，事件处理器，一个函数

> 一个时间事件属于定时还是定期取决于事件处理器的返回，如果是AE_NOMORE则是定时事件，如果返回另一个时间点，那么就是定期，另一个时间点就是下次执行时间。

> **目前的redis版本只支持定期时间而没有使用定时事件**

#### 实现

服务器将所有的时间事件放在一个无序链表，每当时间事件执行器执行时遍历链表，查询时间已到达的，调用处理器处理。

> 无序指的是事件不按时间排序，所以需要遍历整个链表
>
> 但无序链表不影响性能，因为正常模式下的redis只有一个serverCron事件，相当于指针。

![](https://upload-images.jianshu.io/upload_images/15199742-0b64c74a751c8235.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

