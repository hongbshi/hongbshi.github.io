---
title: 浅谈C/C++链接库
date: 2019-08-28 16:34:08
categories:
- C/C++
- 链接库
tags:
- 编译
- 链接
- 动态链接库
- 静态链接库
---

## 一. 说明
1. 本文后续代码的编译以及执行环境为Centos 7.6 x86_64, g++ 4.8.5
2. 本文后续会用到linux下nm, ldd命令。nm用于查看文件中的符号, 例如变量, 函数名称。ldd用于查看动态链接库或者可执行文件的依赖库(动态链接库)。


## 二. 编译链接
1. 程序员写出的代码为.c或者.cpp, 这些文件需要经过: **预处理(处理代码中的include, 宏等)、编译(生成汇编代码)、汇编(将汇编代码生成二进制文件)、链接**才能生成可执行程序。**本文将预处理、编译、汇编的过程都看做是编译, 简化读者理解。更多细节可以参考相关资料**。
2. 生成可执行文件后, 通过终端进行执行
3. g++参数说明,
- -std=`c++11`: 使用c++11标准
- -o: 指定输出文件名称
4. 链接器ld参数:
- -L: 指定链接时搜索的动态链接库路径
- -l: 链接某个库, 例如链接libmath.so, 写为-lmath


### 2.1 编译
1. 对于c或者`c++`项目而言, 我们认为单个c或者cpp文件是一个编译单元, 通过编译器(gcc, g++, clang, clang++)可以生成编译后的**二进制文件**。例如: 编译file1.cpp, 可以生成file1.o。对于单个编译单元而言, 里面会有一些符号, 例如函数名称, 变量名称, 类名。这些符号可以分为三类: 
- 对外提供的, 也就是说其他的编译单元可以使用的
- 对外依赖的, 也就是说本单元需要外部的其他编译单元提供的符号
- 自己内部使用的, 这种符号只有本编译单元自身需要使用, 外部不可见
2. 通过nm, 我们可以查看某个编译单元存在哪些符号


### 2.2 链接
1. C/C++项目中含有很多个c文件或者cpp文件, 这些文件经过编译生成了对应的二进制文件。需要通过链接器将这些文件链接, 进而生成可执行程序。
2. linux下链接器为ld, 利用该工具我们可以将这些文件链接, 进而生成可执行程序。
3. **在进行链接时, 每个编译单元需要的符号, 都需要能够找到对应的定义。例如: 某个编译单元需要其他编译单元提供符号fun1, 这是一个函数, 如果链接器没能从其他编译单元找到这个符号, 就会报我们经常看到的未定义错误。若果出现多次, 则会报出重复定义的错误。**

### 2.3 示例
1. math.h
```cpp
#ifndef _MATH_H_
#define _MATH_H_

int add(int a, int b);

#endif
```
2. math.cpp
```cpp
#include "math.h"

int add(int a, int b){
    return a + b;
}
```
3. main.cpp
```cpp
#include <iostream>

#include "math.h"

using namespace std;

int main(int argc, char **argv){
    int a = 100, b = 200;
    int result = add(a, b);
    cout << result << endl;
}
```

4. 生成可执行文件
- 编译math.cpp: g++ -std=c++11 -c math.cpp, 生成math.o
- 编译main.cpp: g++ -std=c++11 -c main.cpp, 生成main.o
- 生成可以执行的文件: g++ -v math.o main.o -o main, 可以看到g++的编译链接过程

```
Using built-in specs.
COLLECT_GCC=g++
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC)
COMPILER_PATH=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/:/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/:/usr/libexec/gcc/x86_64-redhat-linux/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/:/usr/lib/gcc/x86_64-redhat-linux/
LIBRARY_PATH=/usr/lib/gcc/x86_64-redhat-linux/4.8.5/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64/:/lib/../lib64/:/usr/lib/../lib64/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-v' '-o' 'main' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/collect2 --build-id --no-add-needed --eh-frame-hdr --hash-style=gnu -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o main /usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64/crt1.o /usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64/crti.o /usr/lib/gcc/x86_64-redhat-linux/4.8.5/crtbegin.o -L/usr/lib/gcc/x86_64-redhat-linux/4.8.5 -L/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64 -L/lib/../lib64 -L/usr/lib/../lib64 -L/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../.. main.o math.o -lstdc++ -lm -lgcc_s -lgcc -lc -lgcc_s -lgcc /usr/lib/gcc/x86_64-redhat-linux/4.8.5/crtend.o /usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64/crtn.o
```
- **其中最后一行调用collect2(对ld进行了包装)会执行真正的链接操作, 我们直接调用这一句也可以生成main可执行文件**
- 可以看出linux下的链接操作比较复杂, 不是简单的ld main.o math.o即可成功的。


## 三. 问题
通过上面的介绍, 我们知道一个c/cpp文件通过编译链接, 最终生成可执行文件。无论任何语言, 程序员在写代码时, 都不可避免需要使用到库, 本文主要介绍C/C++中的库, 总体而言, 我们将这些库分为静态链接库(通常以.a结尾)，动态链接库(通常以.so结尾)。首先我们来看几个问题:
1. 什么是静态链接库?什么是动态链接库?
2. 静态链接库如何生成?动态链接库如何生成?
3. 静态链接库是否可以依赖其他的静态链接库? 是否可以依赖其他动态链接库?
4. 动态链接库是否可以依赖其他的静态链接库? 是否可以依赖其他的动态链接库?
5. 链接静态库时?其依赖的库该如何链接?
6. 链接动态库时?其依赖的库该如何链接?
7. 使用第三方库时, 使用静态链接库还是动态链接库?


## 四. Hello World
本节以hello world为例,

```cpp
#include <iostream>
using namespace std;

int main(int argc, char **argv){
    cout << "hello world" << endl;
}
```
1. 编译程序: g++ -std=c++11 -o main main.cpp
2. 使用ldd查看main的依赖: ldd main

```
linux-vdso.so.1 =>  (0x00007ffcf53fa000)
libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f7828b3b000)
libm.so.6 => /lib64/libm.so.6 (0x00007f7828839000)
libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f7828623000)
libc.so.6 => /lib64/libc.so.6 (0x00007f7828256000)
/lib64/ld-linux-x86-64.so.2 (0x00007f7828e42000)
```
- 可以看出, **最简单的hello world程序也需要链接一些库**
- 上述的几种链接库, 感兴趣的可以逐个研究


## 五. 动态链接库 vs 静态链接库
1. 本节以2.3中的示例代码为例, 将math.h, math.cpp打包为静态链接库以及动态链接库, 在main.cpp中引用

### 5.1 静态链接库
1. 编译: g++ -std=c++11 -fPIC -c math.cpp
- fPIC用于生成位置无关的代码, 更多细节可以查找相关资料
2. 生成静态链接库: ar -crv libmath.a math.o
3. 使用这个静态链接库:
- 使用静态库时, 我们需要math.h文件, 这个文件中定义了这个库对外提供的功能
- 除了math.h文件, 我们需要在链接阶段链接libmath.a
4. 示例: main.cpp中已经导入了math.h文件, 编译main.c并链接libmath.a, g++ -std=c++11 -o main main.cpp -L. -lmath
5. ldd main可以看出, main文件不再依赖libmath.a文件

### 5.2 动态链接库
1. 生成动态链接库: g++ -std=c++11 -shared -fPIC math.cpp -o libmath.so
2. 使用动态链接库:
- 需要使用math.h头文件, 该文件定义了库对外提供的功能
- 链接阶段需要链接libmath.so
3. 示例: g++ -std=c++11 -o main main.cpp -L. -lmath
4. 执行main, 会发现无法执行
```
./main: error while loading shared libraries: libmath.so: cannot open shared object file: No such file or directory
```
5. 我们先用ldd 查看main的依赖库:
```
linux-vdso.so.1 =>  (0x00007ffd2adde000)
libmath.so => not found
libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fd3b7ee6000)
libm.so.6 => /lib64/libm.so.6 (0x00007fd3b7be4000)
libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fd3b79ce000)
libc.so.6 => /lib64/libc.so.6 (0x00007fd3b7601000)
/lib64/ld-linux-x86-64.so.2 (0x00007fd3b81ed000)
```
很奇怪, libmath.so没有找到, 我们在第三步编译时明明将这个库加入进去了。这个是由于, 在链接阶段, 链接器可以在当前目录找到libmath.so。执行阶段, 搜索动态链接库时, 并没有包含当前目录, 所以报错。我们可以通过export LD_LIBRARY_PATH=/libpath将libmath.so所在路径放入动态链接库的搜索路径中。此时即可成功执行。

### 5.3 对比
1. 静态链接库, 动态链接库都是二进制文件(ELF格式, 详细信息可以查找相关资料)
2. 从静态链接库生成的过程来看, 其本质就是将多个编译单元(.o文件), 打包为一个新的文件。链接静态链接库时, 会将静态链接库的代码合并进程序中。
3. 链接动态链接库时, 并不会将动态链接库的内容合并进代码中, 而是在程序执行时, 搜索动态链接库, 再进行链接。



## 六. 库之间的依赖
### 6.1 源代码
1. first.h
```cpp
#ifndef __FIRST_H_
#define __FIRST_H_

#include <cstdio>

void first();

#endif 
```
2. first.cpp
```cpp
#include"first.h"

void first()
{
    printf("This is first!\n");
}
```
3. second.h
```cpp
#ifndef __SECOND_H_
#define __SECOND_H_
 
#include <cstdio>
void second();

#endif
```
4. second.cpp
```cpp
#include"first.h"
#include"second.h"

void second()
{
    printf("This is second!\n");
    first();
}
```
5. main.cpp
```
#include"second.h"
int main()
{
    second();
    return 0;
}
```


### 6.2 静态库依赖静态库
1. 生成libfirst.a静态链接库
```
g++ -std=c++11 -fPIC -c first.cpp
ar -crv libfirst.a first.o
```

2. 生成libsecond.a并链接libfirst.a
```
g++ -std=c++11 -c second.cpp -L. -lfirst
ar -crv libsecond.a second.o
```

3. main.cpp中使用libsecond.a
执行: g++ -std=c++11 main.cpp -L. -lsecond -o main
会出现以下错误:
./libsecond.a(second.o): In function `second()':
second.cpp:(.text+0xf): undefined reference to `first()'
collect2: error: ld returned 1 exit status


4. 解释说明

- 通过nm, 我们查看libsecond.a中的符号, 找出未定义的符号, 执行nm -u libsecond.a, 即可发现first并没有定义(编译器编译后的符号并不是first, 我这里是_Z5firstv)。我们明明在生成libsecond.a时链接了libfirst.a?
- 主要的原因是: 生成静态链接库时, 只是将second.cpp生成的second.o打包, 并没有真正的将libfirst.a中的内容链接进libsecond.a
- 静态库不与其他静态库链接。我们使用archiver工具(例如Linux上的ar)将多个静态链接库打包为一个静态链接库


5. 解决方案
- 将first.cpp, second.cpp打包为一个静态链接库: g++ -std=c++11 -fPIC -c first.cpp second.cpp, ar -crv libsecond.a first.o second.o。main中可以直接链接libsecond.a即可
- 同时链接libsecond.a, libfirst.a



### 6.3 动态库依赖静态库
1. 生成libfirst.a静态链接库, 这一步与5.2节相同
2. 生成libsecond.so静态链接libfirst.a
```
g++ -std=c++11 second.cpp -fPIC -shared -o libsecond.so -L. -lfirst
```
- nm -u libseond.so, 我们可以看出, 并没有出现first, 也就是说, libfirst.a已经被链接进libsecond.so中了

3. 编译main.cpp
```
g++ -std=c++11 main.cpp -L. -lsecond -o main
```



### 6.4 静态库依赖动态库
1. 生成libfirst.so
```
g++ -std=c++11 first.cpp -shared -fPIC -o libfirst.so
```

2. 生成libsecond.a链接libfirst.so
```
g++ -std=c++11 -c second.cpp -fPIC -L. -lfirst
ar crv libsecond.a second.o
```
- nm -u libsecond.a, 可以看到_Z5firstv, 说明并没有将libfirst.so中包含进libsecond.a

3. 编译main.cpp
```
g++ -std=c++11 main.cpp -L. -lsecond -lfirst -o main
```
- 如果没有链接first, 会发现链接错误, 找不到first函数的定义


### 6.5 动态库依赖动态库

1. 生成libfirst.so
```
g++ -std=c++11 first.cpp -shared -fPIC -o libfirst.so
```

2. 生成libsecond.so链接libfirst.so
```
g++ -std=c++11 second.cpp -shared -fPIC -o libsecond.so -L. -lfirst
```
- nm -u libsecond.so, 可以看到_Z5firstv, 这个就是first函数
- ldd libsecond.so, 也可以看到libfirst.so
- 可以看出, 使用libsecond.so时, 仍然需要libfirst.so

3. 编译main.cpp
```
g++ -std=c++11 main.cpp -L. -lsecond -o main
```
- 可以看出, 能够成功编译。
- 之前讲过libsecond.so需要依赖libfirst.so, 此处为何我们只链接libsecond.so也能成功呢?这里是因为链接器会自动搜索动态链接库的依赖库



## 七. 总结
1. c或者cpp文件经过**编译、链接**生成可执行文件
2. 单个c文件或者cpp文件是一个编译单元。每个编译单元存在3种符号: 自己使用的, 依赖于外部的以及对外提供的。
3. 链接器是将多个编译单元的符号相互链接以形成可执行文件。
4. 库可以分为静态链接库(.a)以及动态链接库(.so)。
5. 使用库时, 除了库文件, 还需要对应的头文件。
6. 单个c文件或者cpp文件, 可能依赖其他的库文件, 但是在编译时, 只需要有声明, 并不需要有具体的定义。
7. 静态库没有链接操作, 静态库只是将多个.o文件打包, 并没有其他操作。静态库可能依赖其他的静态库或者其他的动态库, 用户在使用静态库时, 需要手动链接这些依赖。
8. 动态库有链接操作, 创建动态库时可以链接其他的库, 也可以不链接, 如果链接静态库, 则会将静态库的内容全部放入动态库, 如果链接动态库, 只是放入符号, 在程序初始化时, 将依赖的这些动态库也加载。如果这个动态库依赖了其他库, 但是没有链接, 也可以生成动态库, 但用户在使用这个动态链接库时, 需要手动链接这些依赖, 由于使用者很难知道这些依赖, 所以通常不使用这种方式。
9. 总体而言, 动态库在程序执行阶段才会装进程序, 静态库则在链接阶段直接放进程序。动态库可以由多个程序共享, 节省内存，易于升级。静态库外部依赖少, 更易于部署。



## 八. 扩展
1. 动态库升级问题?假设现在有2个程序: p1, p2, 一个动态链接库libmath.so.1。如果现在math库提供了新版本libmath.so.2, 程序p1需要使用libmath.so.2的新功能, p2则不想使用, 此时该如何升级math库?
- 如果math不兼容前一版, 则系统中需要同时存在两个版本的math库, p1, p2分别链接不同的版本
- 如果math兼容前一版, 系统中是否可以只保留新版的math库呢?此时p1, p2又是否需要重新编译呢?这个问题留给读者自行思考。

2. 某个动态链接库**lib1**动态链接了库**libbase**, 现在应用程序中使用了**lib1**以及**libbase**, 编译应用程序时, 是否需要链接**libbase**?
- 应用程序不仅需要链接**lib1**, 也需要链接**libbase**
- 链接**lib1**只能保证应用程序依赖**lib1**的部分能够正确解析
- 虽然**lib1**动态链接了**libbase**, 但是动态链接真正进行符号解析是在程序执行阶段, 编译阶段无法获取**libbase**的相关信息, 应用程序中如果也使用了**libbase**中的函数, 则必须链接**libbase**, 否则会出现符号未定义
- 如果**lib1**静态链接了**libbase**, 也就是说包含了**libbase**中的函数, 则应用程序不需要在链接**libbase**

3. 菱形依赖问题, A依赖于B以及C, B、C都依赖于D, 但是是不同版本, 例如B依赖于D1, C依赖于D2, 这种情况下如何链接?
- D2兼容于D1(ABI层面兼容), 程序直接链接D2
- D2不兼容于D1, 查看B是否可以依赖D2重新编译
- 链接器的参数, 直接链接两个版本。ld的参数--default-symver或者--version-script

4. 讨论
- 动态链接会有大量的依赖问题(windows dll hell)
- 由于采用模块化, 又允许升级单个模块, 菱形依赖问题对于很多语言都是存在的
- rust, go等语言都开始采用源码编译的方式, 解决依赖问题


## 九. 参考
- http://blog.chinaunix.net/uid-26548237-id-3837099.html
- https://www.cnblogs.com/fnlingnzb-learner/p/8119729.html
- https://blog.csdn.net/coolwaterld/article/details/85088288
- https://blog.habets.se/2012/05/Shared-libraries-diamond-problem.html
