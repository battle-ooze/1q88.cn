---
title: 在 Node.js 上运行 tiddlywiki
date: 2018-04-13 16:23:51
tags: [tiddlywiki, node]
categories: Code
---

> tiddlywiki 是一个“非线性个人Web 笔记本”，最大的特点是整个数据的程序都包含在一个 html 文件里，只要打开这个 html 文件就可以完成所有的笔记操作，功能非常强大。

## 基本用法

[https://tiddlywiki.com/](https://tiddlywiki.com/) 官网提供了详细的教程，单机使用的话只要下载一个空的 html 就可以了。

笔记一般是一种随性的行为，特别是对于我这样的老年程序员来说，想到的如果不马上记下来可能就会忘记，所以单机的 tiddlywiki 显然不能满足我的需求。当然也可以把文件放到自己的服务器上，但是你会发现每次保存都会下载一个新的 html，然后你需要自己上传和替换，非常麻烦。tiddlywiki 提供了 Node.js 的版本，可以将 tiddlywiki 部署在服务器上，这样在页面上的操作都会实时保存，而无需自己手动更新。
<!-- More -->
## 安装

首先通过 npm 安装

```bash
npm install -g tiddlywiki
```

## 启动服务

```bash
tiddlywiki mynewwiki --init server #初始化一个tiddlywiki服务的文件夹
tiddlywiki mynewwiki --server #启动服务
```

看到以下字样说明启动成功，默认会启动到 `8080` 端口

```bash
Serving on 127.0.0.1:8080
(press ctrl-C to exit)
```

之后访问 `127.0.0.1:8080` 即可。

## nginx 反向代理和密码验证

如果使用的是云服务器，一般默认不会对外暴露 `8080` 端口，而且通过 IP + 端口访问也不方便，可以用 nginx 做一层反向代理来让我们可以在外网通过域名访问。同时可以增加密码验证，因为 tiddlywiki 本身没有权限验证，所有人都可以修改笔记内容，显然是不安全的。

### 生成密码文件

安装密码生成工具

```bash
yum install httpd-tools
```

生成密码文件，其中 `.passwd` 是文件名，`admin` 是账户名

```bash
htpasswd -c .passwd admin
```

然后输入并确认密码。

### nginx 配置

生成密码文件后就可以在 nginx 中配置

```
server {
  server_name example.com;
  location / {
    proxy_pass http://127.0.0.1:8080;
    auth_basic "password";
    auth_basic_user_file /path/to/.passwd;
  }
}
```

这样就可以通过域名并且需要输入用户名密码访问 tiddlywiki

## 使用 pm2 启动

使用上述方法启动的话，如果会话关闭，服务也会随之关闭，要随时随地都能访问到我们的wiki，需要在后台启动tiddlywiki，因为是 node 服务，我们可以用 pm2 来启动和管理。

### 安装 pm2

```bash
npm install pm2 -g
```

### 启动

```bash
cd mynewwiki #cd到文件夹
pm2 start --name tiddlywiki /usr/bin/tiddlywiki -- --server #启动
```

启动后可以通过 `pm2 list` 查看进程情况。

```bash
┌────────────┬────┬──────┬───────┬────────┬─────────┬────────┬─────┬───────────┬──────┬──────────┐
│ App name   │ id │ mode │ pid   │ status │ restart │ uptime │ cpu │ mem       │ user │ watching │
├────────────┼────┼──────┼───────┼────────┼─────────┼────────┼─────┼───────────┼──────┼──────────┤
│ tiddlywiki │ 0  │ fork │ 22407 │ online │ 0       │ 18h    │ 0%  │ 79.5 MB   │ root │ disabled │
└────────────┴────┴──────┴───────┴────────┴─────────┴────────┴─────┴───────────┴──────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app
```

此时，就可以通过在浏览器输入配置好的域名来访问搭建好的 tiddlywiki 服务了，在PC上或者手机上都可以随时记录自己的笔记。