## 背景知识

### 文件描述符fd

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述**指向文件的引用**的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。在Linux系统中，流在内核中可以表示成文件的形式。

### IO模型

IO可以理解成对流的操作。

一般对于一个read操作发生时，它会经历两个阶段。

- 第一个阶段是等待数据准备。
- 第二个阶段是真正读取的过程，将数据从内核缓冲区拷贝到用户进程缓冲区中，

而五种常见的IO模型也是围绕这两个阶段来区分的。

- **同步模型（synchronous IO）**
  - 阻塞IO（bloking IO）
  - 非阻塞IO（non-blocking IO）
  - 多路复用IO（multiplexing IO）
  - 信号驱动式IO（signal-driven IO）
- **异步IO（asynchronous IO）**

其中，IO多路复用就是一种机制，实现一个进程可以监视多个描述符，一旦某个描述符就绪，就能够通知程序进行相应的读写操作。IO多路复用相比于多线程的优势在于系统的开销小，系统不必创建和维护进程或线程，免去了线程或进程的切换带来的开销。而操作系统支持IO多路复用的系统调用有select，poll和epoll。

## select

先来看看select的函数声明：

```c
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

`fd_set`是表示文件描述符集合的数据结构。readfds，writefds和exceptfds分别对应三类文件描述符集。当select被调用时，内部逻辑如下：

1. 将3个fd集copy到内核，这里限制了fd最大数量为1024
2. 线程阻塞，直到超时或内核检测到有fd可读或可写，内核会通知监控者select，select返回可读或可写的fd总数
3. 那么用户进程如何找到可读可写的fd呢？select会将之前传递给内核的fd集从内核copy到用户进程。用户进程通过遍历的方式找到可读可写的fd。

缺点：

1. copy次数过多，而且每次调用select方法都要进行fd集的copy操作
2. select监控fd数量有限
3. 用户进程通过遍历的方式找到可读写的fd，时间复杂度为o(n)，IO效率随着fd数量增多而线性下降

##  poll

先来看看poll的函数声明：

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

pollfd是表示文件描述符集合的数据结构。

```c
struct pollfd {
    int fd; //文件描述符
    short events; //监视的请求事件
    short revents; //已发生的事件
};
```

poll与select差不多，但poll的pollfd没有最大数量的限制，可是IO效率依旧没有提升orz。

## epoll

select/poll都只有一个方法，而epoll的操作过程有3个方法，分别是`epoll_create()`， `epoll_ctl()`，`epoll_wait()`。

### epoll_create()

```c
int epoll_create(int size)；//用于创建一个epoll的句柄，size是指监听的描述符个数。
```

该方法会在内核创建专属于epoll的高速cache区，并在该缓冲区建立红黑树和就绪链表，用户态传入的文件句柄将被放到红黑树中。

### epoll_ctl()

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
```

该方法对`epoll_create()`所创建的内核cache区进行操作的，操作对象是需要监听的fd。

比如，把要监听的fd注册到cache内，那么`epoll_ctl()`会将fd插入到红黑树中，并向内核注册了该fd的回调函数。内核在检测到某fd可读可写时则调用该回调函数，而回调函数的工作是将fd放到就绪链表。

###  epoll_wait()

```c
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

`epoll_wait`只需监控就绪链表，如果就绪链表有fd，则表示该fd可读可写，并返回给用户态（少量的copy）；

该函数返回需要处理的事件数目，如返回0表示已超时。

### 小结

执行epoll_create时，在创建了红黑树和就绪链表。执行epoll_ctl时，如果增加fd，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树上，然后向内核注册回调函数，用于当中断事件到来时向准备就绪链表中插入数据。执行epoll_wait时返回就绪链表里的数据即可。

因此，epoll比select和poll高效的原因是：

1. 减少了用户态和内核态之间文件句柄的copy
2. 降低了在文件句柄集中查找的时间复杂度。用红黑树维护fd集，可以将查找fd的时间复杂度降为o(logn)。

## 参考

- https://www.zhihu.com/question/20122137
- http://www.jianshu.com/p/dfd940e7fca2#
- http://gityuan.com/2015/12/06/linux_epoll/

