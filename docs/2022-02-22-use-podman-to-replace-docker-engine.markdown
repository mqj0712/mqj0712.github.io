---
layout: default
title:  "忘记docker engine, 拥抱podman"
date:   2022-02-22 12:12:12 +0800
categories: docker podman
---

# 背景
使用docker需要基于root账户运行历来都是个痛点, 无论是命令行的麻烦(*虽然可以用alias* 来消除sudo 的使用), 还是数据的owner方面, 包括导出的tar, 以及挂载的本地文件, 都会出现只有root能访问的问题, 然而最近看到一篇文章 [Docker 大势已去，Podman 即将崛起](https://blog.csdn.net/qq_48289488/article/details/121905018), 可能是解决这个问题的方案, 然而一直没有动力, 直到最近升级了docker engine, 导致本机一致无法运行 mysql 5.6 镜像的container, 就干脆卸载了docker 和 docker-compose, 转投podman 和 podman-compose.


# 安装

![podman](/assets/images/2021/2/podman.svg)  
[Podman官网: https://podman.io](https://podman.io/)

官方提供了各种平台的[安装方式](https://podman.io/getting-started/installation), 例如 Arch Linux:

```bash
    sudo pacman -S podman podman-compose podman-dnsname
```
其中`podman`是主程序, 后面的 `podman-compose` 和`podman-dnsname` 是替换*docker-compose* 用的 




# 配置以及支持包
重点是在账户下运行podman, 因此两个额外的支持包需要安装:`slirp4netns`, `fuse-overlayfs` , 分别是支持用户空间的网络和文件系统.

另一个必须的包是 `shadow-utils`, 在Arch Linux中是 `shadow`, 安装完成后, 修改文件 (如果没有可以用root用户创建)

* /etc/subuid

        <username>:10000:65536

* /etc/subgid

        <username>:10000:65536

然后使用命令让podman应用上面的修改

```bash
    podman system migrate
```

默认情况下用户空间的podman不能绑定本机小于1024的端口号, 如果有特殊需要, 例如443, 则可以修改kernel的参数, 对Arch linux, 创建文件 `/etc/sysctl.d/podman.conf`

```
net.ipv4.ip_unprivileged_port_start=443
```

# 其他

* 支持docker hub repo, 在 `/etc/containers/registries.d/somefile.conf` 或者 `$HOME/.config/containers/registries.conf`中增加配置:
```
unqualified-search-registries = ["docker.io"]
```

* 使用docker-compose.yml 无法解析service 名称

    这个通过安装podman-dnsname解决

* 如果本机配置了代理, proxy相关的环境变量(http_proxy, https_proxy...)会默认增加到container里面, 如果这不是你想要的, 可以通过修改 `/etc/containers/` 或 `
$HOME/.config/containers/` 中的containers.conf , 就可以解决这个
```
http_proxy = false
```