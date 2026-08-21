---
title: 'OpenWrt开发入门实战(一)Docker环境搭建'
date: 2024-10-10T16:01:50+08:00
draft: false
tags: ["linux", "openwrt", "docker"]
author: ["zhumouren"]
---

# 1. 环境准备
- 一台带有Windows10(64位)及以上专业版的电脑（其他操作系统也行，只要能用Docker就好了
- CPU为X86_64
- Docker(用作OpenWrt的编译环境和测试环境)

# 2. 使用Docker搭建OpenWrt编译环境
本文使用docker-compse构建，构建脚本目录环境为
```
docker-linux-env/
|   docker-compose.yml
|
----ubuntu-compile-openwrt/
|   |   Dockerfile
|   |   sources.list
|
-
```
以下为各文件的具体内容  

docker-compose.yml
``` yml
version: '3'
services:
  ubuntu-compile-openwrt:
    build: ./ubuntu-compile-openwrt
    environment:
      TZ: Asia/Shanghai
    volumes:
      - compile-openwrt:/root # compile-openwrt 是数据卷
      - compile-openwrt-home:/home
    ports:
     - "2211:22"

volumes:
  compile-openwrt:
  compile-openwrt-home:

```

Dockerfile
```dockerfile
# 以最新的Ubuntu镜像为模板
FROM ubuntu:24.04

# 将本目录下的sources.list作为容器的一个文件
ADD sources.list /root/sources.list
# 使用国内Ubuntu源，更新快
RUN mv /etc/apt/sources.list  /etc/apt/sources.list_bak
RUN cp /root/sources.list  /etc/apt/sources.list

RUN apt update
# 安装常用工具
RUN apt install -y vim git nano

# 安装编译OpenWrt官方实例相关工具
RUN apt install -y build-essential clang flex bison g++ gawk \
gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
python3-setuptools rsync swig unzip zlib1g-dev file wget

# 安装当前镜像对当前OpenWrt编译所需要库
RUN apt install -y libelf-dev locales

# 设置LOCALE
RUN locale-gen en_US.UTF-8

# 修改root密码
RUN echo 'root:pw' | chpasswd

# 添加自定义用户
RUN adduser buildbot \
    && echo 'buildbot:pw' | chpasswd

# 安装ssh
RUN apt install -y openssh-server
RUN mkdir -p /var/run/sshd


# 开放22端口
EXPOSE 22
#设置自启动命令
CMD ["/usr/sbin/sshd", "-D"]
```

sources.list
```
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-backports main restricted universe multiverse

# 以下安全更新软件源包含了官方源与镜像站配置，如有需要可自行修改注释切换
deb http://security.ubuntu.com/ubuntu/ noble-security main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ noble-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-proposed main restricted universe multiverse
# # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-proposed main restricted universe multiverse
```

创建完以上文件之后，在docker-linux-env目录下执行
```shell
docker-compose up ubuntu-compile-openwrt -d
```

等待docker镜像和容器运行起来后，可以通过ssh进入容器内部。可以通过 ssh -p 2211 buildbot@127.0.0.1 进入容器。

# 3. 下载OpenWrt源码
上述步骤正常执行后，进入ubuntu-compile-openwrt容器内部，初始化自己的git配置，然后开始下述步骤。

## 3.1 fork一个自己的OpenWrt版本
1. 先去github上的OpenWrt(https://github.com/openwrt/openwrt)页面，点击fork创建一个自己的版本。
2. 进入编译容器内部，下载自己的OpenWrt源码
```shell
cd /home/buildbot
git clone git@github.com:zzbamboo/openwrt.git # 这是作者本人的地址，实际下载时换成自己的地址
```
3. 从OpenWrt源创建一基于23.05发行版本的分支
```shell
cd openwrt
git remote add upstream https://github.com/openwrt/openwrt.git # 把OpenWrt源地址设为上游地址
git fetch upstream # 获取上游所有分支
git pull upstream openwrt-23.05:openwrt-23.05 #pull openwrt-23.05
git checkout -b openwrt-23.05-study-demo # 创建一个demo分支
```

# 4. 编译OpenWrt 
## 4.1 编译前准备
```shell
# 更新、下载软件包，如果由于众所周知的网络问题导致下载失败，可以先配置代理
./scripts/feeds update -a
./scripts/feeds install -a
```
在OpenWrt项目中`staging_dir/host/bin`目录下有与编译目标无关的通用工具，这些工具在我们之后的开发中有所帮助，现在把这些工具加入到环境变量中。
```shell
export PATH=/home/buildbot/openwrt/staging_dir/host/bin:$PATH
```
最好把上面那条命令追加到`~/.bash_profile`中，不然退出登陆后会失效。


## 4.2 选择构建目标版本
```shell
# Target System选择x86, Subtarget选择x86_64 Target Profile选择Generic x86/64，然后按Esc退出并保存。选择X86是方便在本机设备上进行测试。
make menuconfig #make menuconfig 会打开一个图形化的配置界面。

```

## 4.3 开始编译
```shell
make V=s #开始等待编译吧!
```
如果不出意外，编译成功后在openwrt/bin/targets/x86/64目录下会生成很多文件。

# 5. 构建一个OpenWrt容器

## 5.1 创建docker相关文件
再创建一个目录，目录结构和文件具体内容如下所示
```
openwrt-example/
|   docker-compose.yml
|
|---openwrt-23.05-study-demo/
|   |   openwrt-23.05-study-demo/
|   |   Dockerfile
|   |   
-
```

docker-compose.yml
```yml
version: '3'
services:
  openwrt-23.05-study-demo:
    build: ./openwrt-23.05-study-demo
    environment:
      TZ: Asia/Shanghai
    ports:
     - "11122:22"
     - "11180:80"
    

```

Dockerfile
```dockerfile
# 空白镜像
FROM scratch

ADD openwrt-x86-64-generic-rootfs.tar.gz /

CMD ["/sbin/init"]

```

## 5.1 获取rootfs
在openwrt编译环境中，把文件 openwrt/bin/targets/x86/64/openwrt-x86-64-generic-rootfs.tar.gz 下载到上面创建的文件夹openwrt-example/openwrt-23.05-study-demo/

## 5.2 构建OpenWrt容器
在openwrt-example文件夹下执行
```shell
docker-compose up openwrt-23.05-study-demo -d
```
如果不出意外就能看到容器已经运行