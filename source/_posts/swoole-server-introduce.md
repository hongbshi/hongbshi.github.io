---
title: swoole_server_introduce
date: 2019-05-27 10:19:08
categories:
- swoole
- server
tags:
- server
- swoole
---
# 一. 基础知识
## 1.1 Swoole
Swoole是面向生产环境的php异步网络通信引擎, php开发人员可以利用Swoole开发出高性能的server服务。Swoole的server部分, 内容很多, 也涉及很多的知识点, 本文仅对其server进行简单的概述, 具体的实现细节在后续的文章中再进行详细介绍。

## 1.2 网络编程
1. 网络通信是指在一台(或者多台)机器上启动一个(或者多个)进程, 监听一个(或者多个)端口, 按照某种协议(可以是标准协议http, dns; 也可以是自行定义的协议)与客户端交换信息。
2. 目前的网络编程多是在tcp, udp或者更上层的协议之上进行编程。Swoole的server部分是基于tcp以及udp协议的。
3. 利用udp进行编程较为简单, 本文主要介绍tcp协议之上的网络编程
4. TCP网络编程主要涉及4种事件,
- 连接建立: 主要是指客户端发起连接(`connect`)以及服务端接受连接(`accept`)
- 消息到达: 服务端接受到客户端发送的数据, 该事件是TCP网络编程最重要的事件, 服务端对于该类事件进行处理时, 可以采用阻塞式或者非阻塞式, 除此之外, 服务端还需要考虑分包, 应用层缓冲区等问题
- 消息发送成功: 发送成功是指应用层将数据成功发送到内核的套接字发送缓冲区中, 并不是指客户端成功接受数据。对于低流量的服务而言, 数据通常一次性即可发送完, 并不需要关心此类事件。如果一次性不能将全部数据发送到内核缓冲区, 则需要关心消息是否成功发送(阻塞式编程在系统调用(`write`, `writev`, `send`等)返回后即是发送成功, 非阻塞式编程则需要考虑实际写入的数据是否与预期一致)
- 连接断开: 需要考虑客户端断开连接(`read`返回0)以及服务端断开连接(`close`, `shutdown`)

5. tcp建立连接的过程如下图,
![](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/tcp/%E5%BB%BA%E7%AB%8B%E8%BF%9E%E6%8E%A5.png)
- 图中, ACK、SYN表示标志位, seq、ack为tcp包的序号以及确认序号

6. tcp断开连接的过程如下图,
![](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/tcp/%E6%96%AD%E5%BC%80%E8%BF%9E%E6%8E%A5.png)
- 上图考虑的是客户端主动断开连接的情况, 服务端主动断开连接也类似
- 图中, FIN、ACK表示标志位, seq、ack为tcp包的序号以及确认序号

## 1.3 进程间通信
1. 进程之间的通信有无名管道(pipe), 有名管道(fifo), 信号(signal), 信号量(semaphore), 套接字(socket), 共享内存(shared memory)等方式
2. Swoole中采用unix域套接字(套接字的一种)用于多进程之间的通信(指Swoole内部进程之间)

## 1.4 socketpair
1. socketpair用于创建一个套接字对, 类似于pipe, 不同的是pipe是单向通信, 双向通信需要创建两次, socketpair调用一次即可实现双向通信, 除此之外, 由于使用的是套接字, 还可以定义数据交换的方式
2. socketpair系统调用
```cpp
int socketpair(int domain, int type, int protocol, int sv[2]);
//domain表示协议簇
//type表示类型
//protocol表示协议, SOCK_STREAM表示流协议(类似tcp), SOCK_DGRAM表示数据报协议(类似udp)
//sv用于存储建立的套接字对, 也就是两个套接字文件描述符
//成功返回0, 否则返回-1, 可以从errno获取错误信息
```
- 调用成功后sv[0], sv[1]分别存储一个文件描述符
- 向sv[0]中写入, 可以从sv[1]中读取
- 向sv[1]中写入, 可以从sv[0]中读取
- 进程调用socketpair后, fork子进程, 子进程会默认继承sv[0], sv[1]这两个文件描述符, 进而可以实现父子进程间通信。例如, 父进程向sv[0]中写入, 子进程从sv[1]中读取; 子进程向sv[1]中写入, 父进程从sv[0]中读取。

## 1.5 守护进程(daemon)
1. 守护进程是一种特殊的后台进程, 它脱离于终端, 用于周期性的执行某种任务
2. 进程组
- 每个进程都属于一个进程组
- 每个进程组都有一个进程组号, 也就是该组组长的进程号(PID)
- 一个进程只能为自己或者其子进程设置进程组号
3. 会话
- 一个会话可以包含多个进程组, 这些进程组中最多只能有一个前台进程组(也可以没有), 其余为后台进程组
- 一个会话最多只能有一个控制终端
- 用户通过终端登录或者网络登录, 会创建一个新的会话
- 进程调用系统调用`setsid`可以创建一个新的会话, 调用`setsid`的进程不能是某个进程组的组长。`setsid`调用完成后, 该进程成为这个会话的首进程(领头进程), 同时变成一个新的进程组的组长, 如果该进程之前有控制终端, 该进程与终端的联系也被断开

4. 创建守护进程的方式
- fork子进程后, 父进程退出, 子进程执行`setsid`, 子进程即可成为守护进程。这种方式下, 子进程是会话的领头进程, 可以重新打开终端, 此时可以再次fork, fork产生的子进程无法再打开终端(只有会话的领头进程才能打开终端)。第二次fork并不是必须的, 只是为了防止子进程再次打开终端
- linux提供了daemon函数(该函数并不是系统调用, 为库函数)用于创建守护进程

## 1.6 swoole tcp server示例
```php
<?php
//创建server
$serv = new Swoole\Server('0.0.0.0', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_TCP);
//设置server的参数
$serv->set(array(
    'reactor_num' => 2, //reactor thread num
    'worker_num' => 3,  //worker process num
));

//设置事件回调
$serv->on('connect', function ($serv, $fd){
    echo "Client:Connect.\n";
});
$serv->on('receive', function ($serv, $fd, $reactor_id, $data) {
    $serv->send($fd, 'Swoole: '.$data);
    $serv->close($fd);
});
$serv->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动server
$serv->start();
```
- 上述代码在cli模式下执行时, 经过词法分析, 语法分析生成opcode, 进而交由zend虚拟机执行
- zend虚拟机在执行到$serv->start()时, 启动Swoole server
- 上述代码中设置的事件回调是在worker进程中执行, 后文会详细介绍Swoole server模型

# 二. swoole server
## 2.1 base模式
1. 说明
- base模式采用多进程模型, 这种模型与nginx一致, 每个进程只有一个线程, 主进程负责管理工作进程, 工作进程负责监听端口, 接受连接, 处理请求以及关闭连接
- 多个进程同时监听端口, 会有惊群问题, linux 3.9之前版本的内核, Swoole没有解决惊群问题
- linux 内核3.9及其后续版本提供了新的套接字参数SO_REUSEPORT, 该参数允许多个进程绑定到同一个端口, 内核在接受到新的连接请求时, 会唤醒其中一个进行处理, 内核层面也会做负载均衡, 可以解决上述的惊群问题, Swoole也已经加入了这个参数
- base模式下, reactor_number参数并没有实际作用
- 如果worker进程数设置为1, 则不会fork出worker进程, 主进程直接处理请求, 这种模式适合调试

2. 启动过程
- php代码执行到$serv->start()时, 主进程进入int swServer_start(swServer *serv)函数, 该函数负责启动server
- 在函数swServer_start中会调用swReactorProcess_start, 这个函数会fork出多个worker进程
- 主进程和worker进程各自进入自己的事件循环, 处理各类事件

## 2.2 process模式
1. 说明
- 这种模式为多进程多线程, 有主进程, manager进程, worker进程, task_worker进程
- 主进程下有多个线程, 主线程负责接受连接, 之后交给react线程处理请求。 react线程负责接收数据包, 并将数据转发给worker进程进行处理, 之后处理worker进程返回的数据
- manager进程, 该进程为单线程, 主要负责管理worker进程, 类似于nginx中的主进程, 当worker进程异常退出时, manager进程负责重新fork出一个worker进程
- worker进程, 该进程为单线程, 负责具体处理请求
- task_worker进程, 用于处理比较耗时的任务, 默认不开启
- worker进程与主进程中的react线程使用域套接字进行通信, worker进程之间不进行通信

2. 启动过程
- Swoole server启动入口: swServer_start函数,
```
//php 代码中$serv->start(); 会调用函数, 进行server start
int swServer_start(swServer *serv);

// 该函数首先进行必要的参数检查
static int swServer_start_check(swServer *serv);
// 其中有,
if (serv->worker_num < serv->reactor_num)
{
    serv->reactor_num = serv->worker_num;
}//也就是说reactor_num <= worker_num

//之后执行factory start, 也就是swFactoryProcess_start函数, 该函数会fork出manager进程, manager进程进而fork出worker进程以及task_worker进程
if (factory->start(factory) < 0)
{
    return SW_ERR;
}

//然后主进程的主线程生成reactor线程
if (serv->factory_mode == SW_MODE_BASE)
{
    ret = swReactorProcess_start(serv);
}
else
{
    ret = swReactorThread_start(serv);
}
```
- 如果设置了daemon模式, 在必要的参数检查完后, 先将自己变为守护进程再fork manager进程, 进而创建reactor线程

- 主进程先fork出manager进程, manager进程负责fork出worker进程以及task_worker进程。worker进程之后进入int swWorker_loop(swServer *serv, int worker_id), 也就是进入自己的事件循环, task_worker也是一样, 进入自己的事件循环。
```
static int swFactoryProcess_start(swFactory *factory);
//swFactoryProcess_start会调用swManager_start生成manager进程
int swManager_start(swServer *serv);
// manager进程会fork出worker进程以及task_worker进程
```
- 主进程pthread_create出react线程, 主线程和react线程各自进入自己的事件循环, reactor线程执行static int swReactorThread_loop(swThreadParam *param), 等待处理事件
```
//主线程执行swReactorThread_start, 创建出reactor线程
int swReactorThread_start(swServer *serv);
```

3. 结构图
Swoole process模式结构如下图所示,
![swoole_server](https://picturestore.nos-eastchina1.126.net/swoole/swoole%20process%20%E7%BB%93%E6%9E%84%E5%9B%BE.jpg)
- 上图并没有考虑task_worker进程, 在默认情况下, task_worker进程数为0

# 三. 请求处理流程(process模式)
## 3.1 reactor线程与worker进程之间的通信
1. Swoole master进程与worker进程之间的通信如下图所示,
![image](https://picturestore.nos-eastchina1.126.net/swoole/swoole%20process%20ipc.png)
- Swoole使用SOCK_DGRAM, 而不是SOCK_STREAM, 这里是因为每个reactor线程负责处理多个请求, reactor接收到请求后会将信息转发给worker进程, 由worker进程负责处理,如果使用SOCK_STREAM, worker进程无法对tcp进行分包, 进而处理请求
- swFactoryProcess_start函数中会根据worker进程数创建对应个数的套接字对, 用于reactor线程与worker进程通信(详见swPipeUnsock_create函数)

2. 假设reactor线程有2个, worker进程有3个, 则reactor与worker之间的通信如下图所示,
![image](https://picturestore.nos-eastchina1.126.net/swoole/swoole%20process%20ipc%E5%AE%9E%E4%BE%8B.png)
- 每个reactor线程负责监听几个worker进程, 每个worker进程只有一个reactor线程监听(reactor_num <= worker_num)。Swoole默认使用worker_process_id % reactor_num对worker进程进行分配, 交给对应的reactor线程进行监听
- reactor线程收到某个worker进程的数据后会进行处理, 值得注意的是, 这个reactor线程可能并不是发送请求的那个reactor线程。

3. reactor线程与worker进程通信的数据包
```c
//包头
typedef struct _swDataHead
{
    int fd;
    uint32_t len;
    int16_t from_id;
    uint8_t type;
    uint8_t flags;
    uint16_t from_fd;
#ifdef SW_BUFFER_RECV_TIME
    double time;
#endif
} swDataHead;

//reactor线程向worker进程发送的数据, 也就是worker进程收到的数据包
typedef struct
{
    swDataHead info;
    char data[SW_IPC_BUFFER_SIZE];
} swEventData;

//worker进程向reactor线程发送的数据, 也就是reactor线程收到的数据包
typedef struct
{
    swDataHead info;
    char data[0];
} swPipeBuffer;
```

## 3.2 请求处理
1. master进程中的主线程负责监听端口(`listen`), 接受连接(`accept`, 产生一个fd), 接受连接后将请求分配给reactor线程, 默认通过fd % reactor_number进行分配, 之后通过`epoll_ctl`将fd加入到对应reactor线程中, 刚加入时监听写事件, 因为新接受连接创建的套接字写缓冲区为空, 故而一定可写, 会被立刻触发, 进而reactor线程进行一些初始化操作
- 存在多个线程同时操作一个epollfd(通过系统调用`epoll_create`创建)的情况
- 多个线程同时调用`epoll_ctl`是线程安全的(对应一个epollfd), 一个线程正在执行, 其他线程会被阻塞(因为需要同时操作epoll底层的红黑树)
- 多个线程同时调用`epoll_wait`也是线程安全的, 但是一个事件可能会被多个线程同时接收到, 实际中不建议多个线程同时`epoll_wait`一个epollfd。Swoole中也是不存在这种情况的, Swoole中每个reactor线程都有自己的epollfd
- 一个线程调用`epoll_wait`, 一个线程调用`epoll_ctl`, 根据man手册, 如果`epoll_ctl`新加入的fd已经准备好, 会使得执行`epoll_wait`的线程变成非阻塞状态(可以通过man `epoll_wait`查看相关内容)。
```
//主线程, 如果fd已经ready, reactor线程会被唤醒
epoll_ctl(epollfd, fd, ...);
//reactor线程
epoll_wait(epollfd,...)
```

2. reactor线程中fd的写事件被触发, reactor线程负责处理, 发现是首次加入, 没有数据可写, 则会开启读事件监听, 准备接受客户端发送的数据

3. reactor线程读取到用户的请求数据, **一个请求的数据接收完后**, 将数据转发给worker进程, 默认是通过fd % worker_number进行分配
- reactor发送给worker进程的数据包, 会包含一个头部, 头部中记录了reactor的信息
- 如果发送的数据过大, 则需要将数据进行分片, 限于篇幅, 数据分片, 后续再进行详细讲述
- 可能存在多个reactor线程同时向同一个worker进程发送数据的情况, 故而Swoole采用SOCK_DGRAM模式与worker进程进行通信, 通过每个数据包的包头, worker进程可以区分出是由哪个reactor线程发送的数据, 也可以知道是哪个请求

4. worker进程收到reactor发送的数据包后, 进行处理, 处理完成后, 将请求结果发送给主进程
- worker进程发送给主进程的数据包, 也会包含一个头部, 当reactor线程收到数据包后, 能够知道对应的reactor线程, 请求的fd等信息

5. 主进程收到worker进程发送的数据包, 这个会触发某个reactor线程进行处理
- **这个reactor线程并不一定是之前发送请求给worker进程的那个reactor线程**
- 主进程的每个reactor线程都负责监听worker进程发送的数据包, 每个worker发送的数据包只会由一个reactor线程进行监听, 故而只会触发一个reactor线程

6. reactor线程处理worker进程发送的请求处理结果, 如果是直接发送数据给客户端, 则可以直接发送, 如果需要改变这个这个连接的监听状态(例如`close`), 则需要先找到监听这个连接的reactor线程, 进而改变这个连接的监听状态(通过调用`epoll_ctl`)
- reactor处理线程与reactor监听线程可能并不是同一个线程
- reactor监听线程负责监听客户端发送的数据, 进而转发给worker进程
- reactor处理线程负责监听worker进程发送给主进程的数据, 进而将数据发送给客户端

# 四. gdb调试
## 4.1 process模式启动
```
//fork manager进程
#0  0x00007ffff67dae64 in fork () from /lib64/libc.so.6
#1  0x00007ffff553888a in swoole_fork () at /root/code/swoole-src/src/core/base.c:186
#2  0x00007ffff556afb8 in swManager_start (serv=serv@entry=0x1353f60) at /root/code/swoole-src/src/server/manager.cc:164
#3  0x00007ffff5571dde in swFactoryProcess_start (factory=0x1353ff8) at /root/code/swoole-src/src/server/process.c:198
#4  0x00007ffff556ef8b in swServer_start (serv=0x1353f60) at /root/code/swoole-src/src/server/master.cc:651
#5  0x00007ffff55dc808 in zim_swoole_server_start (execute_data=<optimized out>, return_value=0x7fffffffac50)
    at /root/code/swoole-src/swoole_server.cc:2946
#6  0x00000000007bb068 in ZEND_DO_FCALL_SPEC_RETVAL_UNUSED_HANDLER () at /root/php-7.3.3/Zend/zend_vm_execute.h:980
#7  execute_ex (ex=0x7ffff7f850a8) at /root/php-7.3.3/Zend/zend_vm_execute.h:55485
#8  0x00000000007bbf58 in zend_execute (op_array=op_array@entry=0x7ffff5e7b340, return_value=return_value@entry=0x7ffff5e1d030)
    at /root/php-7.3.3/Zend/zend_vm_execute.h:60881
#9  0x0000000000737554 in zend_execute_scripts (type=type@entry=8, retval=0x7ffff5e1d030, retval@entry=0x0,
    file_count=file_count@entry=3) at /root/php-7.3.3/Zend/zend.c:1568
#10 0x00000000006db4d0 in php_execute_script (primary_file=primary_file@entry=0x7fffffffd050) at /root/php-7.3.3/main/main.c:2630
#11 0x00000000007be2f5 in do_cli (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:997
#12 0x000000000043fc1f in main (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:1389


// pthread_create reactor线程
#0  0x00007ffff552e960 in pthread_create@plt () from /usr/local/lib/php/extensions/no-debug-non-zts-20180731/swoole.so
#1  0x00007ffff5576959 in swReactorThread_start (serv=0x1353f60) at /root/code/swoole-src/src/server/reactor_thread.c:883
#2  0x00007ffff556f006 in swServer_start (serv=0x1353f60) at /root/code/swoole-src/src/server/master.cc:670
#3  0x00007ffff55dc808 in zim_swoole_server_start (execute_data=<optimized out>, return_value=0x7fffffffac50)
    at /root/code/swoole-src/swoole_server.cc:2946
#4  0x00000000007bb068 in ZEND_DO_FCALL_SPEC_RETVAL_UNUSED_HANDLER () at /root/php-7.3.3/Zend/zend_vm_execute.h:980
#5  execute_ex (ex=0x7fffffffab10) at /root/php-7.3.3/Zend/zend_vm_execute.h:55485
#6  0x00000000007bbf58 in zend_execute (op_array=op_array@entry=0x7ffff5e7b340, return_value=return_value@entry=0x7ffff5e1d030)
    at /root/php-7.3.3/Zend/zend_vm_execute.h:60881
#7  0x0000000000737554 in zend_execute_scripts (type=type@entry=8, retval=0x7ffff5e1d030, retval@entry=0x0,
    file_count=file_count@entry=3) at /root/php-7.3.3/Zend/zend.c:1568
#8  0x00000000006db4d0 in php_execute_script (primary_file=primary_file@entry=0x7fffffffd050) at /root/php-7.3.3/main/main.c:2630
#9  0x00000000007be2f5 in do_cli (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:997
#10 0x000000000043fc1f in main (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:1389
```

## 4.2 base模式启动
```
//base 模式下的启动
#0  0x00007ffff67dae64 in fork () from /lib64/libc.so.6
#1  0x00007ffff553888a in swoole_fork () at /root/code/swoole-src/src/core/base.c:186
#2  0x00007ffff5558557 in swProcessPool_spawn (pool=pool@entry=0x7ffff2d2a308, worker=0x7ffff2d2a778)
    at /root/code/swoole-src/src/network/process_pool.c:392
#3  0x00007ffff5558710 in swProcessPool_start (pool=0x7ffff2d2a308) at /root/code/swoole-src/src/network/process_pool.c:227
#4  0x00007ffff55741cf in swReactorProcess_start (serv=0x1353f60) at /root/code/swoole-src/src/server/reactor_process.cc:176
#5  0x00007ffff556f21d in swServer_start (serv=0x1353f60) at /root/code/swoole-src/src/server/master.cc:666
#6  0x00007ffff55dc808 in zim_swoole_server_start (execute_data=<optimized out>, return_value=0x7fffffffac50)
    at /root/code/swoole-src/swoole_server.cc:2946
#7  0x00000000007bb068 in ZEND_DO_FCALL_SPEC_RETVAL_UNUSED_HANDLER () at /root/php-7.3.3/Zend/zend_vm_execute.h:980
#8  execute_ex (ex=0x7ffff2d2a308) at /root/php-7.3.3/Zend/zend_vm_execute.h:55485
#9  0x00000000007bbf58 in zend_execute (op_array=op_array@entry=0x7ffff5e7b340, return_value=return_value@entry=0x7ffff5e1d030)
    at /root/php-7.3.3/Zend/zend_vm_execute.h:60881
#10 0x0000000000737554 in zend_execute_scripts (type=type@entry=8, retval=0x7ffff5e1d030, retval@entry=0x0,
    file_count=file_count@entry=3) at /root/php-7.3.3/Zend/zend.c:1568
#11 0x00000000006db4d0 in php_execute_script (primary_file=primary_file@entry=0x7fffffffd050) at /root/php-7.3.3/main/main.c:2630
#12 0x00000000007be2f5 in do_cli (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:997
#13 0x000000000043fc1f in main (argc=2, argv=0x1165cd0) at /root/php-7.3.3/sapi/cli/php_cli.c:1389
```

# 五. 总结及思考
1. 本文主要介绍了Swoole server的两种模式: base模式、process模式, 详细讲解了两种模式的网络编程模型, 并重点介绍了process模式下, 进程间通信的方式、请求的处理流程等。
2. process模式下, 为什么不直接在主进程中创建多个线程, 由线程直接进行处理请求(可以避免进程间通信的开销), 而是创建出manager进程, 再由manager进程创建出worker进程, 由worker进程处理请求?
- 个人觉得可能是php对多线程的支持不是很友好, phper大都也只是进行单线程编程
- ZendVM 提供的 TSRM 虽然也是支持多线程环境，但实际上这是一个 按线程隔离内存的方案, 多线程并没有意义
3. process模式下, 主进程中的每个reactor线程都可以同时处理多个请求, 多个请求是并发处理的, 我们从2个维度看,
- 从主进程的角度看, 主进程同时处理多个请求, 当一个请求包全部接收完后, 转发给worker进程进行处理
- 从某个worker进程的角度看, 这个worker进程收到的请求是串行的, 默认情况下, worker进程也是串行处理请求, 如果单个请求阻塞(Swoole的worker进程会回调phper写的事件处理函数, 该函数可能阻塞), 后续的请求也无法处理, 这个就是排头阻塞问题, 这种情况下可以使用Swoole的协程, 通过协程的调度, 单个请求阻塞时, worker进程可以继续处理其他请求。
4. 使用Swoole创建tcp server时, 由于tcp是字节流的协议, 需要分包, 而Swoole在不清楚客户端与服务端通信协议的情况下, 无法进行分包, process模式下, reactor交给worker进程的数据也只能是字节流的, 需要用户自行处理。当然, 一般情况也不需要自行构建协议, 使用tcp server, Swoole已经支持Http, Https等协议。

# 六. 参考
- UNIX网络编程
- UNIX环境高级编程
- php7 底层实现与源码分析
- https://wiki.swoole.com/
- https://www.cnblogs.com/welhzh/p/3772164.html
- https://www.cnblogs.com/JohnABC/p/4079669.html
