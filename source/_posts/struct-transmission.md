---
title: 结构体传输
date: 2018-11-05 20:10:41
categories:
- 网络编程
- 基础
tags:
- 网络编程
---

# 一. 基础
1. 结构体传输基本上有两种方式,序列化(Json,Xml等)以及直接传输结构体。
2. 下面考虑32位系统，直接发送结构体进行传输。

# 二. 结构体
```cpp
struct Data{
    char v1;
    int v2;
    char v3;
}
```
# 三. 源码
## 3.1 发送方
```cpp
//将data写入buf
bool send(char *buf, int bufLen, const Data *data){
    if(bufLen < sizeof(*data)) return false;
    memcpy(buf, data, sizeof(*data);
    return true;
}

//main function
int main(int argc, char **argv){
    char sendBuf[256] = {};
    Data data;
    data.v1 = 'a';
    data.v2 = htonl(2);
    data.v3 = 'b';
    $ans = send(sendBuf, sizeof(sendBuf), &data);
    if($ans == false) return -1;
    //send data
}
```
## 3.2 接收方
```cpp
//deal data
bool parseData(char *buf, int bufLen, Data *data){
    if(bufLen < sizeof(*data)) return false;
    memecpy(buf, data, sizeof(*data));
    data.v2 = ntohl(data.v2);
    return true;
}

//main
int main(int argc, char **argv){
    char recvBuf[256] = {};
    //read socket recv data
    //deal data
    Data data;
    $ans = parseData(buf, sizeof(recvBuf), &data);
    if($ans == false) return -1;
    //Deal data
}
```
## 3.3 测试
```cpp
#include <iostream>
#include <string>
#include <arpa/inet.h>

using namespace std;

struct Data{
    char v1;
    int v2;
    char v3;
};

void parse(char *buf, Data *data){
    memcpy(data, buf, sizeof(*data));
    data->v2 = ntohl(data->v2);
}

int main (int argc, char **argv){
    Data data;
    data.v1 = 'a';
    data.v2 = htonl(2);
    data.v3 = 'b';
    char buf[128] = {};
    memcpy(buf, &data, sizeof(data));
    Data newData;
    parse(buf, &newData);
    std::cout << newData.v1 << ' ' << newData.v2 << ' ' << newData.v3 << std::endl;
    return 0;
}
//输出: a, 2, b
```

# 四. 注意事项
1. sizeof(data) = 12;
2. 结构体要考虑对齐,发送方与接收方的对齐方式应该是一致的。
3. 发送方按照网络字节序存储,接收方得到网络字节序的数据后，解析成本机字节序。
