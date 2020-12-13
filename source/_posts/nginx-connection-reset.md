---
title: Nginx Connection Reset 问题排查
date: 2020-12-13 18:58:14
categories:
- nginx
tags:
- Nginx
- Tcp
---

## 一. 背景介绍
### 1.1 业务背景
网校服务正在向`K8S`迁移，我们有两个服务之前是绑定到一台机器上部署的，二者之间通过`IP`直接访问，如下图所示，

![image](https://picturestore.nos-eastchina1.126.net/%E8%BF%9E%E6%8E%A5%E9%87%8D%E7%BD%AE/%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.jpg)

调用关系非常简单，`服务A`调用了`服务B`，这里简单说明下`服务A`和`服务B`，
- `服务A`基于`Golang`的`Gin`框架开发，使用`Http`长连接访问`服务B`
- `服务B`基于`C++`的`BRPC`开发

我们想把两个服务进行拆分，通过域名访问。拆分后，访问链路变成了下图，

![image](https://picturestore.nos-eastchina1.126.net/%E8%BF%9E%E6%8E%A5%E9%87%8D%E7%BD%AE/%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A82.jpg)

在拆分之后，我们发现`服务A`出现了大量的`connection reset by peer`的错误，而且这些错误基本都是集中出现，出现的时间点也没有什么规律，本文是对排查过程的简单总结。

### 1.2 Tcp Reset简介
`Tcp`发送`Reset`包有很多种情况，比如说：服务端的全连接队列已满，无法接受新的连接请求；服务端已经关闭连接，客户端仍然向其发送数据；服务端没有处理完客户端发送的所有数据。还有很多其他的情况，我们这里就不再一一列举。本文主要介绍其中的2种。

1. 服务端已经关闭连接，客户端仍然发送数据，这种情况比较容易模拟，也比较容易理解。

2. 我们这里介绍下，服务端没有处理完客户端数据的情况。对于客户端发送的数据, 服务端应用层没有读取完, 就关闭了连接, 服务端会发送`Reset`。这里是为什么呢？思考之后，不难发现， 这是`Tcp`可靠性的保证, `Tcp`需要保证客户端发送的数据, 服务端应用层都能收到，如果服务端应用层没有读取数据，就应该通知客户端，怎么通知呢，就是通过`Tcp Reset`包。下面给出一个简单的示例程序，

```cpp
#include <sys/time.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8211

int main(int argc, char** argv)
{
    int send_sk = socket(AF_INET, SOCK_STREAM, 0);
    if(send_sk == -1)
    {
        perror("socket failed  ");
        return -1;
    }

    struct sockaddr_in s_addr;
    socklen_t len = sizeof(s_addr);
    bzero(&s_addr, sizeof(s_addr));
    s_addr.sin_family = AF_INET;
    inet_pton(AF_INET, "127.0.0.1", &s_addr.sin_addr);
    s_addr.sin_port = htons(PORT);

    if(connect(send_sk, (struct sockaddr*)&s_addr, len) == -1)
    {
        perror("connect fail  ");
        return -1;
    }

    char pcContent[1028]={0};
    write(send_sk, pcContent, 1028);

    sleep(1);
    close(send_sk);
    return 0;
}
```
- 编译`gcc client.c -o client`
- 以上是客户端程序, 客户端发送了1028个字节给服务端，等待1s之后，关闭连接。

```cpp
#include <sys/time.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8211
#define BACKLOG 10
#define MAXRECVLEN 1024

int main(int argc, char *argv[])
{
    char buf[MAXRECVLEN];
    int listenfd, connectfd;   /* socket descriptors */
    struct sockaddr_in server; /* server's address information */
    struct sockaddr_in client; /* client's address information */
    socklen_t addrlen;
    
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
    {
        perror("socket() error. Failed to initiate a socket");
        exit(1);
    }

    /* set socket option */
    int opt = SO_REUSEADDR;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    bzero(&server, sizeof(server));
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr.s_addr = htonl(INADDR_ANY);
    if(bind(listenfd, (struct sockaddr *)&server, sizeof(server)) == -1)
    {
        perror("Bind() error.");
        exit(1);
    }

    if(listen(listenfd, BACKLOG) == -1)
    {
        perror("listen() error. \n");
        exit(1);
    }

    addrlen = sizeof(client);
	printf("wait connect\n");
    if((connectfd=accept(listenfd,(struct sockaddr *)&client, &addrlen))==-1)
    {
     perror("accept() error. \n");
     exit(1);
    }
    printf("connectfd is %d\n", connectfd);
    int ans = recv(connectfd, buf, MAXRECVLEN, 0);
    printf("read data size: %d\n", ans);
    close(listenfd); /* close listenfd */
    return 0;
}
```
- 编译`gcc server.c -o server`
- 服务端启动后, 等待客户端的连接。客户端连接之后，服务端只接收了1024个字节, 就关闭了连接。

### 1.3 Nginx长连接的一些设置参数
这里给出网关`Nginx`的一些配置，网关`Nginx`是`1.15.8`版本，这里只列了其中一部分配置，
```
http{
	upstream backend{
		keepalive 100;
		#Nginx 1.15.3之后可以设置，这里是默认配置
		keepalive_timeout 60s;
		#Nginx 1.15.3之后可以设置，这里是默认配置
		keepalive_requests 100;
		...
	}
	server{
		keepalive_timeout 20s;
		keepalive_requests 100;
		...
	}
}
```
- `server`中的`keepalive_timeout`, 意思是`Nginx`作为服务端，对于客户端的长连接请求，如果20s内，没有收到新的请求，就会关闭这个连接。
- `server`中的`keepalive_requests`，意思是对于客户端的单个长连接，最多处理`100`个请求，处理完`100`个请求后，就关闭这个连接，不再接收新的请求。
- `upstream`中的`keepalive`，意思是这个`upstream`最多的空闲长连接数
- `upstream`中的`keepalive_timeout`，意思是`Nginx`作为客户端，与`upstream`建立长连接后，如果`60s`内没有使用，就关闭这个长连接，不再使用。
- `upstream`中的`keepalive_requests`，意思是`Nginx`作为客户端，与`upstream`建立的长连接，单个连接最多发送`100`个请求，超过之后，就关闭连接。

### 1.4 Golang服务的长连接设置
`Golang`服务在访问网关`Nginx`时，充当客户端的角色，作为客户端的配置如下，我们这里只列出其中一部分，
```go
//
transport = http.Transport{}
transport.MaxIdleConns = 100
transport.MaxIdleConnsPerHost = 100
transport.IdleConnTimeout = 60 * time.Second
//
client = http.Client{Timeout: 300 * time.Millisecond}
client.Transport = &transport
```
这里重点讲下`transport.IdleConnTimeout`的含义，它的意思是单个长连接，如果`60s`内没有被使用，就不再使用这个长连接发送请求。

## 二. 问题排查

服务拆分之后，`Golang`服务出现了大量的`connection reset by peer`错误，很明显，这个错误是网关`Nginx`发送给`Golang`服务的。问题的排查可以分为3个阶段，这里我们一一介绍。

### 2.1 超时设置
之前一直认为，网关`Nginx server`中的`keepalive_timeout`设置的是`75s`，也就是说`Nginx`过了`75s`才关闭这个长连接。问题排查的时候，问了下网关那边的具体配置，告诉我是`20s`。这时候就想，应该是因为我们`Golang`服务设置的超时时间是`60s`导致的。当单个长连接超过`20s`没有被使用后，`Golang`服务认为这个长连接还可以使用，但是网关`Nginx`认为已经超时，所以`Golang`服务再次发送请求，`Nginx`会发送`Tcp Reset`包。

将`transport.IdleConnTimeout`设置为`15s`后， 本以为这个问题一定能解决，但是发现还是会出现`connection reset by peer`。

### 2.2 长连接处理请求数设置

修改了超时时间之后，发现并没有解决`connection reset by peer`的问题。之后，又想到是不是`Nginx server`中的`keepalive_requests`设置导致的。`Golang`服务的长连接，并没有设置单个长连接可以发送的请求数，但是`Nginx`在单个长连接上，只会处理`100`个请求，超过`100`请求后，`Nginx`会关闭连接，这时候如果`Golang`服务发送了请求，`Nginx`就会发送`Tcp Reset`包。

之后，想要设置`Golang Http Client`的长连接处理请求数，找了半天，并没有找到哪个配置项可以配置这个值（这里如果有知道的，可以和我说下）。

没找到配置项后，就想先抓包看下吧，抓到包之后就发现了问题不在这里。

### 2.3 找到原因并解决

想要设置`Golang`服务作为客户端，单个长连接最多处理的请求数无果后，进行了抓包，抓包结果显示，
- 单个长连接并没有处理到`100`个请求，`Nginx`就发送了`Tcp Reset`包。
- 收到`Tcp Reset`包的时间点，还有很多`Tcp Fin`包。

之后，排查网关日志发现，我们出现`connection reset by peer`错误的时候，网关也出现了大量的`init_worker()`的错误。这时候就感觉是网关`Nginx Reload`导致的`connection reset by peer`。

然后，我们复现了这个现象，晚上的时候，让网关`Nginx`更改配置，`Reload`服务，发现我们果然又出现了`connection reset by peer`的错误，到这里基本上就定位了问题原因。

我们思考下，为什么网关`Nginx`重启，会导致我们的服务收到`Tcp Reset`包，不是说`Nginx`是平滑重启，不影响服务的嘛？思考之后，原因如下，
- `Nginx Reload`在创建新的`Worker`进程之后，需要向老的`Worker`发送信号，要求其不再接收新的连接请求，并处理完当前连接的请求后，关闭连接。
- 很明显，我们`Golang`服务是与`Nginx`老的`Worker`建立的长连接，当老的`Worker`处理完一个请求后，发送结果给`Golang`服务，`Golang`服务收到结果之后，可能会继续发送请求，但是这个时候，`Nginx`老的`Worker`可能已经关闭了连接，故而发送`Tcp Reset`包给`Golang`服务。
- 这里留个问题，有没有某种情况是，`Nginx Worker`收到请求后，没有处理就关闭了连接，进而导致服务端`Nginx`发送`Tcp Reset`包给客户端？

找到原因之后，就需要解决这个问题。解决方案有两种，
- `Golang`服务的长连接改为短连接，我们使用的就是这一种
- `Golang`服务继续使用长连接，当出现`connection reset by peer`错误后，进行重试。这里需要注意的是，因为`Nginx`正在`Reload`，大量的长连接都会失效，所以可能需要重试很多次。

## 三. 总结

`Nginx`是非常优秀的负载均衡、反向代理工具，业界使用的非常广泛，`Nginx`的高性能、模块化设计也被很多软件所模仿。我们在使用`Nginx`的时候，可以多思考，多总结，加深对`Nginx`的理解。`Http`的长连接和短连接各有利弊，我们要根据自己的实际场景进行选择。排查问题的时候，多思考，任何一个问题背后都可能有很丰富的知识点。
