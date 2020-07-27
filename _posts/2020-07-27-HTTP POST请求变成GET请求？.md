---
layout: post
title: 'HTTP POST请求变成GET请求？'
date: 2020-07-27
author: pyihe
tags: [问题记录, nginx, http]
---

### 问题描述

客户端发起的HTTP POST请求, 到达服务器后请求方法莫名其妙变成了GET请求, 导致客户端收到的是404。

### 问题定位

1. 首先检查代码, 再三确认并且在测试环境上验证后确保代码没问题。

2. 因为是生产环境出现的问题, 便排查测试环境和生产环境的区别: 

    1. 两者客户端代码相同
    2. 两者服务端代码相同
    3. 生产环境用Nginx做了域名转发, 而测试环境直接使用的IP访问, 未使用代理

于是尝试将问题定位在Nginx转发上, 马上重现问题, 并且观察Nginx日志, 发现转发时出现了如下日志: 

```
[27/Jul/2020:15:53:01 +0800] [访问的URL] "POST /api/v/game/query HTTP/1.1" 301 185 "-" "PostmanRuntime/7.25.0" "-"
[27/Jul/2020:15:53:01 +0800] [访问的URL] "GET /api/v/game/query HTTP/1.1" 404 18 "http://访问的URL/api/v/game/query" "PostmanRuntime/7.25.0" "-"
```

从上面的日志可以看出: 

1. Nginx收到的请求确实为POST请求, 客户端请求没有问题。
2. 从Nginx到服务端请求发生了变化: POST被转换成了GET请求, 由此服务端返回404。

**由此可以确定: 问题的出现是因为Nginx的转发使HTTP方法类型发生了变化。**

### 问题原因

那么Nginx为什么会把POST请求转换成GET请求呢？注意上面的第一行日志中有301的字样, 301状态码的意思是: **资源位置永久改变, 需要重定向**, 通常用于将HTTP请求迁移到HTTPS。

到这里, 回头看看Nginx的配置文件, 文件中配置了`listen 443 ssl`, `ssl_certificate`, `ssl_certificate_key`等参数, 即Nginx配置的是HTTPS服务, 所有请求将以HTTPS访问, 对于HTTP请求, 将会被以HTTPS的形式重定向。

再看看客户端发起请求的URL, 确实是HTTP请求, 所以触发了重定向, 也就导致了问题的产生。

即使通过Nginx将HTTP转换成了HTTPS, 这里也并没有解释为什么POST会变成GET请求, 这里就需要祭出著名的《图解HTTP》中关于状态码的解释了: 

书中关于`3xx`状态码的解释: 

**1. 301-Moved Permanently(永久性重定向), 该状态码表示请求的资源已经被分配了新的URI, 以后应使用资源现在所指的URI, 也就是说如果已经把资源对应的URI保存为书签了, 这时应该按Location首部字段提示的URI重新保存。**

**2. 302-Found(临时重定向), 该状态码表示请求的资源已经被分配了新的URI, 希望用户(本次)能使用新的URI访问。和301不同的是, 302不是永久移动, 只是临时性质的, 也就是已移动的资源对应的URI将来还有可能发生改变, 如果URI被保存为书签, 用户不需要更新书签。**

**3. 303-See Other(存在另一个URI), 该状态码表示请求的资源存在着另一个URI, 应使用GET方法定向获取请求的资源。303和302功能相同, 但303明确表示客户端应当采用GET方法获取资源。比如, 当使用POST方法访问时, 其执行后的处理结果是希望客户端能以GET方法重定向到另一个URI上去, 则返回303状态码。**

**4. 304-Not Modified(未满足条件的URI), 该状态码表示客户端发送附带条件的请求时, 服务器允许请求访问资源, 如果未满足条件, 则返回304。**

**5. 307-Temporary Redirect(临时重定向), 该状态码与302有相同的意义, 302禁止POST变换成GET, 但是在实际使用中, 大家并不遵循, 仍然将POST转换成了GET。307会遵照标准, 不会从POST变成GET。**

**注: 当301、302、303状态码返回时, 几乎所有的浏览器都会把POST改成GET, 并删除请求报文内的主体, 之后请求会自动再次发送。即使301, 302禁止将POST方法改成GET方法, 但实际使用中大家仍然将其改成了GET。**

到这里, 原因已经很明了了。

### 问题解决

对于这里的问题场景, 我们不希望POST请求被改成GET请求, 则解决方法有:  

1. 如果可以, 将客户端发起的HTTP请求改为HTTPS请求, 这样便不会重定向。

2. 将Nginx配置文件中的`return 301 $URI`永久重定向改为`return 307 $URI`临时重定向。

### 参考资料

1. [stack overflow](https://stackoverflow.com/questions/39280361/nginx-loses-post-variable-with-http-https-redirect)

1. [《维基百科》](https://zh.wikipedia.org/wiki/HTTP_301)

2. [《图解HTTP》](https://book.douban.com/subject/25863515//)
