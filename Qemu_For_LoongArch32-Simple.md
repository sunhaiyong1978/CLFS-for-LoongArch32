﻿﻿﻿﻿﻿# <center>使用QEMU运行基于LoongArch32架构的Linux系统（简化版本）</center>

<center>（Qemu For LoongArch32 Simple）</center>  

<center>作者：孙海勇</center>

## 1 前言

　　本文的目的是通过简单的操作使用QEMU运行一个基于LoongArch32制作的Linux系统。

## 2 环境准备
### 准备Linux系统
　　请准备一个通用Linux的环境，比如Fedora、Debian等。

### 下载QEMU
　　在Linux环境中下载可以支持LoongArch架构的QEMU：

　　机器是X86系统，下载地址：

https://github.com/sunhaiyong1978/Yongbao-Embedded/releases/download/0.9/qemu-x86_64-to-loongarch32

　　如果你刚巧手上有一个龙芯3A 5000/6000的机器(使用的是LoongArch ABI 2.0的操作系统)，可以下载：

https://github.com/sunhaiyong1978/Yongbao-Embedded/releases/download/0.9/qemu-loongarch64-to-loongarch32

请将下载的文件更名为qemu-loongarch32，并存放到/usr/bin目录中，如X86的文件使用命令：

```sh
sudo cp qemu-x86_64-to-loongarch32 /usr/bin/qemu-loongarch32
```

给qemu-loongarch32文件赋予可执行权限：

```sh
sudo chmod +x /usr/bin/qemu-loongarch32
```

#### 下载LoongArch的Linux系统
　　在GitHub上有已经制作好的LoongArch32 Linux系统，使用以下地址步骤进行下载：  

```sh
cd /tmp
wget -c https://github.com/sunhaiyong1978/Yongbao-Embedded/releases/download/0.10/loongarch32-Yongbao-Embedded-0.10-20250516-sysroot.tar.xz
```

　　以上下载的系统是一个完全使用已开放的源代码构建的基于LoongArch32指令集架构的Linux系统。


#### 解压缩LoongArch的Linux系统
　　请使用root权限的用户完成以下解压缩的步骤：

```sh
cd /opt
sudo mkdir clfs-os
cd clfs-os
sudo tar xvpf /tmp/loongarch32-Yongbao-Embedded-0.10-20250516-sysroot.tar.xz
```
　　经过一段时间的解压后，我们就在/opt/clfs-os目录中拥有了一个基于LoongArch指令集制作的系统。

## 3 使用QEMU

　　接下来就是在Binfmt注册LoongArch可执行文件的信息了，使用如下命令：

```sh
sudo bash -c 'echo ":qemu-loongarch32:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x02\x01:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-loongarch32:" > /proc/sys/fs/binfmt_misc/register'
```

　　以上命令注册了LoongArch可执行文件“头信息”，并指定符合条件的文件会使用"/usr/bin/qemu-loongarch32"命令来执行，所以这个"/usr/bin/qemu-loongarch32"命令必须真实有效。

#### Chroot到LoongArch系统
　　要想方便的通过QEMU的Linux-User模式chroot到LoongArch的系统中，Binfmt功能的注册是必不可少的，当完成注册后还需要如下的一次性步骤：

```sh
sudo cp /usr/bin/qemu-loongarch32 /opt/clfs-os/usr/bin/
```

　　我们将qemu-loongarch32这个命令文件复制到需要chroot的系统中，这个步骤非常关键，并且要保证qemu-loongarch32在这个chroot的系统中存放的相对位置与当前系统中该命令的目录位置相同，即放在/usr/bin目录下。

　　接下来，我们就是见证胜利的时候，使用root权限的用户或者sudo命令进行chroot：

```sh
sudo chroot /opt/clfs-os
```
　　这个时候我们会看到熟悉的Bash提示符：

　　bash-5.1#

　　尝试的输入一些命令，比如ls、vi，你会发现可以完全直接就运行起来了。

　　再输入：  

```sh
uname -m
```  
　　会返回： 

　　loongarch32

　　代表当前环境正模拟的LoongArch32的架构。

　　此时虽然已经进入到LoongArch32的系统中，但还需挂载一些基础文件系统才能正常工作，命令如下：

```sh
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev
```

　　因当前使用chroot命令切换到了LoongArch32的系统中，默认使用root用户，因此挂载文件系统无需sudo命令即可进行操作。


## 结束

　　现在已经可以在LoongArch32的系统中运行一些命令，也可以编译一些软件，但因为运行在QEMU中，所以性能会受到主机系统本身性能的影响较大。

　　感谢大家支持，欢迎提出宝贵的意见。






