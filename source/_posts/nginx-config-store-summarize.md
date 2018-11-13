---
title: nginx 配置存储概述
date: 2018-11-13 15:56:40
categories:
- nginx
- 配置解析
tags:
- nginx
- 配置存储
---

# 一. 基础
nginx的一般配置如下所示:
```cpp
...
work_porcess: xx;
events{
    ...
    work_connections xx;
}
http{
    //第一级别的配置块
    ...
    upstream xx{
        //第二级别的配置块
        ...
    }
    server{
        //第二级别的配置块
        ...
        location /{
            //第三级别的配置块
            ....
            location /{
                //第四级别的配置块
                ...
            }
        }
    }
    server{
        //第二级别的配置块
        ...
    }
}
```
1. 如何存储上述结构?
> - 由于配置中存在嵌套, 可以使用树型结构进行存储
2. 如何使用上述配置?
> - 从上到下, 按层查找
3. 配置解析核心?
> - 对于块中嵌套块的模块存储(例如http), 核心在于要以块为单位进行分析。例如, 最外层为http块, 属于一级配置; http中的server或者upstream属于二级配置块; server中的location属于三级配置块; 依次类推, 每块都看作一个完整的结构。
> - 由于存在块中嵌套, 内层的块有些配置需要从外层块继承, 有些配置不能继承, 故而每块的配置需要按区域划分。
> - 二级配置块需要放到一级配置块中, 这样才能从一级配置块中找到所有的二级配置块, 以此类推。
> - 由于配置解析是依靠各个模块完成的, 故而配置存储的树形结构中需要以模块为单位存储(也就是每个模块具有自己的存储结构, 可以放在树形结构的某个位置上)。


# 二. nginx配置存储结构
由于结构比较复杂, 此处分为2部分。
1. 第一部分配置存储结构图
![image](https://picturestore.nos-eastchina1.126.net/nginx/nginx%E9%85%8D%E7%BD%AE%E5%AD%98%E5%82%A81.png)

> - 图中只给出了http中的server配置, 没有画出server中的location以及location嵌套的location

2. 第二部分配置存储结构图
![image](https://picturestore.nos-eastchina1.126.net/nginx/nginx%E9%85%8D%E7%BD%AE%E5%AD%98%E5%82%A82.png)


# 三. http块存储结构
对于http的配置存储, 应当以块为单位进行分析,
1. 总体而言, http块按树形存储
2. 大体上可以分为3个级别(不考虑location中的location), 每个级别的配置分为3块(main块, srv块, loc块)
3. 第二级别的配置继承第一级别的main块配置
4. 第三级别的配置继承第二级别的main块以及srv块配置
5. 第四及其后续级别的配置继承前一级的main块, srv块配置
![image](https://picturestore.nos-eastchina1.126.net/nginx/nginx%E9%85%8D%E7%BD%AE%E5%AD%98%E5%82%A8-http%E6%A6%82%E5%86%B5.png)


# 四. nginx upstream块存储
1. upstream的配置与server类似, 属于第二级别配置块
2. upstream的配置块中main块从第一级别的main块继承
![image](https://picturestore.nos-eastchina1.126.net/nginx/nginx%E9%85%8D%E7%BD%AE%E5%AD%98%E5%82%A83-upstream.png)


# 五. nginx proxy存储
1. proxy_pass出现在location块中, 属于第三或者之后级别的配置。
![image](https://picturestore.nos-eastchina1.126.net/nginx/nginx%E9%85%8D%E7%BD%AE%E5%AD%98%E5%82%A84-proxy.png)



