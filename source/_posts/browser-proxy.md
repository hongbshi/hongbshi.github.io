---
title: browser_proxy
date: 2018-11-09 11:38:34
categories:
- 网络编程
- 代理
tags:
- 网络
- 代理
---
# 一. 概述

1. 什么是正向代理?
> - 简单来说就是代理客户端请求的服务器。例如, 浏览器中设置代理翻墙等。

2. 正向代理的主要问题?
> - 代理服务器需要知道客户的目标服务器, 例如: 客户一会请求www.baidu.com, 一会请求www.taobao.com, 代理服务器如何获取目标服务器信息。

3. 正向代理问题的解决方案?
> - 客户端按照某种事先约定的协议通知代理服务器, 例如sock5协议
> - 代理服务器能够直接从客户端与服务端交互的上层数据包(tcp之上)获取目标服务器信息。例如: 客户端与目标服务器按照http协议通信, 代理服务器可以直接从数据包中解析目标服务器。这种情况不适用与客户端与目标服务器加密通信的情况, 例如https。


# 二. socks5
## 2.1 socks5基本流程
sock5通信的基本流程图如下,
![image](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/sock5/socks5%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B.png)

1. 如果不需要认证, 第三步和第四步则不需要。


## 2.2 socks5总结
1. socks5主要完成目标地址传递的功能
> - 浏览器与sock5服务器完成tcp握手
> - 浏览器将目标地址通过socks5协议发送给socks5服务器
> - socks5服务器与目标地址完成tcp握手
> - 浏览器将请求数据发送给socks5服务器, 由其进行转发, 此时可以只做4层tcp代理


# 三. shadowsocks原理
shadowsocks信息交换的基本结构图如下, ss_cli代表shadowsocks客户端, ss_srv代表shadowsocks服务端,
![image](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/shadowsocks/shadowsocks%E4%BB%A3%E7%90%86%E7%BB%93%E6%9E%84%E5%9B%BE.png)

## 3.1 第一阶段
1. 客户端与ss_cli建立tcp连接
2. 客户端与ss_cli协商sock认证参数, ss_cli给与响应, 默认不进行认证
3. 客户端将目标地址发送给ss_cli, ss_cli给与响应, 默认返回成功

## 3.2 第二阶段
4. ss_cli与ss_srv建立tcp连接
5. ss_cli将客户端发送的目标地址发送给ss_srv, 此时按照ss_cli与ss_srv协商好的数据包格式, 并不需要使用sock协议, 同时还会发送加密解密需要的随机数
6. ss_srv与目标地址建立tcp连接

## 3.3 第三阶段
7. 客户端向ss_cli发送tcp数据包
8. ss_cli将数据包稍作处理, 计算长度, 计算哈希值, 并将这些信息一并发送给ss_srv, 此时的数据包格式是ss_cli与ss_srv约定好的
9. ss_srv受到数据后, 解析后发送给目标服务器

## 3.4 第四阶段
10. ss_srv收到目标服务器返回的数据后, 将数据按约定好的格式转发给ss_cli
11. ss_cli受到数据后, 解析转发给客户

## 3.5 总结
1. 通过sock5协议, ss_cli可以知道客户端的目标服务器地址
2. ss_cli与ss_srv可以自行定义协议, 核心是需要将客户端的目标服务器地址以及解密需要的随机变量发送给ss_srv
3. ss_cli, ss_srv不需要也无法知道客户端与目标服务器的具体通信信息。
4. ss_cli, ss_srv后期是tcp的代理, 不管是https还是http对其而言都是一样的。


## 3.6 参考
- https://www.jianshu.com/p/cbea16a096fb


# 四. 抓包分析
1. ss_cli: 127.0.0.1:1080
2. ss_srv: 204.48.26.173:21500
3. 浏览器与ss_cli同属于一个主机, 通信时延低。ss_cli与ss_srv需要通过网络传输, 时延高。数据包分析中可以利用这一点。


## 4.1 浏览器与ss_cli通信
1. tcp握手, 浏览器通过socks5发送目标服务器地址
![image](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/shadowsocks/browser-ss_cli-1.png)

2. 浏览器与ss_cli之间后续的通信, 此处是https协议
![image](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/shadowsocks/browser-ss_cli-2.png)
> - 时间突变的部分是由于ss_cli需要ss_srv将目标服务器的应答信息发送回来。

## 4.2 ss_cli与ss_srv通信
二者的通信格式是自行定义的,
![image](https://picturestore.nos-eastchina1.126.net/%E7%BD%91%E7%BB%9C/shadowsocks/ss_cli-ss_srv.png)
