---
title: 'Android13源码编译'
lead: 记录wsl2 ubuntu2204 通过 docker 编译安卓13源码
date: 2023-11-29
tags:
  - "源码编译"
categories:
  - "源码编译"
sidebar: true #这个标签表示是否在文章页面显示侧边栏。在这个例子中，我们将其设置为 false，因为我们不想在这篇文章的页面上显示侧边栏。如果您想要在文章页面上显示侧边栏，您可以将其设置为 true。

pager: true #这个标签表示是否在文章列表页面上显示分页器。在这个例子中，我们将其设置为 false，因为我们不需要在文章列表页面上显示分页器。如果您有很多文章，并且想要将它们分成多个页面显示，您可以将其设置为 true。

weight: 1 #这个标签表示文章的权重。在 Hugo 中，权重决定了文章在列表中的排序顺序。如果您想要某篇文章排在列表的前面，您可以将其权重设置为更高的数字。在这个例子中，我们将其设置为 1，表示这篇文章应该排在列表的第一位。

menu: 源码编译 #这个标签表示文章所属的菜单。在 Hugo 中，您可以通过定义不同的菜单来组织您的网站导航。在这个例子中，我们将这篇文章添加到名为 "main" 的菜单中。如果您想要将文章添加到其他菜单中，您可以将其设置为相应的菜在
#thumbnail: "img/placeholder.png" # 缩略图
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "post_toc"
---





### 下载源码

新建源码目录

```sh
mkdir ~/android13 && cd ~/android13
```

南方用户 先去[中科大](https://mirrors.ustc.edu.cn/help/aosp.html) 按照aosp镜像使用帮助 下载底包

![image-20231130161314364](../asop%E7%BC%96%E8%AF%91.assets/image-20231130161314364-17013320008172.png)

```shell
wget -c https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar
```

验证一下底包md5看看是否完整 比如当前下载的 a1366894d0ddfbf32db9a719929bdcd7  aosp-20231101.tar

```shell
md5sum aosp-latest.tar
```

解压源码

```shell
tar xvf aosp-latest.tar
```

配置git

```sh
sudo apt-get install git
git config --global user.email xxx@qq.com
git config --global user.name 'abc'
```

 下载 repo 工具。

```shell
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
## 如果上述 URL 不可访问，可以用下面的：
## curl -sSL  'https://gerrit-googlesource.proxy.ustclug.org/git-repo/+/master/repo?format=TEXT' |base64 -d > ~/bin/repo
chmod a+x ~/bin/repo
## 替换repo里的源为中科大的源，这样不用代理，下载源码
export REPO_URL = 'https://gerrit-googlesource.proxy.ustclug.org/git-repo'
```

初始化仓库某个特定的 Android 版本

```sh
repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-13.0.0_r16

## 开始下载源码
repo sync

```

同步时可能出现错误需要拉取仓库源码，不然无法合并

```sh
ls -al        ## 在源码目录中看到隐藏的.repo
cd .repo/repo ## 仓库目录
git pull      ## 拉取源码
cd ../../     ## 回到源码根目录
repo sync     ##再次同步
```



### 编译源码

在源码根目录下新建apt.conf文件（用于配置代理的服务器）

```sh
Acquire::https::proxy "http://172.28.176.1:7890";
Acquire::https::proxy "https://172.28.176.1:7890";
```

在源码根目录下新建sources.list文件（用于配置pip的源）

```sh
deb http://mirrors.163.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-backports main restricted universe multiverse
```

在源码根目录下新建Dockerfile文件（用于生成docker的环境）

```dockerfile
FROM ubuntu:20.04

ARG userid
ARG groupid
ARG username

COPY apt.conf /etc/apt/apt.conf

COPY sources.list etc/apt/sources.list

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update \
    && echo "install package for building AOSP" \
    && apt-get install -y git-core gnupg flex bison build-essential zip curl zlib1g-dev \
        gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev \
        libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig \
    && echo "install utils" \
    && apt-get install -y sudo rsync \
    && echo "install packages for build mesa3d or meson related" \
    && apt-get install -y python3-pip pkg-config python3-dev ninja-build \
    && mkdir -p ~/.pip \
    && echo "[global]" > ~/.pip/pip.conf \
    && echo "index-url = https://mirrors.aliyun.com/pypi/simple" >> ~/.pip/pip.conf \
    && pip3 install mako meson \
    && echo "packages for legacy mesa3d (< 22.0.0)" \
    && apt-get install -y python2 python-mako python-is-python2 python-enum34 gettext

RUN groupadd -g $groupid $username \
    && useradd -m -u $userid -g $groupid $username \
    && echo "$username ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers \
    && echo $username >/root/username \
    && echo "$username:$username" | chpasswd && adduser $username sudo

ENV HOME=/home/$username \
    USER=$username \
    PATH=/src/.repo/repo:/src/prebuilts/jdk/jdk8/linux-x86/bin/:$PATH

ENTRYPOINT chroot --userspec=$(cat /root/username):$(cat /root/username) / /bin/bash -i

```

创建编译的docker 镜像

```sh
docker build --build-arg userid=$(id -u) --build-arg groupid=$(id -g) --build-arg username=$(id -un) -t asop_build_android13 .
```

运行docker镜像

```sh
docker run -it --rm --hostname asop_build_android13 --name asop_build_android13 -v ~/android13:/src asop_build_android13

cd /src

source build/envsetup.sh ## 初始化编译环境

make clean ## 删除out目录下所有文件

lunch ## 选择设备内核和编译版本 lunch sdk_phone_x86_64-userdebug

m ##开始编译
```



### 在Window上运行

上面使用`emulator`命令是在Ubuntu中运行模拟器，会比较卡，画面也不停闪烁。建议在Windows系统中使用Android Studio创建的模拟器运行。

先打包Android镜像：

Android13以下使用命令：

```bash
make sdk sdk_repo
```

在Android13及以上使用命令：

```bash
make emu_img_zip
```

打包完成后会生成`sdk-repo-linux-system-images`为前缀的 AVD 映像 zip 文件，比如我的路径是：
`Aosp/out/target/product/emulator64_x86_64/sdk-repo-linux-system-images-eng.devnn.zip`

将其就地解压或者拷贝到Window系统的某个文件夹下再解压。

笔者选择拷贝到D盘，解压后的位置是：`D:\Downloads\sdk-repo-linux-system-images-eng.devnn`

然后在Android Studio中创建一个模拟器，随便取个名字为 `Pixel_5_WSL`。

在windows命令行中启动模拟器并加载刚才打包的Android系统：

```bash
emulator -avd Pixel_5_WSL -sysdir D:\Downloads\sdk-repo-linux-system-images-eng.devnn\x86_64 -dns-server 8.8.8.8,114.114.114.114 -verbose

## 需要先用android studio 创建一个名为 Pixel_5_WSL的模拟器，才可以启动成功
emulator -list-avds ## 查看创建的名字
##上面这个emulator命令是android sdk的tools目录下的。已将tools目录添到环境变量path中。
```



### 源码导入AndroidStudio中

mmm development/tools/idegen    ## 执行会在 out/host/linux-x86/framework/目录下生成idegen.jar文件

development/tools/idegen/idegen.sh  ## 源码根目录会生成android.iml和android.ipr两个工程配置文件



### 错误 ninja failed with: signal: killed 

> 需要添加swapfile交换空间
>
> 检查当前的交换空间状态：运行以下命令查看当前交换空间的情况：
> **sudo swapon --show**
>
> 创建交换文件（如果没有交换空间）：运行以下命令以创建一个交换文件（通常以 swapfile 命名）：
> **sudo fallocate -l <size> /swapfile**
>
> 其中 <size> 是您想要分配给交换空间的大小，例如 1G 表示 1 GB 的交换空间。
>
> 设置交换文件权限：运行以下命令以设置交换文件的权限：
> **sudo chmod 600 /swapfile**
>
> 格式化交换文件：运行以下命令以格式化交换文件：
> **sudo mkswap /swapfile**
>
> 启用交换空间：运行以下命令以启用交换空间：
> **sudo swapon /swapfile**
>
> 更新 /etc/fstab 文件：打开 /etc/fstab 文件并添加以下行以在启动时自动挂载交换空间：
> **/swapfile none swap sw 0 0**
>
> 验证交换空间：再次运行 sudo swapon --show 命令，确认交换空间已成功启用。
