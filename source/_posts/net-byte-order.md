---
title: 字节序与位序
date: 2018-11-05 19:52:07
categories:
- 网络编程
- 基础
tags:
- 网络编程
- 数据传输顺序
---

# 一. 本机字节序
1. 小端: 低位字节存在低地址。
2. 大端: 低位字节存在高地址。

# 二. 本机位序
一般情况下，本机位序与本机的字节序一致。
1. 小端字节序: 低位bit存在低地址。
2. 大端字节序: 低位bit存在高地址。

# 三. 网络序
1. 网络字节序(大端)，先传送高位字节，再传送低位字节。
2. 在传输一个字节时，先传送低位bit, 再传送高位bit。

注: 指针指向变量或者数组的起始地址，即指向低地址。

# 四. 代码示例
```cpp
#include <iostream>
#include <string>

using namespace std;

int main (int argc, char **argv){
    union byte_order{
        int a;
        unsigned char b[4];
    };
    byte_order val;
    val.a = 0x01020304;
    printf("address 0x%x byte: 0x%x\n", &val.b[0], val.b[0]);
    printf("address 0x%x byte: 0x%x\n", &val.b[1], val.b[1]);
    printf("address 0x%x byte: 0x%x\n", &val.b[2], val.b[2]);
    printf("address 0x%x byte: 0x%x\n", &val.b[3], val.b[3]);

    struct  bit_order{
        unsigned char a:4;
        unsigned char b:4;
    };
    unsigned char tmp = 0x04;
    bit_order *val1 = (bit_order*)&tmp;
    printf("low bit %d\n", val1->a);
    printf("high bit %d\n", val1->b);
}

//输出:
address 0x56074a68 byte: 0x4
address 0x56074a69 byte: 0x3
address 0x56074a6a byte: 0x2
address 0x56074a6b byte: 0x1
low bit 4
high bit 0
```
