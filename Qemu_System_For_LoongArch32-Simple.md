# <center>使用QEMU（全系统模式）运行基于LoongArch32架构的Linux系统</center>

<center>（Qemu System For LoongArch32）</center>  

<center>作者：孙海勇</center>

## 1 前言

　　本文的目的是讲解如何通过QEMU的全系统模式运行一个基于LoongArch32制作的Linux系统。

## 2 环境准备

### 准备Linux系统

　　请准备一个通用Linux的环境，比如Fedora、Debian等。

### 下载QEMU源码

　　通过下面的步骤下载支持LoongArch32架构的QEMU的源代码：

```sh
git clone https://github.com/loongson-community/qemu.git -b la32-user-exp --depth 1
```

### 编译QEMU

　　接下来将通过编译源码的方式构建一个可以运行LoongArch32系统的QEMU软件。

```sh
pushd qemu
	mkdir -p build
        pushd build
                ../configure --prefix=/usr/local --audio-drv-list=alsa,pa --enable-virglrenderer --enable-fdt=system \
                             --target-list=loongarch64-softmmu --enable-kvm --disable-docs
		ninja
		sudo ninja install
        popd
popd
```

　　注意：编译过程需要系统中安装必要的编译相关软件包。

#### 下载LoongArch的Linux系统

　　在GitHub上有已经制作好的LoongArch32 Linux系统，使用以下地址步骤进行下载：  

```sh
wget -c	https://github.com/sunhaiyong1978/Yongbao-Embedded/releases/download/0.15/loongarch32s-Yongbao-Embedded-0.15-sysroot.tar.xz
wget -c https://github.com/sunhaiyong1978/Yongbao-Embedded/releases/download/0.15/loongarch32s-Yongbao-Embedded-0.15-kernel_la32s.tar.xz
```

　　下载的系统是一个完全使用开源代码构建的基于LoongArch32指令集架构的Linux系统。


#### 解压缩LoongArch的Linux系统

　　请使用root权限的用户完成以下解压缩的步骤：

```sh
# 创建一个8G大小的磁盘文件
dd if=/dev/zero of=loongarch32s-system.raw bs=10M count=800
mkfs.ext4 loongarch32s-system.raw
mkdir clfs-os
sudo mount loongarch32s-system.raw clfs-os/
sudo tar xvpf /tmp/loongarch32s-Yongbao-Embedded-0.15-sysroot.tar.xz -C clfs-os/
sudo tar xvpf /tmp/loongarch32s-Yongbao-Embedded-0.15-kernel_la32s.tar.xz -C clfs-os/
```

　　经过一段时间的解压后，我们就在/opt/clfs-os目录中拥有了一个基于LoongArch指令集制作的系统。

　　提取启动内核：

　　因内核需要在启动系统前就能被qemu命令找到，因此需要单独将内核复制出来。

```sh
sudo cp clfs-os/boot/vmlinux.efi ./
```

　　在启动前要将挂载系统的目录进行卸载：

```sh
sudo umount clfs-os
```

## 3 使用QEMU

　　接下来我们就可以用qemu命令进行目标系统的启动了，使用如下命令：

```sh
/usr/local/bin/qemu-system-loongarch64 -cpu max32 --kernel ${PWD}/vmlinux.efi -append "root=/dev/vda rw" -drive id=mydrive,file=${PWD}/loongarch32s-system.raw,if=none -device virtio-blk-pci,drive=mydrive -m 256M -smp 1 -nographic
```

　　经过一段时间的启动，我们会看到熟悉的登录提示符：

```sh
　　login:
```

　　此时输入用户名root，密码为空即可进入系统。

　　尝试的输入一些命令，比如ls、vi，你会发现可以完全直接就运行起来了。

　　再输入：  

```sh
uname -m
```  
　　会返回： 

```sh
loongarch32
```

　　代表当前环境正模拟的LoongArch32的架构。

## 结束

　　现在已经可以在LoongArch32的系统中运行一些命令，也可以编译一些软件，但因为运行在QEMU中，所以性能会受到主机系统本身性能的影响较大。

　　感谢大家支持，欢迎提出宝贵的意见。


