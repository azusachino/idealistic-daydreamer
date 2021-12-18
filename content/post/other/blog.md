---
title: 记一次博客搭建经历
description:
date: 2021-04-30
slug: other/blog-creation
image: img/2021/04/Porcini.jpg
categories:
  - exp
---

首先说明一点，我的域名是在腾讯云购入的，且已备案，解析采用了 DNSPod，SSL 证书是直接申请的。

## Requirements

- 一个 Github 账户(用于建仓库 & 设置 Github Action)
- 一个域名
- 一台服务器(或者可以用 Github Page)

## 1. 安装 Nginx & 配置 HTTPS

### 1.1 安装 Nginx

因为我的服务器是 centos7，所以配置一下源就可以安装 nginx 了，相关命令如下：

```bash
yum install epel-release # 安装源
yum install nginx -y # 安装nginx
systemctl start nginx # 启动
systemctl enable nginx # 设为开机启动
```

如果 DNS 解析正常的话，这个时候就可以访问成功了。

### 1.2 配置 HTTPS

首先下载证书（具体从哪下载，需要看是谁给你签发的），然后把整套证书文件中 `Nginx` 目录下的 `.crt` 和 `.key` 上传到服务器上，你可以直接放到 `nginx` 的安装目录里，最后在 `nginx.conf` 中启用 `ssl` 的配置就好了，如：

```conf
    server {
        listen       443 ssl;
        # listen       [::]:443 ssl http2 default_server;
        server_name  azusachino.cn;
        root         /usr/share/nginx/html;

        ssl_certificate 1_azusachino.cn_bundle.crt; # nginx.conf相对应的crt路径
        ssl_certificate_key 2_azusachino.cn.key; # nginx.conf相对应的key路径
        # ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # 根据证书下载页教程配置
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; # 根据证书下载页教程配置
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            root html;
            index index.html index.htm;
        }

        error_page 404 /404.html;
        location = /404.html {
        }
    }
```

配置完成之后，`systemctl restart nginx`，然后通过域名访问试一下就知道结果了。

## 2. 开始使用 [hugo](https://gohugo.io/)

A Fast and Flexible Static Site Generator built with love by bep, spf13 and friends in Go.

### 2.1 安装 hugo

因为我的电脑是 win10，而且我安装了`chocolatey`，所以我可以直接通过`choco install hugo`进行安装 。安装之后可以通过`hugo --version`确定安装情况。

如果想通过其他方式安装的话，可以参考[这里](https://gohugo.io/getting-started/installing/)。

### 2.2 使用 hugo

比较细节的使用方式可以参考官网，这里就简单介绍一下。

1. 通过`hugo new yourProject`命令生成新的 hugo 项目。
2. 通过`hugo server`就可以在本地预览项目了。
3. 通过`hugo -D`进行编译，生成的文件都在 public 里，也就是需要放到 nginx 里的所有内容。

#### **比较重要的点**

1. 在[这里](https://themes.gohugo.io/)找到一个喜欢的主题，然后放到`themes`文件夹里面，虽然官方推荐采用`submodule`的形式，但我花了很久也没弄明白怎么用，就直接把主题文件一并上传到仓库了。
2. 根据模板修改`config.yaml`，也就是项目整体的配置，可以参考[我的配置](https://github.com/azusachino/idealistic-daydreamer/blob/main/config.yaml)(注意：**配置和主题是有关联的，我的配置仅供参考**)
3. 另外一个坑了我很久的是，图片一直加载不出来。最终的解决办法是，新建了`static/img`文件夹，然后就可以用`img/xxx.png`的形式在文章中访问到了。

遇到什么问题的话，请善用搜索引擎以及参考别人的博客，或者去看看有没有类似问题的 issue。

## 3. 配置 Github Action

通过配置 Github Action 实现更新代码时自动部署到服务器上。

### 3.1 配置 SSH 自动登录

这个问题坑了我好久，还是太菜了，对很多知识的理解仅仅浮于表面。。。

#### 3.1.1 生成 ssh 密钥对

我们可以通过下面的命令在本地生成`deploy`私钥和`deploy.pub`公钥。

```sh
cd ~/.ssh
ssh-keygen -t rsa -f deploy
```

#### 3.1.2 配置远端服务器

首先安装 rsync，github action 需要这个程序同步文件：`yum install rsync`即可。

其次需要将`deploy.pub`的内容追加到远端服务器的`~/.ssh/authorized_keys`中。

然后修改.ssh 文件夹权限，`chmod 600 -R ~/.ssh` 。(`.ssh/`下的文件权限必须是 600，否则会有安全隐患)

之后，需要修改 ssh 的配置文件。

```sh
vim /etc/ssh/sshd_config
```

启用以下的几个配置，并确保 ssh 配置文件的路径与上述一致。

```s
+ PermitRootLogin yes # 允许root登录
+ PubkeyAuthentication yes # 允许通过公钥验证用户
+ RSAAuthentication yes # 通过RSA算法验证
```

最后重启 ssh 认证服务，`systemctl restart sshd`。

因为我们是在本地生成的密钥，可以直接在本地试验是否配置成功，`ssh root@yourIP`。

### 3.2 添加 GitHub Action 配置文件

在项目中新建`.github/workflows/deploy.yml`，然后添加相应的配置，如下：

```yml
name: Deploy Script

on:
  push:
    branches:
      - main # 只在master上push触发部署
    paths-ignore: # 下列文件的变更不触发部署，可以自行添加
      - README.md
      - LICENSE

jobs:
  deploy:
    runs-on: ubuntu-latest # 使用ubuntu系统镜像运行自动化脚本
    steps: # 自动化步骤
      - uses: actions/checkout@v2 # 第一步，下载代码仓库
      - name: Setup Hugo # 第二步，安装 hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "0.82.1" # hugo 版本
      - name: Build # 第三步，编译 hugo
        run: hugo -D --minify
      - name: Deploy to Server # 第四步，通过rsync推文件
        uses: AEnterprise/rsync-deploy@v1.0 # 使用别人包装好的步骤镜像
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }} # 引用配置，SSH私钥
          ARGS: -avz --delete --exclude='*.pyc' # rsync参数，排除.pyc文件
          SERVER_PORT: ${{ secrets.SERVER_PORT }} # 引用配置，SSH端口
          FOLDER: ./public/* #推送的文件夹，路径相对于代码仓库的根目录
          SERVER_IP: ${{ secrets.SSH_HOST }} # 引用配置，服务器的host名（IP或者域名domain.com）
          USERNAME: ${{ secrets.SSH_USERNAME }} # 引用配置，服务器登录名
          SERVER_DESTINATION: ${{ secrets.SERVER_DESTINATION }} # 引用配置，部署到目标文件夹，如 /usr/share/nginx/html
```

然后我们需要在 Github 的项目页配置 secrets，具体见下图：

![github-secrets](img/2021/04/github-secrets.png)

然后就可以推送代码，看 actions 的执行情况了，祝好运。

### 最后

以前觉得很麻烦的事，现在也能耐下性子好好调查，然后坚持做完了，这就是成长吧。

## 参考链接

- [hugo 官方文档](https://gohugo.io/documentation/)
- [Halfrost-Field 冰霜之地 - 大佬的博客](https://github.com/halfrost/Halfrost-Field)
- [hugo 利用 Github Action 自动构建博客](https://bore.vip/archives/hugo-github-action/)
- [rsync-deploy - 部署脚本仓库](https://github.com/AEnterprise/rsync-deploy)
