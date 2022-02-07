---
title: 网络是怎样连接的 - 浏览器与HTTP
description: How did the browser work?
date: 2022-02-07
slug: understand-network-browser-http
image: img/2022/02/SaCalobra.jpg
categories:
  - learning
tags:
  - network
---

通过与浏览器简单的交互，学习互联网中最基础的知识点。

## 生成 HTTP 请求消息

我们要做的第一步就是在浏览器中输入网址，如`https://azusachino.cn/p/understand-network-browser-http/`。

- URL (Uniform Resource Locator)
- URI (Uniform Resource Identifier)

### 解析 URL

浏览器的第一步工作就是对 URL 进行解析，从而生成发送给 Web 服务器的请求消息。

![ ](img/2022/02/network-http-elements.jfif)

### 组装 HTTP 请求

HTTP 协议定义了客户端和服务端之间交互的消息内容和步骤。

1. 客户端向服务端发送请求消息，消息中的内容用 REST 的思想来说就是：针对某个资源进行某个操作的描述。
2. 服务端在收到请求消息之后，会根据描述执行相应的操作，然后将结果存放在响应消息中。

#### 请求/响应消息示例

![ ](img/2022/02/network-http-request.png)

![ ](img/2022/02/network-http-response.png)

#### HTTP 方法清单

![ ](img/2022/02/network-http-methods.jfif)

## 通过 DNS 服务器查询 Web 服务器的 IP 地址

### IP 地址 (IPv4)

一般来说，IP 地址就像是邮政地址一样，收发方都是唯一的，这样才能保证邮件能够到达指定地点。

![ ](img/2022/02/network-http-ip.jfif)

### Socket 库提供查询 IP 能力

```c
#include <stdio.h>
#include <netdb.h>

int main(int argc, char *argv[])
{
    struct hostent *lh = gethostbyname("azusachino.cn");
    if (lh)
        puts(lh->h_name);
    else
        herror("gethostbyname");

    return 0;
}
```

### DSN 解析示例

![ ](img/2022/02/network-http-dns-resolve.jfif)

## DNS 服务器层层接力

![ ](img/2022/02/network-http-dns-search.jfif)

## 委托协议栈发送消息

1. 创建套接字
2. 将管道连接到服务器端的套接字上
3. 收发数据
4. 断开管道并删除套接字

![ ](img/2022/02/network-http-socket.jfif)

## Reference

- [微信读书 | 网络是怎样连接的](https://weread.qq.com/web/reader/6f932ec05dd9eb6f96f14b9kc81322c012c81e728d9d180)
