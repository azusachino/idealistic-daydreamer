---
title: 网络是怎样连接的 - 浏览器与HTTP
description: 探索浏览器内部
date: 2022-02-07
slug: understand-network-browser-http
image: img/2022/02/SaCalobra.jpg
categories:
  - learning
tags:
  - network
---

通过与浏览器进行简单的交互，学习网络相关的基础知识点。

## 整体预览

![ ](img/2022/02/network-preface.jfif)

## 通过浏览器生成 HTTP 请求消息

我们要做的第一步就是在浏览器中输入网址，如`https://azusachino.cn/p/understand-network-browser-http/`。其中网址整体被称作 URL，而`azusachino.cn/`之后的内容叫做 URI。

- URL (Uniform Resource Locator)
- URI (Uniform Resource Identifier)

### 解析 URL

浏览器的第一步工作就是对 URL 进行解析，从而生成发送给 Web 服务器的请求消息。

![ ](img/2022/02/network-http-elements.jfif)

浏览器支持的协议多种多样，如 `http`、`ftp`、`file` 和 `mailto` 等，当然在日常生活中 `http` 和 `https` 用的比较多。

### 组装 HTTP 请求

在解析完 URL 之后，浏览器就会向目标服务器发送 `HTTP` 请求了。其中，`HTTP` 协议定义了客户端和服务端之间交互的消息内容和步骤。

1. 客户端向服务端发送请求消息，其内容用 `REST` 的思想来说就是：针对某个资源进行某种操作的描述。
2. 服务端在收到请求消息之后，会根据描述执行相应的操作，然后将结果存放在响应消息中。

#### HTTP 请求/响应消息示例

请求和响应中有非常多的 Header 信息，这里就不方便展示了，可以借一步查询[官方文档](https://developer.mozilla.org/zh-CN/docs/Glossary/HTTP_header)。

![ ](img/2022/02/network-http-request.png)

![ ](img/2022/02/network-http-response.png)

#### HTTP 方法清单

![ ](img/2022/02/network-http-methods.jfif)

## 通过 DNS 服务器查询 Web 服务器的 IP 地址

当然，在 `HTTP` 请求真正发送到网络之前，浏览器还需要向 `DNS` 服务器查询目标地址的 真实 IP，毕竟是 `TCP/IP` 嘛。

### IP 地址 (IPv4)

一般来说，IP 地址就像是邮政地址一样，收发方都是唯一的，这样才能保证邮件能够到达指定地点。

当然有部分 IP 是可以重复的，比如经常在 docker 中看到的 `172.16.0.0/12`，毕竟 IPv4 的地址已经不够用了，不过重复的 IP 就不能在公网使用了。

![ ](img/2022/02/network-http-ip.jfif)

### 通过 Socket 库获取查询 IP 能力

简单展示 `gethostbyname` 的使用方式。

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

### DNS 解析流程示例

![ ](img/2022/02/network-http-dns-resolve.jfif)

### DNS 服务器的层层接力

![ ](img/2022/02/network-http-dns-search.jfif)

## 委托协议栈发送消息

1. 创建套接字
2. 将管道连接到服务器端的套接字上
3. 收发数据
4. 断开管道并删除套接字

![ ](img/2022/02/network-http-socket.jfif)

## 总结

一味地抄书，我好菜。

## Reference

- [微信读书 | 网络是怎样连接的](https://weread.qq.com/web/reader/6f932ec05dd9eb6f96f14b9kc81322c012c81e728d9d180)
