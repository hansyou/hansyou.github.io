---
title: NAS 上相对安全和方便的 Gitea 搭建方法
description: 本文介绍了作者因网络问题和国内平台监管问题，将私人代码从 GitHub 迁移到自建的 Gitea 服务的过程。作者推荐使用 Docker 搭建 Gitea 服务，因为 Docker 提供了数据迁移的灵活性和方便性。文章详细介绍了 Dockerfile 的配置方法，包括网络、服务、数据卷和端口映射的设置。此外，还介绍了如何通过 `app.ini` 文件进行配置，以及如何设置反向代理和端口转发以确保服务的可访问性。最后，作者分享了自己的部署方案，包括使用路由器进行反向代理和端口转发，确保 Gitea 服务的安全性和可用性。
date: 2025-04-03 17:15:22
categories:
    - NAS 网络附属存储
tags:
    - 技术
---

最近把 Github 上的一些私人代码迁移到了自建的 Gitea 服务，主要原因是网络问题以及国内平台监管问题。既想要良好的网络环境，又不想被国内 DevOps 平台监管，就只能自建了。

玩 NAS 到现在快一年了，现在几乎不想在任何网盘上存储私人数据……只能说买晚了。

## docker 搭建 Gitea 服务

虽然有很多方式搭建 Gitea，但我还是推荐 docker 搭建，因为 docker 可以更方便地迁移数据，先上 dockerfile：

```dockerfile
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: docker.gitea.com/gitea:latest
    container_name: gitea
    restart: unless-stopped
    networks:
      - gitea
    volumes:
      - /path/to/gitea-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
```

只需要修改 `- /path/to/gitea-data:/data`这个地方。通过 docker，可以灵活地将数据放置在需要的磁盘上，更加方便。数据迁移的时候也只需要将映射目录进行相应修改即可。

对于“ports”，家用宽带已经封禁了 22、80、443 端口，因此必须映射到非常用端口。其中，3000 端口用于网页访问，22（容器内）端口用于 ssh 协议。如果你确保不使用 ssh 功能的话，22 端口是可以不用映射的。

如果有一台专用的 vps 来部署 Gitea 服务的话，可以直接映射到 80 和 22 端口。

```dockerfile
    ports: 
      - "80:3000"
      - "22:22"
```

这样的话可以直接通过 ip 地址或域名在浏览器中不带端口号进行访问。

## app.ini 文件配置

以下示例使用 3000 和 2222 端口。

配置好 docker 容器之后，访问 ip: 3000 进入网页初始化界面，按照提示配置即可，比较简单，就不上图了。重点是，后续如何更改配置。

网页初始化的配置文件在 `/path/to/gitea-data/gitea/conf/app.ini` 路径中。

```ini
APP_NAME  = Gitea: Git with a cup of tea
RUN_MODE  = prod
RUN_USER  = git
WORK_PATH = /data/gitea

[repository]
ROOT = /data/git/repositories

[repository.local]
LOCAL_COPY_PATH = /data/gitea/tmp/local-repo

[repository.upload]
TEMP_PATH = /data/gitea/uploads

[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.domain
SSH_DOMAIN = gitea.domain
HTTP_PORT = 3000
ROOT_URL = https://gitea.domain:port/
DISABLE_SSH = false
SSH_PORT = 2222
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = v7cZ2_HQwqXuNTeRJ509CpNDaBqBdZ6PXItuMlwnPBg
OFFLINE_MODE = true
……
```

大概内容如上所示，需要关注的可修改对象主要为 `[server]`：

1. DOMAIN：服务器绑定的**域名**
2. SSH_DOMAIN：所用 SSH 服务的**域名**，一般来说与 DOMAIN 相同
3. HTTP_PORT：**容器内**网页服务端口
4. ROOT_URL：实际访问所使用的**网址链接**，如果进行了反向代理，需填写代理后的网址
5. SSH_PORT：实际 SSH 服务所使用的端口
6. SSH_LISTEN_PORT：容器内 SSH 端口，**不可修改**

由于我已经对 3000 端口网页进行了反向代理，因此使用了 https。

## 反向代理与端口转发

3000 端口的网页使用 http 协议，因此可以直接进行方向代理。

https://gitea.domain:port/ -> http://ip:3000

但是 2222 端口使用的 SSH 服务是不能反向代理的，只能进行端口转发或者直连。

这两点要尤其注意。

## 我的方案

路由器：Lucky 反向代理内网 NAS 的 Gitea 网页服务，Lucky 将 tcp4/tcp6 的 NAS 上 2222 端口转发到路由器的 2222 端口，然后防火墙进行放行即可。
