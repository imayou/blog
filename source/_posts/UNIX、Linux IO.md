---
layout: unix、linux
title: UNIX、Linux IO
date: 2019-04-29 20:53:28
tags:
---
> 前提
- [网络知识梳理--OSI七层网络与TCP/IP五层网络架构及二层/三层网络](https://www.cnblogs.com/kevingrace/p/5909719.html)
- [报文详解](http://www.cnblogs.com/GumpYan/p/5819933.html)

#### 3次握手建立连接
1. 第一次握手：Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。
2. 第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
3. 第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了。

#### 4次挥手断开连接
> 由于TCP连接时全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成数据发送任务后，发送一个FIN来终止这一方向的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再收到数据了，但是在这个TCP连接上仍然能够发送数据，直到这一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭，上图描述的即是如此。
1. 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
2. 第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
3. 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
4. 第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。



#### 套接字（UNIX Linux程序设计教程12.2章）
套接字是应用程序和底层网络协议之间的一个通信端口，这意味`想要读来自网络的数据就读套接字，想要写数据至网络就写套接字，想要控制网络协议选项则要设置套接字`。从程序员的角度来看，套接字等价于网络，就行文件描述字是磁盘操作的一个端口一样。
使用套接字通信必须首先创建套接字，相互通信的两个镜像都必须调用`socket()`来建立自己一端的套接字。
```c
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```
以上函数在通信域`domain`(套接字协议族)中创建一个类型为`type`，使用协议为`protocol`的套接字，并返回一个`文件描述字`，即`套接字描述字`。一个描述字要么代表一个打开的文件，要么代表一个套接字，不会同时代表两者。套接字描述字和文件描述字可以在许多函数中互相通用，例如，`close()`可以销毁一个套接字；`read()`、`write()`可以读写套接字。但是，与管道历史，套接字不支持`文件定位`操作。
> 数据传输的单位有些是字节流，其他的则将若干字节组成一个记录，也称为“数据包”。

#### 文件描述字的打开、创建和关闭（UNIX Linux程序设计教程3.1章）
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char * filename, int flages[, mode_t mode]);
int create(const char * filename, mode_t mode);
```
open()调用成功返回为文件的filename新创建的一个描述字，同时文件位置定位于文件的开始处；调用失败不会创建或修改文件。如果两个进程同时打开一个文件，系统会保证各自有自己的文字描述字。
#### 文件锁
文件锁根据其访问分为读锁和写锁，读锁也称为“共享读锁”或“共享锁”；写锁也称为“互斥写锁”或“互斥锁”

#### 非阻塞I/O（UNIX Linux程序设计教程3.7章）
在默认情况下关于I/O的系统调用总是阻塞的，调用必须等待操作完成，即读写到数据才能返回，但是慢(UNIX根据阻塞还是非阻塞分为2类，一类是所谓的“慢”系统调用，其他的则归为另一类)系统调用并不包括所有阻塞I/O。

非阻塞I/O可以使用I/O操作如open()、read()或write()不会被永久阻塞，如果操作不能完成，这些调用将立即返回并给出错误码指明操作可能被阻塞。

对给定的文件描述字，有两种方法指定非阻塞I/O：
1. 在open()时指定O_NONBLOCK文件状态标志
2. 对已经打开的描述字，调用fcntl()函数设置O_NONBLOCK文件状态标志

（例子3-7）在非阻塞I/O情况下传输数据，当读写不能立即完成时，会返回-1并置errno为EAGAIN。这时往往需要再次(for轮询)调用read()或write()。
这类循环再次调用称为“轮询”，它存在2个问题，多用户系统中它浪费了CPU时间，本来这些用于等待的CPU时间可以做其他事情或者让给其他经常。另外，它仍然没有避免永久阻塞的问题，虽然write()能立即返回，但它可能由于总是不能写出数据而出现`死循环`，我们必须设置一定的时间限制，以便时间到时能跳出循环。
#### 信号驱动的I/O（UNIX Linux程序设计教程10.2章）
上面(3.7章)的非阻塞I/O尽管不阻塞进程，但为了知道描述字上是否有可读数据，进程必须用轮询的方法不断调用read()。使用信号驱动的I/O可以避免这种浪费资源的轮询。

采用信号驱动的I/O，当在描述字上有数据到达时进程会收到一个信号，此时对该描述字进行输入输出操作将保证不会被阻塞。这样在数据未到达这段时间进程可以进行其他工作，并在确知数据已到达的情况下才发出I/O调用。习惯上也称信号驱动的I/O方式为异步I/O（本质上还是同步I/O）。

信号驱动的I/O较为复杂且每个进程只有一个`SIGIO`信号（10.3节），而且不能区别文件描述符产生的信号，为了监控多个输入设备仍然避免不了轮询。解决这类问题最好的方法是使用`select()`和`poll()`--`多路转接I/O`。

`select()`(10.3.1)函数告诉内核：调用它的进程需要等待多种I/O事件，并且仅当这些事件中的一个或多个出现时，或者指定的时间已经过去时才唤醒调用它的进程。
``` c
#include <sys/select.h>
#include <sys/time.h>
int select(int nfds, fd_set *rfds, fd_set * wfds, fd_set * efds, struct timeval *timeout)
```
`timeout`参数告诉内核至多等待多长时间，有3种情形：永远等待、等待固定的一段时间、不等待。

`poll()`(10.3.2)与`select()`在本质上是一样的，都检查一组文件描述字，只是方式不同。`poll()`检查一组文件描述字以查看是否有任何悬挂的事件，并且可以有选择的为等待某个描述字上的事件而指定时间。
``` c
#include <pool.h>
int poll(struct pollfd fds[], nfds_t nfds, int timeout)
```
与`select()`不同，`poll()`通过一个其元素为pollfd类型的结构数组fds指明要等待的描述字集合以及其上的事件。

#### 异步I/O（10.4）
- 同步I/O操作导致发出请求的进程被阻塞直到I/O操作完成。
- 异步I/O操作在I/O期间不导致发出请求的进程被阻塞。
根据这个定义，阻塞、非阻塞、多路转接、信号驱动的I/O均是同步I/O。因为这几种方式一旦I/O操作（read()/write()），进程便需要等待其完成才能继续执行。异步I/O在I/O操作实际间不阻塞进程，进程仍然可以并行的执行其他任务。
使用异步I/O编程典型步骤(10.4.5)
1. 调用open()打开指定的文件获取文件描述字，然后使文件位置指针指向文件开始并选择适当的标志。
2. 创建并填充异步I/O控制块的aiocb。
3. 如果使用型号，建立信号句柄捕获异步I/O操作完成信号。
4. 使用aio_read()、aio_write()或lio_listio()请求异步I/O操作，或调用aio_sync()使异步I/O与磁盘同步。
5. 如果应用需要等待I/O完成，调用aio_suspend(),或者继续执行并调用aio_error()查询操作是否完成，或者继续执行直到有信号到达。
6. I/O操作完成之后，调用aio_erturn()抽取返回值。
7. 调用close()关闭文件。close()在关闭文件之前将等待所有异步I/O完成。


> 升级
- [高性能网络服务器编程：为什么linux下epoll是最好，Netty要比NIO.2好？](https://www.cnblogs.com/tianhangzhang/p/5374034.html)
- [聊聊IO多路复用之select、poll、epoll详解](https://my.oschina.net/xianggao/blog/663655)
- [C10K问题](https://blog.csdn.net/wangtaomtk/article/details/51811011)

> 超神
- [libevent深入浅出](https://aceld.gitbooks.io/libevent/content/)
- [libuv](http://docs.libuv.org/en/v1.x/design.html)
- [AIO](https://www.ibm.com/developerworks/library/l-async/)
- [Lazy Asynchronous I/O For Event-Driven Servers](https://www.usenix.org/legacy/event/usenix04/tech/general/full_papers/elmeleegy/elmeleegy_html/html.html)