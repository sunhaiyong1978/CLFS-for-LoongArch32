# <center>手把手教你构建基于LoongArch32架构的Linux系统</center>

<center>（CLFS For LoongArch32）</center>  

<center>作者：孙海勇</center>


## 0 前言
　　龙芯中科于2021年推出了全新指令集架构LoongArch，其中32位指令集称为LoongArch32。 
 
　　本文的目标是为LoongArch32制作一套基本的Linux系统，作为对新的指令集架构而制作Linux系统，默认该架构平台上无可运行的系统为前提，采用交叉编译的方式为其制作一套基本的Linux系统。  

　　如果了解如何制作LoongArch64的Linux系统，请查看《手把手教你构建基于LoongArch64架构的Linux系统》


## 1 关于软件包的移植
　　对于本文所制作的目标系统是基于LoongArch32架构的Linux系统，对于LoongArch32架构的支持在本文发布时属于比较新的状态，很多Linux系统的基本软件包中都没有包含该指令集相关的支持，为了解决支持新架构的问题，可根据不同情况的软件包采用不同的处理方式。

* 扩充式移植软件包 
 
　　这类软件包通常在Linux系统中与具体指令集架构细节打交道的软件包，例如：Linux内核、GCC、Binutls、Glibc以及LLVM等等，这些软件包通常需要大量代码的加入才能支持新的指令集架构。

　　对于这类软件包，想以最佳的手段支持新指令集架构，当然是提交到官方的最新版本中并得到长期的支持，但要达到这样的结果是需要一个过程，那么在这个过程中则可以采用“打补丁”的方式。但因通常这些软件包需要修改和增加大量的代码，这使得补丁文件通常针对具体的版本，当版本升级后通常补丁不能直接使用，所以要使用对应版本的源代码来使用补丁。  

　　另一种“不太好”的方式是添加了新架构的完整源代码整体提供下载，即补丁已经打在源代码中，这样只要下载修改过的软件包源码就可以使用了。


* 简易移植软件包  

　　这类软件包代码上基本上不涉及汇编或者有非汇编的实现（汇编通常作为优化性能的手段），此类软件包通常有多种指令集架构采用类似的工作行为，可在某一类工作行为上加入新指令集架构的判断或者通过较少的改动即可实现对新指令集架构的移植，比如：Systemd、Automake等，因此针对这类软件包的补丁具有较高的版本通用性，同一个补丁可能适合用于多个版本上，在该类软件包的官方支持新指令集架构之前，采用“打补丁”的方式更适合这类软件包的移植方式。

* 无需移植软件包  

　　这类软件包大多采用非汇编的开发语言进行编写，具有较强的通用性，通常在其所依赖的编译器或者运行环境进行了移植后就可以直接进行编译或使用了。例如Coreutils、Findutils等。  

　　这类软件包也可能需要在配置阶段进行新架构的支持，主要是软件包自带的config.sub和config.guess检查目标系统时没有匹配的架构设置导致错误，这类问题比较好解决，只需要将增加了新架构的Automake软件包中的config.sub和config.guess覆盖软件包中的文件即可。

　　除了以上这些在新架构平台上可移植的软件包外还有一些软件包是针对某一个特定的指令集架构编写的，如果是非核心功能的软件包可以暂时忽略，如果有对应功能的可移植软件包也可以用来替代这些特定平台的软件包。
　　

## 2 准备工作
　　在开始制作前，先做一些准备工作，这包括系统的准备、制作环境的设置以及软件包源代码的下载。

### 2.1 系统的准备
　　首先准备一台可以安装通用Linux系统的机器，对于要用来交叉编译目标平台的系统我们称为“主系统”，“主系统”可以是X86机器上的系统，也可以是其他架构机器上的系统，为了方便讲解，本文采用在X86架构上的Linux系统进行制作讲解。

　　选择一个合适的Linux对于能否顺利完成制作还是有一定的作用，这里可以使用常见的发行版，如Fedora、Debian、CentOS等，也可以使用专门的系统，如LFS的LiveCD等，接下来我们以Fedora系统作为交叉编译的“主系统”进行讲解。

　　为了使制作系统讲解的过程中尽量减少额外的因素导致的问题，我们在一个“重新搭建的”Fedora系统中进行制作，在一个支持dnf命令工具的系统中使用如下命令进行搭建：

```sh
export DISTRO_URL=https://mirrors.bfsu.edu.cn/fedora/releases/38/Everything/x86_64/os/
sudo dnf install @core @c-development glibc-langpack-zh rpm-build git wget texinfo \
                 zlib-devel rsync libunistring-devel libffi-devel gc-devel \
                 expat-devel pcre2-devel glib2-devel cmake openssl-devel libyaml-devel \
                 libxml2-devel cairo-devel libxslt-devel gettext-devel \
                 glib2-static libstdc++-static zlib-static \
                 fpc tcl ncurses-devel gperf openssl icu docbook-style-xsl bc squashfs-tools \
                 graphviz doxygen xmlto xcursorgen dbus-glib lynx gtk-doc sqlite \
                 asciidoc itstools perl-open perl-FindBin \
                 --installroot ${HOME}/la-clfs --disablerepo="*" \
                 --repofrompath core,${DISTRO_URL} \
                 --releasever 38 --nogpgcheck
```
　　以上步骤将在当前用户的目录中创建"la-clfs"的目录，在这个目录中将安装一个基本的制作环境，这里安装的是Fedora 38的系统，读者也可以安装其它的系统作为制作环境。

　　接下来的制作过程都将在这个目录中进行。

　　复制当前系统的域名解析配置文件到新建立的系统中，以便该系统可以访问网络资源。

```sh
cp -a /etc/resolv.conf ${HOME}/la-clfs/etc/
```

　　接下来切换到该目录中:

```sh
sudo chroot ${HOME}/la-clfs
```

　　挂载必要的文件系统：

```sh
mount -t proc proc proc
mount -t sysfs sys sys
mount -t devtmpfs dev dev 
mount -t devpts devpts dev/pts 
mount -t tmpfs shmfs dev/shm
```

### 2.2 制作环境的设置

#### 创建必要的目录
　　使用如下命令创建几个目录，后续的制作过程都将在这些目录中进行。

```sh
export SYSDIR=/opt/mylaos
mkdir -pv ${SYSDIR}
mkdir -pv ${SYSDIR}/downloads
mkdir -pv ${SYSDIR}/build
install -dv ${SYSDIR}/cross-tools
install -dv ${SYSDIR}/sysroot
```

　　简单说明一下这几个目录的用处：

* 通过设置“SYSDIR"变量方便对“基础目录”的使用，该变量设置了一个具体的目录作为“基础目录”，与本次制作相关的工作都在该目录中进行。

* “downloads”目录用来存放各种软件的源码包以及补丁文件；

* “build”目录用来编译各个软件包；

* “cross-tools”目录用来存放交叉工具链及相关的软件；

* “sysroot”用来存放目标平台系统。

#### 创建制作用户

　　为了防止制作过程中意外的对系统本身造成破坏，创建一个普通用户的账号，后续的制作过程除非需要特殊权限操作，否则对于目标系统的一切操作都使用该用户进行。

```sh
groupadd lauser
useradd -s /bin/bash -g lauser -m -k /dev/null lauser
```
　　设置目录为新创建用户所属：

```sh
chown -Rv lauser ${SYSDIR}
chmod -v a+wt ${SYSDIR}/{sysroot,cross-tools,downloads,build}
```


##### 切换到制作用户

　　使用命令切换到新创建的用户：  

```sh
su - lauser
```

　　使用“su”命令进行切换时加上“-”参数可以防止切换前的用户环境变量带到新用户环境中。

##### 设置制作用户环境

　　为制作用户设置最精简和必要的环境变量，以帮助后续制作过程的开展，以下为用户的环境变量进行长期设置。

```sh
cat > ~/.bash_profile << "EOF"
exec env -i HOME=${HOME} TERM=${TERM} PS1='\u:\w\$ ' /bin/bash
EOF
```

```sh
cat > ~/.bashrc << "EOF"
set +h
umask 022
export SYSDIR="/opt/mylaos"
export BUILDDIR="${SYSDIR}/build"
export DOWNLOADDIR="${SYSDIR}/downloads"
export LC_ALL=POSIX
export CROSS_HOST="$(echo $MACHTYPE | sed "s/$(echo $MACHTYPE | cut -d- -f2)/cross/")"
export CROSS_TARGET="loongarch32-unknown-linux-gnu"
export MABI="ilp32"
export BUILD32="-mabi=ilp32"
export PATH=${SYSDIR}/cross-tools/bin:/bin:/usr/bin
export JOBS=-j8
unset CFLAGS
unset CXXFLAGS
EOF
```

　　这里设置了几个环境变量，下面简单介绍这些变量的含义：

* SYSDIR：方便引用“基础目录”，可以通过修改该变量所设置的路径来改变所使用的“基础目录”。
* BUILDIR：该变量指定的目录用来进行软件包编译过程使用的目录。
* DOWNLOADDIR：该变量指定的目录存放制作系统的过程中所需要的软件包及一些必要的补丁文件。
* CROSS_HOST:设置“主系统”所使用的架构系统描述词
* CROSS_TARGET：设置“目标系统”所使用的架构系统描述词。
* MABI:指定“目标系统”默认使用的ABI名称。
* BUILD32：设置编译“目标系统”中的软件包为32位ABI时使用的ABI参数。

　　设置好用户环境配置文件后通过source命令使环境设置生效，使用命令：

```sh
source ~/.bash_profile
```



#### 创建目标系统的目录结构

　　我们要制作的目标系统是常规的Linux/GNU系统，我们按照常规的Linux/GNU系统所使用的目录结构创建目标系统的目录，命令如下:

```sh
pushd ${SYSDIR}/sysroot
	mkdir -pv ./{boot,home,root,mnt,opt,srv,run}
	mkdir -pv ./etc/{opt,sysconfig,profile.d}
	mkdir -pv ./media/{floppy,cdrom}
	mkdir -pv ./usr/{,local/}{include,src}
	mkdir -pv ./usr/local/{bin,lib,sbin}
	mkdir -pv ./usr/{,local/}share/{color,dict,doc,info,locale,man}
	mkdir -pv ./usr/{,local/}share/{misc,terminfo,zoneinfo}
	mkdir -pv ./usr/{,local/}share/man/man{1..8}
	mkdir -pv ./var/{cache,local,log,mail,opt,spool}
	mkdir -pv ./var/lib/{color,misc,locate}
	mkdir -pv ./usr/{lib{,32},bin,sbin}
	ln -sfv usr/{lib{,32},bin,sbin} ./
	mkdir -pv ./lib/firmware
	mkdir -pv ./{dev,proc,sys}
	ln -sfv ../run ./var/run
	ln -sfv ../run/lock ./var/lock
	install -dv -m 1777 ./tmp ./var/tmp
	ln -sfv . ./boot/boot
popd
```
　　目标系统将存放在${SYSDIR}/sysroot目录中，所以以该目录为基础创建各种目录和链接文件。

### 2.3 下载软件包


　　为了使用最新的软件包构建目标系统，这可能需要从网络中下载软件包源代码及补丁文件，下载的文件建议存放在“downloads”目录中。

```sh
pushd ${SYSDIR}/downloads
```

　　然后可以使用wget工具下载相应版本的软件包，例如下载coreutils-9.7这个软件包，可使用命令：

```sh
	wget https://ftp.gnu.org/gnu/coreutils/coreutils-9.7.tar.xz
```

　　各软件包的下载地址见各软件包制作步骤的开始处。

```sh
popd
```


## 3 制作交叉工具链及相关工具

　　接下来就正式进入交叉工具链和相关工具的制作环节。

### 3.1 Linux内核头文件

* 制作步骤  
　　按以下步骤制作Linux内核头文件并安装到目标系统目录中。

```sh
pushd ${BUILDDIR}
git clone https://github.com/FlyGoat/linux.git --depth 1 -b b4/la32
pushd linux
	make mrproper
	make ARCH=loongarch INSTALL_HDR_PATH=dest headers_install
	find dest/include -name '.*' -delete
	mkdir -pv ${SYSDIR}/sysroot/usr/include
	cp -rv dest/include/* ${SYSDIR}/sysroot/usr/include
popd
popd
```

### 3.2 交叉编译器之Binutils

* 制作步骤  
　　按以下步骤制作交叉编译工具链中的Binutils并安装到存放交叉工具链的目录中。

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/binutils-gdb.git --depth 1 -b la32
pushd binutils-gdb
	rm -rf gdb* libdecnumber readline sim
	mkdir tools-build
	pushd tools-build
    	CC=gcc AR=ar AS=as \
	    ../configure --prefix=${SYSDIR}/cross-tools --build=${CROSS_HOST} --host=${CROSS_HOST} \
	                 --target=${CROSS_TARGET} --with-sysroot=${SYSDIR}/sysroot --disable-nls \
	                 --disable-static --enable-64-bit-bfd --disable-werror
    	make configure-host ${JOBS}
    	make ${JOBS}
    	make install-strip
    	cp -v ../include/libiberty.h ${SYSDIR}/sysroot/usr/include
    popd
popd
popd
```

### 3.3 GMP
　　https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz

　　制作交叉工具链中所使用的GMP软件包。

```sh
tar xvf ${DOWNLOADDIR}/gmp-6.3.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gmp-6.3.0
	./configure --prefix=${SYSDIR}/cross-tools --enable-cxx --disable-static
	make ${JOBS}
	make install
popd
```

### 3.4 MPFR
　　https://ftp.gnu.org/gnu/mpfr/mpfr-4.2.2.tar.xz

　　制作交叉工具链中所使用的MPFR软件包。

```sh
tar xvf ${DOWNLOADDIR}/mpfr-4.2.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/mpfr-4.2.2
	./configure --prefix=${SYSDIR}/cross-tools --disable-static --with-gmp=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.5 MPC
　　https://ftp.gnu.org/gnu/mpc/mpc-1.3.1.tar.gz

　　制作交叉工具链中所使用的MPC软件包。

```sh
tar xvf ${DOWNLOADDIR}/mpc-1.3.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/mpc-1.3.1 
	./configure --prefix=${SYSDIR}/cross-tools --disable-static --with-gmp=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.6 交叉编译器之GCC（精简版）

* 制作步骤  
　　制作交叉编译器中的GCC，第一次编译交叉工具链的GCC需要采用精简方式进行编译和安装，否则会因为缺少目标系统的C库而导致部分内容编译链接失败，制作过程如下：

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/gcc.git --depth 1 -b la32
pushd gcc
	mkdir tools-build
	pushd tools-build
		AR=ar LDFLAGS="-Wl,-rpath,${SYSDIR}/cross-tools/lib" \
		../configure --prefix=${SYSDIR}/cross-tools --build=${CROSS_HOST} --host=${CROSS_HOST} \
		             --target=${CROSS_TARGET} --disable-nls \
		             --with-mpfr=${SYSDIR}/cross-tools --with-gmp=${SYSDIR}/cross-tools \
		             --with-mpc=${SYSDIR}/cross-tools \
		             --with-newlib --disable-shared --with-sysroot=${SYSDIR}/sysroot \
		             --disable-decimal-float --disable-libgomp --disable-libitm \
		             --disable-libsanitizer --disable-libquadmath --disable-threads \
		             --disable-target-zlib --with-system-zlib --enable-checking=release \
		             --enable-languages=c
		make all-gcc all-target-libgcc ${JOBS}
		make install-strip-gcc install-strip-target-libgcc
	popd
popd
rm -rf gcc
popd
```

对于目标是LoongArch架构来说，目前有几个参数是需要特别注意的：  
* ```--with-newlib```，因为当前没有目标系统Glibc的支持，所以使用newlib来临时支援GCC的运行。    
* ```--disable-shared```，使用newlib需要配合该参数。    
* ```--with-sysroot```， 指定默认使用的sysroot路径，该参数对交叉工具链来说极其重要。    
* ```--enable-languages=c```，这次仅编译C语言的支持就可以了，因为当前没有目标系统的Glibc，只能制作精简版。


### 3.7 Automake
　　https://ftp.gnu.org/gnu/automake/automake-1.17.tar.xz

　　Automake软件包中提供了许多软件包集成用来生成Makefile文件的脚本，制作步骤如下：

```sh
tar xvf ${DOWNLOADDIR}/automake-1.17.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/automake-1.17
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

　　打上补丁并安装到交叉工具链的目录中，这样当后续有软件包需要更新脚本文件时就可以通过本次安装的Automake中的脚本文件来进行替换。

### 3.8 目标系统的Glibc
　　在制作并安装好交叉工具链的Binutils、精简版的GCC以及Linux内核的头文件后就可以编译目标系统的Glibc了，制作和安装步骤如下：

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/glibc.git --depth 1 -b la32
pushd glibc
    mkdir -v build-32
    pushd build-32
	BUILD_CC="gcc" CC="${CROSS_TARGET}-gcc ${BUILD32} ${CFLAGS} -Wno-implicit-function-declaration" \
        CXX="${CROSS_TARGET}-gcc ${BUILD32}" \
        AS="${CROSS_TARGET}-as" AR="${CROSS_TARGET}-ar" RANLIB="${CROSS_TARGET}-ranlib" \
        ../configure --prefix=/usr --host=${CROSS_TARGET} --build=${CROSS_HOST} \
                 --libdir=/usr/lib32 --libexecdir=/usr/lib32/glibc \
                 --with-binutils=${SYSDIR}/cross-tools/bin \
                 --with-headers=${SYSDIR}/sysroot/usr/include \
                 --enable-stack-protector=strong --enable-add-ons \
                 --disable-werror --disable-nscd libc_cv_slibdir=/usr/lib32 \
                 --enable-kernel=5.19.0
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
    popd
    mkdir -v build-locale
    pushd build-locale
	../configure --prefix=/usr --libdir=/usr/lib32 --libexecdir=/usr/lib32/glibc \
                 --enable-stack-protector=strong --enable-add-ons \
                 --disable-werror libc_cv_slibdir=/usr/lib32
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot localedata/install-locales
    popd
popd

```
　　```sed -i "s@5.19.0@4.15.0@g" sysdeps/unix/sysv/linux/loongarch/configure{,.ac}``` Glibc默认支持的内核为5.19以上版本，这里用来将内核版本的需求降低到4.15，以便使用较低版本的内核。

　　Glibc是目标系统的一部分，因此指定prefix等路径参数时是按照常规系统的路径进行设置的，所以必须在安装时指定DESTDIR来指定安装到存放目标系统的目录中。


### 3.9 Libxcrypt
	https://github.com/besser82/libxcrypt/releases/download/v4.4.38/libxcrypt-4.4.38.tar.xz
	较新的glibc中不再提供libcrypt库的支持，需要使用libxcrypt软件包来代替原glibc中的libcrypto库。

```sh
tar xvf ${DOWNLOADDIR}/libxcrypt-4.4.38.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libxcrypt-4.4.38
        patch -Np1 -i ${DOWNLOADDIR}/0001-fix-configure-error-under-loongarch32-architecture.patch
	CFLAGS="${CFLAGS} -Wno-error=stringop-overread" \
	./configure --prefix=/usr --host=${CROSS_TARGET} --build=${CROSS_HOST} \
                 --libdir=/usr/lib32 --disable-werror
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

### 3.10 交叉编译器之GCC（完整版）
　　完成目标系统的Glibc之后就可以着手制作交叉工具链中完整版的GCC了，制作步骤如下：

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/gcc.git --depth 1 -b la32
pushd gcc
	mkdir tools-build-all
	pushd tools-build-all
		AR=ar LDFLAGS="-Wl,-rpath,${SYSDIR}/cross-tools/lib" \
		../configure --prefix=${SYSDIR}/cross-tools --build=${CROSS_HOST} \
		             --host=${CROSS_HOST} --target=${CROSS_TARGET} \
		             --with-sysroot=${SYSDIR}/sysroot --with-mpfr=${SYSDIR}/cross-tools \
		             --with-gmp=${SYSDIR}/cross-tools --with-mpc=${SYSDIR}/cross-tools \
		             --enable-__cxa_atexit --enable-threads=posix --with-system-zlib \
		             --enable-libstdcxx-time --enable-checking=release --disable-multilib \
			     --enable-linker-build-id --with-linker-hash-style=gnu --enable-gnu-unique-object --enable-initfini-array --enable-gnu-indirect-function \
		             --enable-languages=c,c++,fortran,objc,obj-c++,lto
		make ${JOBS}
		make install-strip
	popd
popd
popd
```

在完成目标系统的Glibc之后编译gcc就可以增加和修改一些编译参数了，主要是如下：  
* 去掉了```--with-newlib```和```--disable-shared```，因为有Glibc，所以不再需要newlib了。  
* ```--enable-threads=posix```,可以设置线程支持了。    
* ```--enable-languages=c,c++,fortran,objc,obj-c++,lto```，可以支持更多的开发语言了。

### 3.11 File
　　https://astron.com/pub/file/file-5.46.tar.gz

　　File软件包的官方发布版已经集成了LoongArch的支持，可以识别出LoongArch架构的二进制文件，制作时使用5.40以上的版本。

```sh
tar xvf ${DOWNLOADDIR}/file-5.46.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/file-5.46
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.12 Autoconf
　　https://ftp.gnu.org/gnu/autoconf/autoconf-2.72.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/autoconf-2.72.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/autoconf-2.72
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.13 Autoconf-Archive
　　https://mirror.truenetwork.ru/gnu/autoconf-archive/autoconf-archive-2024.10.16.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/autoconf-archive-2024.10.16.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/autoconf-archive-2024.10.16
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.14 Libtool
　　https://ftp.gnu.org/gnu/libtool/libtool-2.5.4.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libtool-2.5.4.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libtool-2.5.4
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.15 Perl
　　https://www.cpan.org/src/5.0/perl-5.40.2.tar.xz

　　为了配合目标系统中编译Perl相关的软件包时能使用正确的路径，因此我们需要在交叉工具链中安装一个目标系统相同版本的Perl软件包。

```sh
tar xvf ${DOWNLOADDIR}/perl-5.40.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/perl-5.40.2
    sed -i "s@/usr/include@${SYSDIR}/cross-tools/include@g" ext/Errno/Errno_pm.PL
    CFLAGS="-D_LARGEFILE64_SOURCE" ./configure.gnu --prefix=${SYSDIR}/cross-tools \
            -Dprivlib=${SYSDIR}/cross-tools/lib/perl5/5.4x/core_perl \
            -Darchlib=${SYSDIR}/cross-tools/lib32/perl5/5.4x/core_perl \
            -Dsitelib=${SYSDIR}/cross-tools/lib/perl5/5.4x/site_perl \
            -Dsitearch=${SYSDIR}/cross-tools/lib32/perl5/5.4x/site_perl \
            -Dvendorlib=${SYSDIR}/cross-tools/lib/perl5/5.4x/vendor_perl \
            -Dvendorarch=${SYSDIR}/cross-tools/lib32/perl5/5.4x/vendor_perl
    make ${JOBS}
    make install
popd
```

### 3.16 XML-Parser
　　https://cpan.metacpan.org/authors/id/T/TO/TODDR/XML-Parser-2.47.tar.gz

　　给交叉工具链中的Perl提供XML-Parser软件包提供的Perl组件。

```sh
tar xvf ${DOWNLOADDIR}/XML-Parser-2.47.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/XML-Parser-2.47
    ${SYSDIR}/cross-tools/bin/perl Makefile.PL
    make ${JOBS}
    make install
popd
```

### 3.17 URI
　　https://www.cpan.org/authors/id/O/OA/OALDERS/URI-5.32.tar.gz

　　给交叉工具链中的Perl提供URI软件包提供的Perl组件。

```sh
tar xvf ${DOWNLOADDIR}/URI-5.32.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/URI-5.32
    ${SYSDIR}/cross-tools/bin/perl Makefile.PL
    make ${JOBS}
    make install
popd
```

### 3.18 Python3
　　https://www.python.org/ftp/python/3.13.3/Python-3.13.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/Python-3.13.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/Python-3.13.3
	CFLAGS="${CFLAGS} -fPIC"
	./configure --prefix=${SYSDIR}/cross-tools --with-platlibdir=lib32 \
	            --disable-shared --with-system-expat --with-system-ffi \
	            --with-ensurepip=yes --enable-optimizations \
	            ac_cv_broken_sem_getvalue=yes
	make ${JOBS}
	make install
	sed -i "s@-lutil @@g" ${SYSDIR}/cross-tools/bin/python3.13-config
	cp ${SYSDIR}/cross-tools/lib32/python3.13/_sysconfigdata__linux_{x86_64-linux-gnu,${CROSS_TARGET}}.py
	sed -i -e "/'CC'/s@'gcc@'${CROSS_TARGET}-gcc@g" \
	       -e "/'CXX'/s@'g++@'${CROSS_TARGET}-g++@g" \
	       -e "/'LDSHARED'/s@'gcc@'${CROSS_TARGET}-gcc@g" \
	       -e "/'SOABI'/s@-x86_64-linux-gnu@@g" \
	       -e "/'EXT_SUFFIX'/s@-x86_64-linux-gnu@@g" \
	       ${SYSDIR}/cross-tools/lib32/python3.13/_sysconfigdata__linux_${CROSS_TARGET}.py
popd
```

### 3.19 Setuptools
　　https://files.pythonhosted.org/packages/source/s/setuptools/setuptools-80.4.0.tar.gz

　　Setuptools软件包是Python的基础软件包之一。

```sh
tar xvf ${DOWNLOADDIR}/setuptools-80.4.0.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/setuptools-80.4.0
        ${SYSDIR}/cross-tools/bin/python3 setup.py build
        ${SYSDIR}/cross-tools/bin/python3 setup.py install
popd
```
	编译Setuptools软件包建议使用pip命令，但因为当前使用pip命令编译存在依赖问题，所以本次编译Setuptools软件包直接使用python命令运行setup.py脚本进行编译和安装。


### 3.20 Pip
　　https://github.com/pypa/pip/archive/25.1.1/pip-25.1.1.tar.gz

　　pip软件包是Python的基础软件包之一。

```sh
tar xvf ${DOWNLOADDIR}/pip-25.1.1.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/pip-25.1.1
        ${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
        ${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user pip
popd
```

### 3.21 Flit-Core
　　https://files.pythonhosted.org/packages/source/f/flit_core/flit_core-3.12.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/flit_core-3.12.0.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/flit_core-3.12.0
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
        ${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user flit-core
popd
```

### 3.22 Wheel
　　https://files.pythonhosted.org/packages/source/w/wheel/wheel-0.46.1.tar.gz

　　Wheel软件包是Python的基础软件包之一。

```sh
tar xvf ${DOWNLOADDIR}/wheel-0.46.1.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/wheel-0.46.1
        ${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
        ${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user wheel
popd
```

### 3.23 Setuptools

　　依赖关系满足后再次使用pip命令来重新编译和安装Setuptools软件包。

```sh
tar xvf ${DOWNLOADDIR}/setuptools-80.4.0.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/setuptools-80.4.0
        ${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
        ${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user setuptools
popd
```

### 3.24 MarkupSafe
　　https://files.pythonhosted.org/packages/source/m/markupsafe/markupsafe-3.0.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/markupsafe-3.0.2.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/markupsafe-3.0.2
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user MarkupSafe
popd
```

### 3.25 Jinja2
　　https://files.pythonhosted.org/packages/source/j/jinja2/jinja2-3.1.6.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/jinja2-3.1.6.tar.gz -C ${BUILD_DIRECTORY}
pushd ${BUILD_DIRECTORY}/jinja2-3.1.6
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --force-reinstall --no-user Jinja2
popd
```

### 3.26 Meson
　　https://github.com/mesonbuild/meson/archive/1.8.0/meson-1.8.0.tar.gz

　　目标系统中部分软件对meson有版本要求，我们在交叉工具链的环境中提供一个较高版本的meson。

```sh
tar xvf ${DOWNLOADDIR}/meson-1.8.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/meson-1.8.0
	${SYSDIR}/cross-tools/bin/python3 setup.py build
	${SYSDIR}/cross-tools/bin/python3 setup.py install
popd
```

### 3.27 Pkg-Config
　　https://distfiles.dereferenced.org/pkgconf/pkgconf-2.4.3.tar.xz

　　为了能在交叉编译目标系统的过程中使用目标系统中已经安装的“pc”文件，我们在交叉工具链的目录中安装一个专门用来从目标系统目录中的查询“pc”文件的pkg-config命令，制作过程如下：

```sh
tar xvf ${DOWNLOADDIR}/pkgconf-2.4.3.tar.xz -C ${BUILDDIR}/
pushd ${BUILDDIR}/pkgconf-2.4.3
        ./configure --prefix=${SYSDIR}/cross-tools --build=${CROSS_HOST} \
                --host=${CROSS_HOST} --target=${CROSS_TARGET}
	make ${JOBS}
	make install
        ln -sf pkgconf ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-pkg-config
popd
```

### 3.28 Groff
　　https://ftp.gnu.org/gnu/groff/groff-1.23.0.tar.gz

　　编译目标系统的过程中会对Groff版本有一定要求，因此在交叉工具链的目录中安装一个版本较新的Groff。

```sh
tar xvf ${DOWNLOADDIR}/groff-1.23.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/groff-1.23.0
	PAGE=A4 ./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.29 Guile
　　https://ftp.gnu.org/gnu/guile/guile-3.0.10.tar.xz

　　编译目标系统的过程中会需要用到Guile软件包提供的工具，在交叉工具链的目录中安装一个版本较新的Guile。

```sh
tar xvf ${DOWNLOADDIR}/guile-3.0.10.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/guile-3.0.10
    ./configure --prefix=${SYSDIR}/cross-tools
    make ${JOBS}
    make install
popd
```

### 3.30 Make
　　https://ftp.gnu.org/gnu/make/make-4.4.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/make-4.4.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/make-4.4.1
	./configure --prefix=${SYSDIR}/cross-tools
	make ${JOBS}
	make install
popd
```

### 3.31 LLVM

```sh
pushd ${BUILDDIR}
git clone https://github.com/heiher/llvm-project.git --depth 1 -b la32
pushd llvm-project
    mkdir -p native-build
    pushd native-build
        LDFLAGS="${LDFLAGS} -lutil" PKG_CONFIG_SYSROOT_DIR="" \
        cmake .. -G Ninja -DCMAKE_INSTALL_PREFIX:PATH=${SYSDIR}/cross-tools \
                 -DCMAKE_CXX_COMPILER="g++" -DCMAKE_C_COMPILER="gcc" \
                 -DBUILD_SHARED_LIBS:BOOL=OFF   -DCMAKE_BUILD_TYPE=Release  \
                 -DCMAKE_C_FLAGS="-DNDEBUG" -DCMAKE_CXX_FLAGS="-DNDEBUG" \
                 -DLLVM_ENABLE_LIBCXX:BOOL=OFF \
		 -DLLVM_ENABLE_TERMINFO:BOOL=OFF \
                 -DLLVM_ENABLE_RTTI:BOOL=ON -DLLVM_BUILD_LLVM_DYLIB:BOOL=ON  \
                 -DLLVM_LINK_LLVM_DYLIB:BOOL=ON  \
                 -DCMAKE_INSTALL_RPATH="${SYSDIR}/cross-tools/lib32;\\\${ORIGIN}/../lib32" \
                 -DLLVM_BUILD_EXTERNAL_COMPILER_RT:BOOL=ON   \
                 -DLLVM_INSTALL_TOOLCHAIN_ONLY:BOOL=OFF \
		 -DDEFAULT_SYSROOT:PATH="${SYSDIR}/sysroot" \
                 -DLLVM_ENABLE_PROJECTS='llvm;clang;lld;lldb' \
		 -DCLANG_DEFAULT_LINKER=lld \
                 -DLLVM_DEFAULT_TARGET_TRIPLE=${CROSS_TARGET}
        ninja
        ninja install
    popd
popd
rm -rf llvm-project
popd
```

### 3.32 QEMU

```sh
pushd ${BUILDDIR}
git clone https://github.com/loongson-community/qemu.git --depth 1 -b la32-user-exp
pushd qemu
    sed -i "/HAVE_OPENAT2_H/d" meson.build
    mkdir build
    pushd build
	LDFLAGS="" \
        ../configure --prefix=${SYSDIR}/cross-tools --target-list=loongarch32-linux-user --disable-docs --static
        ninja
        cp qemu-loongarch32 ${SYSDIR}/cross-tools/bin/qemu-loongarch32.bin
    popd
popd
popd

echo "#!/bin/bash -e
${SYSDIR}/cross-tools/bin/qemu-loongarch32.bin -L ${SYSDIR}/sysroot -E LD_LIBRARY_PATH=.libs:${SYSDIR}/sysroot/usr/lib32 \"\$@\"" \
> ${SYSDIR}/cross-tools/bin/qemu-loongarch32

echo "#!/bin/bash -e
${SYSDIR}/cross-tools/bin/qemu-loongarch32.bin -L ${SYSDIR}/sysroot -E LD_TRACE_LOADED_OBJECTS=1 -E LD_LIBRARY_PATH=${SYSDIR}/sysroot/usr/lib32 \"\$@\"" \
> ${SYSDIR}/cross-tools/bin/qemu-loongarch32-ldd

chmod +x ${SYSDIR}/cross-tools/bin/qemu-loongarch32{,-ldd}

```
　　上面步骤中创建了两个脚本命令qemu-loongarch32和qemu-loongarch32-ldd，前者可以执行目标系统中的二进制程序，后者可以查看目标系统的二进制程序或库文件需要的动态链接库。

### 3.33 Glib
　　https://download.gnome.org/sources/glib/2.84/glib-2.84.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/glib-2.84.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/glib-2.84.1
    mkdir native-build
    pushd native-build
        meson --prefix=${SYSDIR}/cross-tools --libdir=${SYSDIR}/cross-tools/lib64 \
	      --default-library=both -Dtests=false -Dintrospection=disabled \
              --buildtype=release ..
        ninja
        ninja install
    popd
popd
```

### 3.34 Gobject-Introspection
　　https://download.gnome.org/sources/gobject-introspection/1.84/gobject-introspection-1.84.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gobject-introspection-1.84.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gobject-introspection-1.84.0
    mkdir native-build
    pushd native-build
        meson --prefix=${SYSDIR}/cross-tools --libdir=${SYSDIR}/cross-tools/lib64 \
              --buildtype=release ..
        ninja
        ninja install
    popd
popd

cat > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-g-ir-scanner << EOF
#!/bin/bash -e
LD_LIBRARY_PATH=${SYSDIR}/cross-tools/lib64/ \
${SYSDIR}/cross-tools/bin/g-ir-scanner \
      --add-include-path=${SYSDIR}/sysroot/usr/share/gir-1.0 \
      --use-binary-wrapper=${SYSDIR}/cross-tools/bin/qemu-loongarch32 \
      --use-ldd-wrapper=${SYSDIR}/cross-tools/bin/qemu-loongarch32-ldd "\$@"
EOF
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-g-ir-scanner

cat > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-g-ir-compiler << EOF
#!/bin/bash -e
LD_LIBRARY_PATH=${SYSDIR}/cross-tools/lib64/ \
${SYSDIR}/cross-tools/bin/g-ir-compiler \
          --includedir=/${SYSDIR}/sysroot/usr/share/gir-1.0 "\$@"
EOF
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-g-ir-compiler

```
　　为了在给目标系统中的一些软件正确生成对应的gir文件，我们在交叉编译目录中增加脚本命令来完成这个目标。

### 3.35 Vala
　　https://download.gnome.org/sources/vala/0.56/vala-0.56.18.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/vala-0.56.18.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/vala-0.56.18
    ./configure --prefix=${SYSDIR}/cross-tools --disable-valadoc
    make ${JOBS}
    make install
popd
cat > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-vapigen << EOF
#!/bin/bash -e
${SYSDIR}/cross-tools/bin/vapigen --vapidir=${SYSDIR}/sysroot/usr/share/vala/vapi \
                                  --girdir=${SYSDIR}/sysroot/usr/share/gir-1.0  "\$@"
EOF
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-vapigen

cat > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-valac << EOF
#!/bin/bash -e
${SYSDIR}/cross-tools/bin/valac --vapidir=${SYSDIR}/sysroot/usr/share/vala/vapi "\$@"
EOF
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-valac

```

## 4 制作目标系统
　　交叉工具链及其相关的工具制作并安装完成后就可以继续制作目标系统了。
　　
### 4.1 软件包制作说明

#### 架构测试脚本替换
　　在制作目标系统的过程中会可能遇到configure阶段提示不识别loongarch32架构的字样，这通常是软件包自带的架构探测脚本没有增加对loongarch32架构的识别，因此需要去对该问题进行处理，处理的方式通常有两种：  

　　1. 删除配置脚本，然后通过automake命令自动将新的探测脚本加入到软件包中，具体的操作方式为：  

```sh
rm config.guess config.sub
automake --add-missing
```

　　这里假定config.guess和config.sub两个脚本文件在软件包的第一级目录下，也可能是在build-aux之类的目录，找到文件并删除，然后使用automake命令的“--add-missing”参数来运行，该参数会自动确认是否缺少探测架构的脚本，如果缺少会从Automake软件包安装的目录中复制过来，automake运行的是我们在交叉工具链目录中的，已经增加了LoongArch架构的判断，这样软件包就可以正常运行了。

　　2.直接替换文件，具体的操作方式为：

```sh
cp ${SYSDIR}/sysroot/usr/share/automake-1.17/config.* config/
```

　　如果使用automake命令无法解决，可以直接复制Automake软件包安装的脚本文件，以我们安装的Automake-1.17版本为例，从${SYSDIR}/sysroot/usr/share/automake-1.17/中复制config开头的文件覆盖当前要编译的软件包中的同名文件即可，这里假定需要覆盖的文件在config目录中，也可能是在其它目录，可根据需要进行覆盖。

　　也可能一个软件包中有多个探测脚本，那么就需要全部进行覆盖。

#### 安装目标系统中的软件包

　　目标系统中的软件包安装的时候都是按照根目录（“/”）来存放的，所以如果直接安装那么就会安装到主系统的目录中，这必然是不对的，所以我们在安装的时候增加“DESTDIR”参数指定一个目录，那么安装的文件将以该指定目录作为安装的根目录进行安装，当我们设置为目标系统存放的目录时，就可以避开与主系统的冲突也可以让安装的软件包以正常的目录结构进行安装。

　　在后续的制作过程中会看到大多数软件包都支持“DESTDIR”参数设置，但也有个别软件包不支持，这种情况下通常采用软件包支持的类似功能参数来解决。

#### 交叉编译软件包
　　通常在带有configure配置脚本的软件包可以使用“build”、“host”参数来指定编译方式，当“build”和“host”相同时是本地编译，不同时就是交叉编译。

　　“build”参数可以理解为当前主系统所使用的架构系统信息，而“host”则是目标系统运行的架构系统信息，在本文中采用在x86的Linux系统中交叉编译LoongArch32架构的Linux系统，所以根据之前定义的环境变量，“build”指定为```${CROSS_HOST}```则代表了当前主系统，“host”指定为```${CROSS_TARGET}```则代表了要编译生成的目标架构系统。

　　由于是交叉编译，所以在软件包的配置阶段有可能探测的参数错误，这可能导致编译出来的软件包不匹配目标架构系统，这时可以采用指定部分探测参数的方式来解决，通常可以采用创建config.cache文件，然后将一些需要探测的参数和取值写入到该文件中，例如：

```sh
cat > config.cache << "EOF"
    ac_cv_func_mmap_fixed_mapped=yes
    ......
EOF
```

　　然后在configure的参数中指定```--cache-file=config.cache```，这样就可以在探测这些参数时使用文件中设置的值而不是尝试去探测，这可以避免探测到错误的取值。

#### 编译目录
　　多数软件包可以在软件包自己的“根目录”中进行配置和编译，但也有一些软件包会建议创建一个新的目录来配置和编译，对这些需要创建目录进行编译的软件包，我们通常采用在该软件目录下创建一个“build”开头的目录，并在该目录中进行编译，这样便于使用完软件包后的清理工作。

#### 清除编译目录
　　在完成交叉工具的编译后，建议清理一下```${BUILDDIR}```目录，因为该目录中所有文件都是编译过程需要用的，可以简单的将其中所有文件和目录都删除，例如使用如下命令(批量删除时需要注意不要误删系统文件或者重要数据)：

```sh
    rm -rf ${SYSDIR}/build/*
```

　　注意保留build目录本身，因为接下来还需要用到。

#### 设置环境变量

```sh
export LDFLAGS="-Wl,-rpath-link=${SYSDIR}/sysroot/usr/lib32"
export PKG_CONFIG_SYSROOT_DIR=${SYSDIR}/sysroot
export PKG_CONFIG_PATH="${SYSDIR}/sysroot/usr/lib32/pkgconfig:${SYSDIR}/sysroot/usr/share/pkgconfig"
export JOBS="-j8"
```

　　接下来的步骤是制作一个为meson命令用来交叉编译配置的文本文件，步骤如下：

```sh
pushd ${BUILDDIR}
echo "[binaries]" > meson-cross.txt
echo "c = '${CROSS_TARGET}-gcc'" >> meson-cross.txt
echo "cpp = '${CROSS_TARGET}-g++'" >> meson-cross.txt
echo "ar = '${CROSS_TARGET}-ar'" >> meson-cross.txt
echo "strip = '${CROSS_TARGET}-strip'" >> meson-cross.txt
echo "objcopy = '${CROSS_TARGET}-objcopy'" >> meson-cross.txt
echo "pkgconfig = '${CROSS_TARGET}-pkg-config'" >> meson-cross.txt
echo "cups-config = '${CROSS_TARGET}-cups-config'" >> meson-cross.txt
echo "llvm-config = '${CROSS_TARGET}-llvm-config'" >> meson-cross.txt
echo "vala = '${CROSS_TARGET}-valac'" >> meson-cross.txt
echo "exe_wrapper = 'qemu-loongarch32'" >> meson-cross.txt
echo "[properties]" >> meson-cross.txt
echo "sys_root = '${SYSDIR}/sysroot'" >> meson-cross.txt
echo "pkg_config_libdir = '${SYSDIR}/sysroot/usr/lib32/pkgconfig:${SYSDIR}/sysroot/usr/share/pkgconfig'" >> meson-cross.txt
cat >> meson-cross.txt << "EOF"
[host_machine]
system = 'linux'
cpu_family = 'loongarch32'
cpu = 'loongarch32'
endian = 'little'
EOF
popd
```
　　以上步骤完成后将在${BUILDDIR}目录中生成一个meson-cross.txt文件，该文件包含了meson在进行交叉编译时目标架构的名字、系统、使用的工具链命令以及编译参数等等，这样在接下来使用meson进行配置的软件包在配置阶段中引用该文件就可以了。

　　我们再为cmake命令定义用来交叉编译配置的文本文件，步骤如下：

```sh
cat > ${BUILDDIR}/cmake-cross.cmake << EOF
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR loongarch32)

set(CMAKE_CROSSCOMPILING_EMULATOR qemu-loongarch32)
set(CMAKE_SYSROOT ${SYSDIR}/sysroot)
set(CMAKE_FIND_ROOT_PATH ${SYSDIR}/sysroot/usr)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
EOF
```
　　以上步骤完成后将在${BUILDDIR}目录中生成一个cmake-cross.txt文件，该文件包含了cmake在进行交叉编译配置时的目标架构名字、系统以及参数等等，这样在接下来使用cmake进行配置的软件包在配置阶段中引用该文件就可以了。



### 4.2 软件包的制作

#### Man-Pages
　　https://mirrors.edge.kernel.org/pub/linux/docs/man-pages/man-pages-6.14.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/man-pages-6.14.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/man-pages-6.14
	make prefix=/usr DESTDIR=${SYSDIR}/sysroot install -R
popd
```
　　Man-Pages软件包没有配置阶段，直接安装到目标系统的目录中即可。

#### Iana-Etc
　　https://github.com/Mic92/iana-etc/releases/download/20250505/iana-etc-20250505.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/iana-etc-20250505.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/iana-etc-20250505
	cp -v services protocols ${SYSDIR}/sysroot/etc
popd
```
　　Iana-Etc软件包无需配置编译，只要将包含的文件复制到目标系统的目录中即可。

#### TZ-Data
　　https://data.iana.org/time-zones/releases/tzdata2025b.tar.gz

```sh
mkdir ${BUILDDIR}/tzdata-2025
tar xvf ${DOWNLOADDIR}/tzdata2025b.tar.gz -C ${BUILDDIR}/tzdata-2025
pushd ${BUILDDIR}/tzdata-2025
    ZONEINFO=${SYSDIR}/sysroot/usr/share/zoneinfo
    mkdir -pv $ZONEINFO/{posix,right}
    for tz in etcetera southamerica northamerica europe africa antarctica  \
              asia australasia backward; do
        /sbin/zic -L /dev/null   -d $ZONEINFO       ${tz}
        /sbin/zic -L /dev/null   -d $ZONEINFO/posix ${tz}
        /sbin/zic -L leapseconds -d $ZONEINFO/right ${tz}
    done
    cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
    /sbin/zic -d $ZONEINFO -p Asia/Shanghai
    unset ZONEINFO
    ln -sfv /usr/share/zoneinfo/Asia/Shanghai ${SYSDIR}/sysroot/etc/localtime
popd
```
tzdata软件包安装的是一组时区文件，这些文件可以设置系统当前的时区，让系统了解哪些时间是正确的。

#### GMP
　　https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gmp-6.3.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gmp-6.3.0
	rm config.guess config.sub
	automake --add-missing
	CFLAGS="${CFLAGS} -std=gnu17" \
	ABI=standard ./configure --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --prefix=/usr --libdir=/usr/lib32 --enable-cxx
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/lib{gmp,gmpxx}.la
popd
```
　　GMP软件包自带的探测架构脚本不支持LoongArch，因此删除探测脚本并用automake命令重新安装探测脚本。

#### MPFR
　　https://ftp.gnu.org/gnu/mpfr/mpfr-4.2.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/mpfr-4.2.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/mpfr-4.2.2
	./configure --build=${CROSS_HOST} --host=${CROSS_TARGET} --prefix=/usr --libdir=/usr/lib32
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libmpfr.la
popd
```

#### MPC
　　https://ftp.gnu.org/gnu/mpc/mpc-1.3.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/mpc-1.3.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/mpc-1.3.1
	./configure --build=${CROSS_HOST} --host=${CROSS_TARGET} --prefix=/usr --libdir=/usr/lib32
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libmpc.la
popd
```

#### Zlib
　　https://zlib.net/zlib-1.3.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/zlib-1.3.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/zlib-1.3.1
	CROSS_PREFIX=${CROSS_TARGET}- ./configure --prefix=/usr --libdir=/usr/lib32
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

### Binutils
　　这次编译的Binutils是目标系统中使用的，在交叉编译阶段不会使用到它。

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/binutils-gdb.git --depth 1 -b la32
pushd binutils-gdb
	rm -rf gdb* libdecnumber readline sim
	mkdir cross-build
	pushd cross-build
		../configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		             --host=${CROSS_TARGET} --enable-shared --disable-werror \
		             --with-system-zlib --enable-64-bit-bfd
		make tooldir=/usr ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot tooldir=/usr install
	popd
popd
popd
```

### GCC
　　与上面编译的Binutils一样，这次编译的GCC也是在目标系统中使用的编译器，在交叉编译阶段不会使用到它，但是其提供的libgcc、libstdc++等库可以为后续软件包的编译提供链接用的库。

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/gcc.git --depth 1 -b la32
pushd gcc
	sed -i 's@\./fixinc\.sh@-c true@' gcc/Makefile.in
	mkdir cross-build
	pushd cross-build
		../configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		             --host=${CROSS_TARGET} --target=${CROSS_TARGET} \
		             --enable-__cxa_atexit --enable-threads=posix \
		             --with-system-zlib --enable-libstdcxx-time \
		             --enable-checking=release \
                             --disable-multilib \
		             --with-build-sysroot=${SYSDIR}/sysroot \
                             --enable-linker-build-id --with-linker-hash-style=gnu --enable-gnu-unique-object --enable-initfini-array --enable-gnu-indirect-function \
		             --enable-languages=c,c++,fortran,objc,obj-c++,lto
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
		ln -sv /usr/bin/cpp ${SYSDIR}/sysroot/lib
		ln -sv gcc ${SYSDIR}/sysroot/usr/bin/cc
	popd
popd
popd
```

　　因在目标系统中使用，所以编译的完整一些，将C、C++以及Fortran等语言的支持加上。

#### Bzip2
　　https://www.sourceware.org/pub/bzip2/bzip2-1.0.8.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/bzip2-1.0.8.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/bzip2-1.0.8
	sed -i.orig -e "/^all:/s/ test//" Makefile
	sed -i -e 's:ln -s -f $(PREFIX)/bin/:ln -s -f :' Makefile
	sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
	make CC=${CROSS_TARGET}-gcc -f Makefile-libbz2_so ${JOBS}
	make clean
	make CC=${CROSS_TARGET}-gcc ${JOBS}
	make PREFIX=${SYSDIR}/sysroot/usr install
	cp -v bzip2-shared ${SYSDIR}/sysroot/bin/bzip2
	cp -av libbz2.so* ${SYSDIR}/sysroot/lib32
	ln -sfv ../../lib32/libbz2.so.1.0 ${SYSDIR}/sysroot/usr/lib32/libbz2.so
	ln -sfv bzip2 ${SYSDIR}/sysroot/bin/bunzip2
	ln -sfv bzip2 ${SYSDIR}/sysroot/bin/bzcat
popd
```
　　制作Bzip2软件包的过程用了两次make来生成了共享库和静态库，将它们都安装到目标系统中。

　　由于Bzip2软件包没有configure的配置脚本，因此在编译的时候直接给make命令指定CC参数，该参数用来设置编译程序时使用的编译器命令名，这里设置了交叉编译器的命令名，使得接下来的编译采用交叉编译器进行。

　　安装Bzip2软件包时因没有DESTDIR参数用来设置安装根目录，所以在PREFIX参数中加入目标系统存放目录的路径。

#### XZ
　　https://github.com/tukaani-project/xz/releases/download/v5.8.1/xz-5.8.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/xz-5.8.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/xz-5.8.1
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm ${SYSDIR}/sysroot/usr/lib32/liblzma.la
popd
```

#### Zstd
　　https://github.com/facebook/zstd/releases/download/v1.5.7/zstd-1.5.7.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/zstd-1.5.7.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/zstd-1.5.7
	make CC="${CROSS_TARGET}-gcc" PREFIX=/usr LIBDIR=/usr/lib32 ${JOBS}
	make CC="${CROSS_TARGET}-gcc" PREFIX=/usr LIBDIR=/usr/lib32 DESTDIR=${SYSDIR}/sysroot install
popd
```


#### File
　　https://astron.com/pub/file/file-5.46.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/file-5.46.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/file-5.46
	./configure --prefix=/usr  --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Ncurses
　　https://ftp.gnu.org/gnu/ncurses/ncurses-6.5.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/ncurses-6.5.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/ncurses-6.5
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --with-shared --without-debug \
	            --without-normal --enable-pc-files --without-ada \
	            --with-pkg-config-libdir=/usr/lib32/pkgconfig --enable-widec \
	            --disable-stripping
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	
	for lib in ncurses form panel menu ; do
	    rm -vf                    ${SYSDIR}/sysroot/usr/lib32/lib${lib}.so
	    echo "INPUT(-l${lib}w)" > ${SYSDIR}/sysroot/usr/lib32/lib${lib}.so
	    ln -sfv ${lib}w.pc        ${SYSDIR}/sysroot/usr/lib32/pkgconfig/${lib}.pc
	done
	
	rm -vf  ${SYSDIR}/sysroot/usr/lib32/libcursesw.so
	echo "INPUT(-lncursesw)" > ${SYSDIR}/sysroot/usr/lib32/libcursesw.so
	ln -sfv libncurses.so      ${SYSDIR}/sysroot/usr/lib32/libcurses.so
	rm -fv ${SYSDIR}/sysroot/usr/lib32/libncurses++w.a
popd

cp -v ${SYSDIR}/sysroot/usr/bin/ncursesw6-config ${SYSDIR}/cross-tools/bin/
sed -i "s@-L\$libdir@@g" ${SYSDIR}/cross-tools/bin/ncursesw6-config
```
　　在安装完目标系统的Ncurses后，复制了一个ncursesw6-config脚本命令到交叉编译目录中，这是因为后续编译一些软件包时会调用该命令来获取安装到目标系统中的Nucrses库链接信息，而如果主系统中的库与目标系统中的库链接不一致可能导致链接失败，因此提供一个可以正确链接信息的脚本是有效的解决方案。


#### Readline
　　https://ftp.gnu.org/gnu/readline/readline-8.2.13.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/readline-8.2.13.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/readline-8.2.13
	sed -i '/MV.*old/d' Makefile.in
	sed -i '/{OLDSUFF}/c:' support/shlib-install
	rm support/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET} \
				--disable-static --with-curses
	make SHLIB_LIBS="-lncursesw" ${JOBS}
	make SHLIB_LIBS="-lncursesw" DESTDIR=${SYSDIR}/sysroot install
popd
```

　　由于交叉编译的原因，Redaline的配置脚本无法正确的探测目标系统中安装的Ncurses软件包，因此在配置中加入```--with-curses```参数保证加入Ncurses的支持以及在编译阶段加入```SHLIB_LIBS="-lncursesw"```以保证正确链接库文件。

#### M4
　　https://ftp.gnu.org/gnu/m4/m4-1.4.20.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/m4-1.4.20.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/m4-1.4.20
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} AUTOMAKE=automake ACLOCAL=aclocal
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### BC
　　https://github.com/gavinhoward/bc/archive/7.0.3/bc-7.0.3.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/bc-7.0.3.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/bc-7.0.3
	CC="${CROSS_TARGET}-gcc" HOSTCC="gcc" CFLAGS="${CFLAGS} -std=gnu17" ./configure --prefix=/usr
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Flex
　　https://github.com/westes/flex/files/981163/flex-2.6.4.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/flex-2.6.4.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/flex-2.6.4
	patch -Np1 -i ${DOWNLOADDIR}/0001-flex-2.6.4-fix-build-error-for-gcc-15.patch
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static ac_cv_func_malloc_0_nonnull=yes \
	            ac_cv_func_realloc_0_nonnull=yes
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	ln -sv flex ${SYSDIR}/sysroot/usr/bin/lex
popd
```

#### Attr
　　https://bigsearcher.com/mirrors/nongnu/attr/attr-2.5.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/attr-2.5.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/attr-2.5.1
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static --sysconfdir=/etc
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm ${SYSDIR}/sysroot/usr/lib32/libattr.la
popd
```

#### Acl
　　https://bigsearcher.com/mirrors/nongnu/acl/acl-2.3.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/acl-2.3.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/acl-2.3.2
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm ${SYSDIR}/sysroot/usr/lib32/libacl.la
popd
```

#### Libcap
　　https://mirrors.edge.kernel.org/pub/linux/libs/security/linux-privs/libcap2/libcap-2.76.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libcap-2.76.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libcap-2.76
	make CROSS_COMPILE="${CROSS_TARGET}-" BUILD_CC="gcc" GOLANG=no prefix=/usr lib=lib32 ${JOBS}
	make CROSS_COMPILE="${CROSS_TARGET}-" BUILD_CC="gcc" GOLANG=no prefix=/usr lib=lib32 \
		 DESTDIR=${SYSDIR}/sysroot install
popd
```

　　因为该软件包没有配置脚本，所以直接在make命令上增加指定编译器的参数```CROSS_COMPILE="${CROSS_TARGET}-"```，这里要注意CROSS_COMPILE指定的是交叉编译工具的前缀而不是具体命令名，这样在编译过程中各种编译、汇编和链接相关的命令都会自动加上这个指定的前缀。

　　另外在编译过程中会编译在主系统中运行的程序，这个时候不能使用交叉编译器编译，所以还需要指定```BUILD_CC="gcc"```这个参数来保证编译这些要运行的程序使用的是本地编译器。

#### Shadow
　　https://github.com/shadow-maint/shadow/archive/4.17.4/shadow-4.17.4.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/shadow-4.17.4.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/shadow-4.17.4
	sed -i 's/groups$(EXEEXT) //' src/Makefile.in
	find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
	find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
	find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;
	sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD YESCRYPT:' \
	    -e 's:/var/spool/mail:/var/mail:'                   \
	    -e '/PATH=/{s@/sbin:@@;s@/bin:@@}'                  \
	    -i etc/login.defs
	./configure --sysconfdir=/etc --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET} --with-{b,yes}crypt \
				--with-group-name-max-length=32 --with-libbsd=no
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot exec_prefix=/usr install
	mkdir -pv ${SYSDIR}/sysroot/etc/default
popd
```

　　该软件包修改了一些默认的设置，下面介绍以下主要修改的内容：  
　　1、将用户密码的加密模式从DES改为YESCRYPT，后者相对前者更难破解。
　　2、一些默认路径的修改。

#### Sed
　　https://ftp.gnu.org/gnu/sed/sed-4.9.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/sed-4.9.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/sed-4.9
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Pkg-Config
　　https://pkg-config.freedesktop.org/releases/pkg-config-0.29.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/pkg-config-0.29.2.tar.gz -C ${BUILDDIR}/
pushd ${BUILDDIR}/pkg-config-0.29.2
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --with-internal-glib --disable-host-tool \
	            glib_cv_stack_grows=yes glib_cv_uscore=no \
	            ac_cv_func_posix_getpwuid_r=yes ac_cv_func_posix_getgrgid_r=yes
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	mkdir -p ${SYSDIR}/cross-tools/tmp/bin
	ln -sf /bin/pkg-config ${SYSDIR}/cross-tools/tmp/bin/${CROSS_TARGET}-pkg-config
popd
```
　　在制作目标系统的Pkg-config软件包的配置过程中因无法探测目标系统中的部分配置设置而会导致配置失败，因此需要在configure阶段强制设置部分参数的取值。

#### PSmisc
　　https://sourceforge.net/projects/psmisc/files/psmisc//psmisc-23.7.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/psmisc-23.7.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/psmisc-23.7
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                ac_cv_func_malloc_0_nonnull=yes ac_cv_func_realloc_0_nonnull=yes
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Gettext
　　https://ftp.gnu.org/gnu/gettext/gettext-0.25.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gettext-0.25.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gettext-0.25
	sed -i "/hello-c++-kde/d" gettext-tools/examples/Makefile.in
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static \
	            --with-libncurses-prefix=${SYSDIR}/sysroot
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libgettext*.la
	rm -v ${SYSDIR}/sysroot/usr/lib32/libtextstyle.la
popd
```

#### Bison
　　https://ftp.gnu.org/gnu/bison/bison-3.8.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/bison-3.8.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/bison-3.8.2
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### TCL
　　https://sourceforge.net/projects/tcl/files/Tcl//8.6.14//tcl8.6.14-src.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/tcl8.6.14-src.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/tcl8.6.14
    SRCDIR=$(pwd)
    pushd unix
	    ./configure --prefix=/usr --libdir=/usr/lib32 --mandir=/usr/share/man \
	                --build=${CROSS_HOST} --host=${CROSS_TARGET} --enable-64bit
	    make ${JOBS}
	    sed -i -e "s|$SRCDIR/unix|${SYSDIR}/sysroot/usr/lib32|" \
	           -e "s|$SRCDIR|${SYSDIR}/sysroot/usr/include|" \
	           -e "/TCL_INCLUDE_SPEC/s|/usr/include|${SYSDIR}/sysroot/usr/include|" \
	           tclConfig.sh
	    sed -i -e "s|$SRCDIR/unix/pkgs/tdbc1.1.3|${SYSDIR}/sysroot/usr/lib32/tdbc1.1.3|" \
               -e "s|$SRCDIR/pkgs/tdbc1.1.3/generic|${SYSDIR}/sysroot/usr/include|"    \
               -e "s|$SRCDIR/pkgs/tdbc1.1.3/library|${SYSDIR}/sysroot/usr/lib32/tcl8.6|" \
               -e "s|$SRCDIR/pkgs/tdbc1.1.3|${SYSDIR}/sysroot/usr/include|"            \
               pkgs/tdbc1.1.3/tdbcConfig.sh
	    sed -i -e "s|$SRCDIR/unix/pkgs/itcl4.2.2|${SYSDIR}/sysroot/usr/lib32/itcl4.2.2|" \
               -e "s|$SRCDIR/pkgs/itcl4.2.2/generic|${SYSDIR}/sysroot/usr/include|"    \
               -e "s|$SRCDIR/pkgs/itcl4.2.2|${SYSDIR}/sysroot/usr/include|"            \
               pkgs/itcl4.2.2/itclConfig.sh
	    unset SRCDIR
	    make DESTDIR=${SYSDIR}/sysroot install
	    make DESTDIR=${SYSDIR}/sysroot install-private-headers
	    ln -sfv tclsh8.6 ${SYSDIR}/sysroot/usr/bin/tclsh
	popd
popd
```

#### Expect
　　https://sourceforge.net/projects/expect/files/Expect//5.45.4//expect5.45.4.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/expect5.45.4.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/expect5.45.4
	patch -Np1 -i ${DOWNLOADDIR}/0001-enable-cross-compilation.patch
	autoreconf -ifv
	CFLAGS="${CFLAGS} -Wno-implicit-function-declaration -Wno-incompatible-pointer-types -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	            --with-tcl=${SYSDIR}/sysroot/usr/lib32 \
	            --enable-shared
	make ${JOBS}
	make TCLSH_PROG=/usr/bin/tclsh DESTDIR=${SYSDIR}/sysroot install
	ln -svf expect5.45.4/libexpect5.45.4.so ${SYSDIR}/sysroot/usr/lib32
popd
```

#### Dejagnu
　　https://ftp.gnu.org/gnu/dejagnu/dejagnu-1.6.3.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/dejagnu-1.6.3.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/dejagnu-1.6.3
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Grep
　　https://ftp.gnu.org/gnu/grep/grep-3.12.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/grep-3.12.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/grep-3.12
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Bash
　　https://ftp.gnu.org/gnu/bash/bash-5.2.37.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/bash-5.2.37.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/bash-5.2.37
cat > config.cache << "EOF"
	ac_cv_func_mmap_fixed_mapped=yes
	ac_cv_func_strcoll_works=yes
	ac_cv_func_working_mktime=yes
	bash_cv_func_sigsetjmp=present
	bash_cv_getcwd_malloc=yes
	bash_cv_job_control_missing=present
	bash_cv_printf_a_format=yes
	bash_cv_sys_named_pipes=present
	bash_cv_ulimit_maxfds=yes
	bash_cv_under_sys_siglist=yes
	bash_cv_unusable_rtsigs=no
	gt_cv_int_divbyzero_sigfpe=yes
EOF

	CFLAGS_FOR_BUILD="-std=gnu17" CFLAGS="-std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --without-bash-malloc \
	            --with-installed-readline --cache-file=config.cache
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	ln -sv bash ${SYSDIR}/sysroot/bin/sh
popd
```

　　Bash软件在交叉编译时的配置阶段会有大量的参数探测错误，需要我们手工指定这些参数的真实取值，创建一个文本文件，将这些参数的取值写进去，并在configure配置中增加```--cache-file=config.cache```参数（其中config.cache就是保存参数的文本文件名）。

#### Bash-Completion
　　https://github.com/scop/bash-completion/releases/download/2.16.0/bash-completion-2.16.0.tar.xz

　　bash-completion软件包是bash的扩展包，可以帮助完善bash使用过程中命令参数的补全。

```sh
tar xvf ${DOWNLOADDIR}/bash-completion-2.16.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/bash-completion-2.16.0
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
                    --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libtool
　　https://ftp.gnu.org/gnu/libtool/libtool-2.5.4.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libtool-2.4.7.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libtool-2.4.7
        ./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
        make ${JOBS}
        make DESTDIR=${SYSDIR}/sysroot install
popd
sed -i -e "s@${SYSDIR}/cross-tools/${CROSS_TARGET}@/usr@g" \
       -e "s@${SYSDIR}/cross-tools/lib/gcc@/usr/lib32/gcc@g" \
       -e "s@${SYSDIR}/cross-tools/bin@/usr/bin@g" \
       -e "s@${SYSDIR}/sysroot/lib@/usr/lib@g" \
       -e "s@${SYSDIR}/sysroot/usr@/usr@g" \
       -e "s@${CROSS_TARGET}-@@g" \
       ${SYSDIR}/sysroot/usr/bin/libtool
```
　　交叉编译生成的libtool文件中包含了大量的在交叉编译中使用的路径，因此在目标系统中使用libtool会存在路径不对的问题，我们通过sed命令将这些交叉编译环境中的路径转换成目标系统实际的路径。

#### GDBM
　　https://ftp.gnu.org/gnu/gdbm/gdbm-1.25.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/gdbm-1.25.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/gdbm-1.25
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static --enable-libgdbm-compat
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm ${SYSDIR}/sysroot/usr/lib32/libgdbm*.la
popd
```

#### gperf
　　https://ftp.gnu.org/gnu/gperf/gperf-3.3.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/gperf-3.3.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/gperf-3.3
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Expat
　　https://sourceforge.net/projects/expat/files/expat/2.7.1/expat-2.7.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/expat-2.7.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/expat-2.7.1
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --without-docbook
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libexpat.la
popd
```

#### Autoconf
　　https://ftp.gnu.org/gnu/autoconf/autoconf-2.72.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/autoconf-2.72.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/autoconf-2.72
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Autoconf-Archive
　　https://mirror.truenetwork.ru/gnu/autoconf-archive/autoconf-archive-2024.10.16.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/autoconf-archive-2024.10.16.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/autoconf-archive-2024.10.16
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Automake
　　https://ftp.gnu.org/gnu/automake/automake-1.17.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/automake-1.17.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/automake-1.17
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

　　在交叉编译目录中我们安装了一个Automake软件包，该软件包提供了增加LoongArch支持的探测脚本，有很多软件都会需要用这些脚本来覆盖自己源代码中的脚本。

　　在制作的目标系统中当然也需要改其中的Automake软件包，也使其支持LoongArch，这样将来在目标系统中配置编译一些软件包时就可以使用上。

#### Kmod
　　https://mirrors.edge.kernel.org/pub/linux/utils/kernel/kmod/kmod-34.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/kmod-34.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/kmod-34.2
	rm m4/l*.m4
	./autogen.sh
	./configure --prefix=/usr --libdir=/usr/lib32 --bindir=/bin \
	            --sysconfdir=/etc --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	            --with-xz --with-zstd --with-zlib --disable-manpages \
	            --with-bashcompletiondir=/usr/share/bash-completion/completions \
		    --with-fishcompletiondir=/usr/share/fish/vendor_completions.d \
		    as_ln_s="ln -sf"
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install

	for target in depmod insmod lsmod modinfo modprobe rmmod; do
		ln -sfv ../bin/kmod ${SYSDIR}/sysroot/sbin/$target
	done
	ln -sfv kmod ${SYSDIR}/sysroot/bin/lsmod
popd
```

#### Elfutils
　　https://sourceware.org/ftp/elfutils/0.193/elfutils-0.193.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/elfutils-0.193.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/elfutils-0.193
	sed -i.orig "s@GENDIS_ENV) \./i386_gendis@GENDIS_ENV) qemu-loongarch32 \./i386_gendis@g" libcpu/Makefile.am
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
				--host=${CROSS_TARGET} --disable-debuginfod --enable-libdebuginfod=dummy --enable-maintainer-mode \
				ac_cv_null_dereference=no
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
echo '#!/bin/bash -e
qemu-loongarch32 /opt/mylaos/sysroot/usr/bin/eu-readelf "$@"' > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-eu-readelf
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-eu-readelf
```

　　该软件包使用交叉编译会有个别功能探测错误，使用指定参数和取值的方式来解决，该制作步骤上采用了另一种设置参数取值的方式，若要指定的参数数值不多的情况下可以直接在configure的参数中进行设置,如```ac_cv_null_dereference=no```这就是这种设置方式，也可以通过将这两个参数写到“config.cache”，然后通过“--cache-file=config.cache”来使用。

#### Libffi

```sh
pushd ${BUILDDIR}
git clone https://github.com/cloudspurs/libffi.git --depth 1 -b la32
pushd libffi
	CFLAGS="${CFLAGS} -Wno-implicit-function-declaration -mcmodel=medium" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static --with-gcc-arch=native
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	cp -a ${CROSS_TARGET}/.libs/libffi_convenience.a ${SYSROOT_DIR}/usr/lib32/libffi_pic.a
popd
popd
```

#### OpenSSL
　　https://github.com/openssl/openssl/releases/download/openssl-3.5.0/openssl-3.5.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/openssl-3.5.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/openssl-3.5.0
	CC="${CROSS_TARGET}-gcc" \
	./Configure --prefix=/usr --openssldir=/etc/ssl \
				--libdir=lib32 shared zlib linux-generic32
	make -j${JOBS} EX_LIBS="-latomic"
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

　　OpenSSL是一个十分重要的安全算法库，通常对不同的架构可以使用汇编对算法进行优化，但其也提供了通用的C实现，因此可以采用```linux-generic32```来指定用通用实现进行编译，当然通用实现的性能是相对较低的，在今后如果有了针对LoongArch32的优化支持则可以修改该参数来达到优化编译的目的。

#### Coreutils
　　https://ftp.gnu.org/gnu/coreutils/coreutils-9.7.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/coreutils-9.7.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/coreutils-9.7
	FORCE_UNSAFE_CONFIGURE=1 \
	./configure --prefix=/usr  --build=${CROSS_HOST} --host=${CROSS_TARGET} \
				--enable-no-install-program=kill,uptime
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	mv -v ${SYSDIR}/sysroot/usr/bin/chroot ${SYSDIR}/sysroot/usr/sbin
popd
```

#### Check
　　https://github.com/libcheck/check/archive/0.15.2/check-0.15.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/check-0.15.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/check-0.15.2
	./configure --prefix=/usr --libdir=/usr/lib32 \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Diffutils
　　https://ftp.gnu.org/gnu/diffutils/diffutils-3.12.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/diffutils-3.12.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/diffutils-3.12
	./configure --prefix=/usr  --build=${CROSS_HOST} --host=${CROSS_TARGET} gl_cv_func_strcasecmp_works=yes
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Gawk
　　https://ftp.gnu.org/gnu/gawk/gawk-5.3.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gawk-5.3.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gawk-5.3.2
	sed -i 's/extras//' Makefile.in
	./configure --prefix=/usr  --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Findutils
　　https://ftp.gnu.org/gnu/findutils/findutils-4.10.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/findutils-4.10.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/findutils-4.10.0
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --localstatedir=/var/lib/locate
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Intltool
　　https://launchpad.net/intltool/trunk/0.51.0/+download/intltool-0.51.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/intltool-0.51.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/intltool-0.51.0
    sed -i 's:\\\${:\\\$\\{:' intltool-update.in
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	sed -i "s@${SYSDIR}/cross-tools@@g" ${SYSDIR}/sysroot/usr/bin/intltool*
popd
```

#### Groff
　　https://ftp.gnu.org/gnu/groff/groff-1.23.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/groff-1.23.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/groff-1.23.0
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	PAGE=A4 ./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make TROFFBIN=troff GROFFBIN=groff GROFF_BIN_PATH= ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Less
　　https://www.greenwoodsoftware.com/less/less-668.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/less-688.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/less-688
	./configure --prefix=/usr --sysconfdir=/etc --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### gzip
　　https://ftp.gnu.org/gnu/gzip/gzip-1.14.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gzip-1.14.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gzip-1.14
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Iproute2
　　https://mirrors.edge.kernel.org/pub/linux/utils/net/iproute2/iproute2-6.14.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/iproute2-6.14.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/iproute2-6.14.0
	sed -i /ARPD/d Makefile
	rm -fv man/man8/arpd.8
	PKG_CONFIG=${CROSS_TARGET}-pkg-config \
	make CC="${CROSS_TARGET}-gcc" HOSTCC="gcc" KERNEL_INCLUDE=${SYSDIR}/sysroot/usr/include \
	     NETNS_RUN_DIR=/run/netns ${JOBS}
	PKG_CONFIG=${CROSS_TARGET}-pkg-config \
	make CC="${CROSS_TARGET}-gcc" HOSTCC="gcc" KERNEL_INCLUDE=${SYSDIR}/sysroot/usr/include \
		 SBINDIR=/usr/sbin DESTDIR=${SYSDIR}/sysroot install
popd
```

　　IPRoute2软件包没有配置阶段，直接在make命令中使用“CC”变量指定交叉编译器，而对于在编译过程中会临时编译一些在本地运行的程序时就需要使用“HOSTCC”变量来指定本地编译器，否则“HOSTCC”会使用“CC”变量的指定编译器，那么编译出来的程序就无法在交叉编译的主系统中运行了。

#### Kbd
　　https://mirrors.edge.kernel.org/pub/linux/utils/kbd/kbd-2.7.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/kbd-2.7.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/kbd-2.7.1
	patch -Np1 -i ${DOWNLOADDIR}/0001-Save-font-support-UTF-8.patch
	sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
	sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
	autoreconf -ifv
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} --disable-vlock
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

　　交叉编译KBD时可能会缺少链接库而导致制作失败，此时可以通过LIBS变量指定缺少链接的库而完成KBD软件包的制作。

#### Libpipeline
　　https://download.savannah.gnu.org/releases/libpipeline/libpipeline-1.5.8.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libpipeline-1.5.8.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libpipeline-1.5.8
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Make
　　https://ftp.gnu.org/gnu/make/make-4.4.1.tar.gz

```sh
pushd ${BUILDDIR}/make-4.4.1
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Patch
　　https://ftp.gnu.org/gnu/patch/patch-2.8.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/patch-2.8.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/patch-2.8
	./configure --prefix=/usr -build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libunistring
　　https://ftp.gnu.org/gnu/libunistring/libunistring-1.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libunistring-1.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libunistring-1.3
	./configure --prefix=/usr --libdir=/usr/lib32 \
		    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libunistring.la
popd
```

#### Libpsl
　　https://github.com/rockdaboot/libpsl/releases/download/0.21.5/libpsl-0.21.5.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libpsl-0.21.5.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libpsl-0.21.5
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libpsl.la
popd
```

#### Curl
　　https://curl.se/download/curl-8.13.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/curl-8.13.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/curl-8.13.0
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
                --host=${CROSS_TARGET} --with-openssl \
                --enable-threaded-resolver --with-ca-path=/etc/ssl/certs
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	cp ${SYSDIR}/sysroot/usr/bin/curl-config ${SYSDIR}/cross-tools/bin/
popd
```

#### Libidn2
　　https://ftp.gnu.org/gnu/libidn/libidn2-2.3.8.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libidn2-2.3.8.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libidn2-2.3.8
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libidn2.la
popd
```

#### CMake
　　https://cmake.org/files/v4.0/cmake-4.0.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/cmake-4.0.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/cmake-4.0.2
    patch -Np1 -i ${DOWNLOADDIR}/0001-Fix-for-support-LoongArch32.patch
    mkdir build
    pushd build
        cmake -DCMAKE_CXX_COMPILER="${CROSS_TARGET}-g++" -DCMAKE_C_COMPILER="${CROSS_TARGET}-gcc" \
	      -DCMAKE_CXX_STANDARD_LIBRARIES="-latomic" \
              -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_DOC_DIR=/share/doc/cmake-4.0.2 \
              -DOPENSSL_ROOT_DIR=${SYSDIR}/sysroot/usr -DCMAKE_BUILD_TYPE=Release ../
        sed -i "/P cmake_install.cmake/s@\tbin/cmake@\t/bin/cmake@g" Makefile
        make ${JOBS}
        make DESTDIR=${SYSDIR}/sysroot install
    popd
popd
```

#### Python3

```sh
tar xvf ${DOWNLOADDIR}/Python-3.13.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/Python-3.13.3
	patch -Np1 -i ${DOWNLOADDIR}/0001-Python-3.13.1-fix-for-cross-build.patch
cat > config.cache << "EOF"
    ac_cv_aligned_required=no
    ac_cv_broken_sem_getvalue=no
    ac_cv_computed_gotos=yes
    ac_cv_pthread_is_default=yes
    ac_cv_pthread_system_supported=yes
    ac_cv_working_tzset=yes
    ac_cv_buggy_getaddrinfo=no
    ac_cv_file__dev_ptmx=yes
    ac_cv_file__dev_ptc=no
EOF
	./configure --prefix=/usr  --libdir=/usr/lib32 \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET} --enable-shared \
		    --with-build-python=${SYSDIR}/cross-tools/bin/python3 \
	            --with-system-expat --with-system-ffi --with-ensurepip=install \
	            --enable-optimizations --with-platlibdir=lib32 \
		    --with-openssl=${SYSDIR}/sysroot/usr \
	            --cache-file=config.cache
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
ln -sv python3 ${SYSDIR}/sysroot/usr/bin/python

cp -v ${SYSDIR}/sysroot/usr/bin/python3.13-config ${SYSDIR}/cross-tools/bin/
sed -i "/prefix_real/s@=.*@=${SYSDIR}/sysroot/usr@g" ${SYSDIR}/cross-tools/bin/python3.13-config

cp ${SYSDIR}/sysroot/usr/lib32/python3.13/_sysconfigdata__linux_loongarch32-linux-gnu.py ${SYSDIR}/cross-tools/lib32/python3.13/_sysconfigdata__linux_${CROSS_TARGET}.py
sed -i "/'INCLUDEPY'/s@'/usr/include@'${SYSDIR}/sysroot/usr/include@g" ${SYSDIR}/cross-tools/lib32/python3.13/_sysconfigdata__linux_${CROSS_TARGET}.py

sed -i -e "s@${SYSDIR}/sysroot@@g" \
       -e "s@${CROSS_TARGET}-@@g" \
       ${SYSDIR}/sysroot/usr/lib32/python3.13/_sysconfigdata__linux_loongarch32-linux-gnu.py

echo "#!/bin/bash -e
if [ -f ${SYSROOT_DIR}/usr/bin/python3 ]; then
    qemu-loongarch32 ${SYSROOT}/sysroot/usr/bin/python3 \"\$@\"
else
    _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} ${SYSDIR}/cross-tools/bin/python3 \"\$@\"
fi
" > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-python3
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-python3
```

#### Python-Setuptools

```sh
tar xvf ${DOWNLOADDIR}/setuptools-80.4.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/setuptools-80.4.0
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user setuptools --root=${SYSDIR}/sysroot --prefix=/usr
popd
```

#### Python-Pip

```sh
tar xvf ${DOWNLOADDIR}/pip-25.1.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/pip-25.1.1
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user pip --root=${SYSDIR}/sysroot --prefix=/usr
	sed -i "s@${SYSDIR}/cross-tools@@g" ${SYSDIR}/sysroot/bin/pip{,3{,.13}}
popd
```

#### Flit_Core

```sh
tar xvf ${DOWNLOADDIR}/flit_core-3.12.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/flit_core-3.12.0
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user flit_core --root=${SYSDIR}/sysroot --prefix=/usr
popd
```

#### Wheel

```sh
tar xvf ${DOWNLOADDIR}/wheel-0.46.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/wheel-0.46.1
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user wheel --root=${SYSDIR}/sysroot --prefix=/usr
popd
```

#### MarkupSafe
　　https://files.pythonhosted.org/packages/source/m/markupsafe/markupsafe-3.0.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/markupsafe-3.0.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/markupsafe-3.0.2
    CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
    ${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
    CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
    ${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user MarkupSafe --root=${SYSROOT_DIR} --prefix=/usr
popd
```

#### Jinja2
　　https://files.pythonhosted.org/packages/source/j/jinja2/jinja2-3.1.6.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/jinja2-3.1.6.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/jinja2-3.1.6
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user Jinja2 --root=${SYSDIR}/sysroot --prefix=/usr
popd
```

#### Man-DB
　　https://download.savannah.gnu.org/releases/man-db/man-db-2.13.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/man-db-2.13.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/man-db-2.13.1
	rm $(dirname $(find -name "config.sub"))/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --sysconfdir=/etc --disable-setuid \
	            --enable-cache-owner=bin 	--with-browser=/usr/bin/lynx \
	            --with-vgrind=/usr/bin/vgrind --with-grap=/usr/bin/grap
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Tar
　　https://ftp.gnu.org/gnu/tar/tar-1.35.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/tar-1.35.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/tar-1.35
	FORCE_UNSAFE_CONFIGURE=1 ./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Texinfo
　　https://ftp.gnu.org/gnu/texinfo/texinfo-7.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/texinfo-7.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/texinfo-7.2
sed -e 's/__attribute_nonnull__/__nonnull/' \
    -i gnulib/lib/malloc/dynarray-skeleton.c
	for i in $(dirname $(find -name "config.sub"))
	do
		rm ./$i/config.{sub,guess}
		pushd $(dirname ./$i)
			automake --add-missing
		popd
	done
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	make DESTDIR=${SYSDIR}/sysroot TEXMF=/usr/share/texmf install-tex
popd
```

#### SQLite
　　https://github.com/sqlite/sqlite/archive/version-3.49.2/sqlite-3.49.2.tar.gz

```sh
unzip ${DOWNLOADDIR}/sqlite-3.49.2.tar.gz -d ${BUILDDIR}
pushd ${BUILDDIR}/sqlite-3.49.2
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-fts5 --soname=legacy \
		CPPFLAGS="-DSQLITE_ENABLE_FTS3=1 \
                          -DSQLITE_ENABLE_FTS4=1 \
                          -DSQLITE_ENABLE_COLUMN_METADATA=1 \
                          -DSQLITE_ENABLE_UNLOCK_NOTIFY=1 \
                          -DSQLITE_ENABLE_DBSTAT_VTAB=1 \
                          -DSQLITE_SECURE_DELETE=1 \
                          -DSQLITE_ENABLE_FTS3_TOKENIZER=1"
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libsqlite3*.la
popd
```

#### Util-Linux
　　https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.41/util-linux-2.41.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/util-linux-2.41.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/util-linux-2.41
	cp ${SYSDIR}/sysroot/usr/share/automake-1.16/config.* config/
	./configure  --build=${CROSS_HOST} --host=${CROSS_TARGET} \
        ADJTIME_PATH=/var/lib/hwclock/adjtime \
        --libdir=/usr/lib32 \
        --disable-chfn-chsh --disable-login --disable-nologin \
        --disable-su --disable-setpriv --disable-runuser \
        --disable-pylibmount --disable-static --without-python \
        --without-systemd --disable-makeinstall-chown \
	--with-bashcompletiondir=/usr/share/bash-completion/completions \
	--with-tmpfilesdir=/usr/lib/tmpfiles.d --disable-asciidoc \
        runstatedir=/run
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/lib{blkid,fdisk,mount,smartcols,uuid}.la
popd
```

　　Util-Linux带有大量的命令和库，由于部分命令已经在其它软件包中提供了，所以使用选项参数来关闭这些命令的编译和安装。

#### Systemd
　　https://github.com/systemd/systemd/archive/v257.5/systemd-257.5.tar.gz
　　Systemd采用的是meson命令进行配置阶段的操作，meson与其他常见的configure脚本配置有明显的不同，在当前需要进行交叉编译的情况下要采用特定的配置操作步骤，以下将展开进行说明。

```sh
tar xvf ${DOWNLOADDIR}/systemd-257.5.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/systemd-257.5
	patch -Np1 -i ${DOWNLOADDIR}/0001-systemd-257-add-loongarch32-support.patch
	pushd src/basic
        python3 missing_syscalls.py missing_syscall_def.h $(ls syscalls-*.txt)
	popd
	sed -i -e 's/GROUP="render"/GROUP="video"/' \
	    -e 's/GROUP="sgx", //' rules.d/50-udev-default.rules.in
	sed -i "s@Werror=undef',@& '-Wno-error=format-overflow', '-Wno-error=implicit-function-declaration',@g" meson.build
```

　　以上步骤是为了更新LoongArch32架构的系统调用表，并且对代码进行必要的修正。

　　以下是配置和编译的步骤：

```sh
	mkdir -p build
	pushd build
		meson --prefix=/usr --libdir=/usr/lib32 --sysconfdir=/etc --localstatedir=/var \
		      -Dbuildtype=release -Ddefault-dnssec=no -Dfirstboot=false \
		      -Dinstall-tests=false -Dldconfig=false -Dsysusers=false \
		      -Drpmmacrosdir=no -Dhomed=false -Duserdb=false -Dman=false -Dmode=release \
		      -Dpamconfdir=/etc/pam.d \
		      -Ddbuspolicydir=/usr/share/dbus-1/system.d -Ddbussessionservicedir=/usr/share/dbus-1/services \
		      -Ddbussystemservicedir=/usr/share/dbus-1/system-services \
		      -Dbashcompletiondir=/usr/share/bash-completion/completions \
		      --cross-file ${BUILDDIR}/meson-cross.txt ..
		ninja
		DESTDIR=${SYSDIR}/sysroot ninja install
	popd
popd
```

　　较新版本的Systemd不再使用make命令进行编译了，配合meson使用ninja命令进行编译，在编译后同样用ninja命令安装软件包。

　　安装命令支持“DESTDIR”变量设置，但与make命令不同的是“DESTDIR”变量需要写在ninja命令的前面，安装的参数是“install”，该命令执行后同样将软件包安装到目标系统存放的目录中。

#### DBus
　　https://dbus.freedesktop.org/releases/dbus/dbus-1.16.2.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/dbus-1.16.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/dbus-1.16.2
	mkdir -p build
	pushd build
		meson --prefix=/usr --libdir=/usr/lib32 --sysconfdir=/etc --localstatedir=/var \
		      -Dbuildtype=release --cross-file ${BUILDDIR}/meson-cross.txt \
		      -Dsystemd_system_unitdir=/usr/lib/systemd/system -Dsystemd_user_unitdir=/usr/lib/systemd/user \
                      ..
	ninja
	DESTDIR=${SYSDIR}/sysroot ninja install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libdbus-1.la
	ln -sfv /etc/machine-id ${SYSDIR}/sysroot/var/lib/dbus
popd
```

　　DBus软件包使用meson作为配置阶段的工具，我们使用准备好的meson-cross.txt作为交叉编译的配置文件进行配置。

#### Procps-NG
　　https://sourceforge.net/projects/procps-ng/files/Production//procps-ng-4.0.5.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/procps-ng-4.0.5.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/procps-ng-4.0.5
	./configure --prefix=/usr --libdir=/usr/lib32  --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-static --disable-kill --with-systemd \
	            ac_cv_func_malloc_0_nonnull=yes ac_cv_func_realloc_0_nonnull=yes
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

　　Procps-ng软件包也是在交叉编译方式上会出现参数判断错误的情况，需要在配置阶段指定参数和取值。

#### E2fsprogs
　　https://sourceforge.net/projects/e2fsprogs/files/e2fsprogs/v1.47.2/e2fsprogs-1.47.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/e2fsprogs-1.47.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/e2fsprogs-1.47.2
	mkdir -v build
	pushd build
		CFLAGS="${CFLAGS} -std=gnu17" \
		../configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		             --host=${CROSS_TARGET} --sysconfdir=/etc \
		             --enable-elf-shlibs --disable-libblkid \
		             --disable-libuuid --disable-uuidd --disable-fsck
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
		rm -fv ${SYSDIR}/sysroot/usr/lib32/{libcom_err,libe2p,libext2fs,libss}.la
	popd
popd
cp -av ${SYSDIR}/sysroot/usr/bin/mk_cmds ${SYSDIR}/cross-tools/bin/
sed -i "s@=/usr@=${SYSDIR}/sysroot/usr@g" ${SYSDIR}/cross-tools/bin/mk_cmds
```

#### OpenSSH
　　https://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-10.0p1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/openssh-10.0p1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/openssh-10.0p1
	rm config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr  --libdir=/usr/lib32 --sysconfdir=/etc/ssh \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	            --disable-strip --with-md5-passwords \
	            --with-privsep-path=/var/lib/sshd \
	            --with-default-path=/usr/bin \
	            --with-superuser-path=/usr/sbin:/usr/bin \
	            --with-pid-dir=/run
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install-nokeys host-key
	install -v -m755 contrib/ssh-copy-id ${SYSDIR}/sysroot/usr/bin
popd
```

#### PCIUtils
　　https://mirrors.edge.kernel.org/pub/software/utils/pciutils/pciutils-3.13.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/pciutils-3.13.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/pciutils-3.13.0
	make CROSS_COMPILE="${CROSS_TARGET}-" HOST="${CROSS_TARGET}" \
	     PREFIX=/usr SHARED=yes LIBDIR=/usr/lib32 ${JOBS}
	make CROSS_COMPILE="${CROSS_TARGET}-" HOST="${CROSS_TARGET}" \
	     PREFIX=/usr SHARED=yes LIBDIR=/usr/lib32 STRIP="" \
	     DESTDIR=${SYSDIR}/sysroot install install-lib
popd
```

#### WGet
　　https://ftp.gnu.org/gnu/wget/wget-1.25.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/wget-1.25.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/wget-1.25.0
	rm build-aux/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --sysconfdir=/etc \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	            --with-ssl=openssl
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Make-CA
　　https://github.com/lfs-book/make-ca/archive/v1.16/make-ca-1.16.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/make-ca-1.16.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/make-ca-1.16
        make DESTDIR=${SYSDIR}/sysroot install
        install -vdm755 ${SYSDIR}/sysroot/etc/ssl/local
        cp ${SYSDIR}/sysroot/usr/sbin/make-ca ${SYSDIR}/cross-tools/bin/
        sed -i "/MAKE_CA_CONF/s@\"/etc@\"${SYSDIR}/sysroot/etc@g" ${SYSDIR}/cross-tools/bin/make-ca
        cp ${DOWNLOADDIR}/certdata.txt ./
        cp ${DOWNLOADDIR}/root.crt ./
        cp ${DOWNLOADDIR}/class3.crt ./
        ${SYSDIR}/cross-tools/bin/make-ca --destdir ${SYSDIR}/sysroot -C certdata.txt
        openssl x509 -in root.crt -text -fingerprint -setalias "CAcert Class 1 root" \
                -addtrust serverAuth -addtrust emailProtection -addtrust codeSigning \
                > ${SYSDIR}/sysroot/etc/ssl/local/CAcert_Class_1_root.pem
        openssl x509 -in class3.crt -text -fingerprint -setalias "CAcert Class 3 root" \
                -addtrust serverAuth -addtrust emailProtection -addtrust codeSigning \
                > ${SYSDIR}/sysroot/etc/ssl/local/CAcert_Class_3_root.pem
        ${SYSDIR}/cross-tools/bin/make-ca --destdir ${SYSDIR}/sysroot -r
popd

mkdir -pv ${SYSDIR}/sysroot/etc/profile.d &&
cat > ${SYSDIR}/sysroot/etc/profile.d/pythoncerts.sh << "EOF"
## Begin /etc/profile.d/pythoncerts.sh

export _PIP_STANDALONE_CERT=/etc/pki/tls/certs/ca-bundle.crt

## End /etc/profile.d/pythoncerts.sh
EOF
```

#### Inetutils
　　https://ftp.gnu.org/gnu/inetutils/inetutils-2.6.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/inetutils-2.6.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/inetutils-2.6
	sed -i "/PATH_PROCNET_DEV/s@no@/proc/net/dev@g" paths
	CFLAGS="${CFLAGS} -Wno-implicit-function-declaration" \
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	            --disable-logger --disable-whois --disable-rcp \
	            --disable-rexec --disable-rlogin --disable-rsh \
	            --disable-servers
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	mv -v ${SYSDIR}/sysroot/usr/{,s}bin/ifconfig
	chmod -v +x ${SYSDIR}/sysroot/usr/bin/{ping{,6},traceroute}
popd
```

#### DHCPCD
　　https://github.com/NetworkConfiguration/dhcpcd/archive/v10.2.2/dhcpcd-10.2.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/dhcpcd-10.2.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/dhcpcd-10.2.2
	./configure --prefix=/usr --sysconfdir=/etc --build=${CROSS_HOST} \
	            --host=${CROSS_TARGET} --disable-privsep
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Wireless_Tools
　　https://hewlettpackard.github.io/wireless-tools/wireless_tools.29.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/wireless_tools.29.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/wireless_tools.29
	patch -Np1 -i ${DOWNLOADDIR}/wireless_tools-29-fix_iwlist_scanning-1.patch
	sed  -i.orig "/^INSTALL_LIB/s@/lib/@/lib32/@g" Makefile
	make CC=${CROSS_TARGET}-gcc ${JOBS}
	make PREFIX=${SYSDIR}/sysroot/usr INSTALL_MAN=${SYSDIR}/sysroot/usr/share/man install
popd
```

#### Net-Tools
　　https://sourceforge.net/projects/net-tools/files//net-tools-2.10.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/net-tools-2.10.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/net-tools-2.10
	yes "" | make BINDIR='/usr/bin' SBINDIR='/usr/bin' CC=${CROSS_TARGET}-gcc
	make BINDIR='/usr/bin' SBINDIR='/usr/bin' CC=${CROSS_TARGET}-gcc DESTDIR=${PWD}/dest install
	rm -v dest/usr/bin/{nis,yp}domainname
	rm -v dest/usr/bin/{hostname,dnsdomainname,domainname,ifconfig}
	rm -rv dest/usr/share/man/man1
	rm -rv dest/usr/share/man/man8/ifconfig.8
	cp -av dest/usr/bin/* ${SYSDIR}/sysroot/usr/bin/
	cp -av dest/usr/share/man/man* ${SYSDIR}/sysroot/usr/share/man/
popd
```

#### Libnl
　　https://github.com/thom311/libnl/releases/download/libnl3_11_0/libnl-3.11.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libnl-3.11.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libnl-3.11.0
	rm build-aux/config.{sub,guess}
	automake --add-missing
	./configure --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib32 \
	            --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Sudo
　　https://www.sudo.ws/dist/sudo-1.9.16.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/sudo-1.9.16.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/sudo-1.9.16
	./configure --prefix=/usr --libexecdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--with-secure-path --with-all-insults --with-env-editor \
		--with-passprompt="[sudo] password for %p: "
	sed -i "/^install_uid/s@= 0@= $(id -u)@g" Makefile
	sed -i "/^install_gid/s@= 0@= $(id -u)@g" Makefile
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
sed -i "/wheel ALL=(ALL:ALL) ALL/s@# @@g" ${SYSDIR}/sysroot/etc/sudoers.dist
```

#### NSPR
　　https://archive.mozilla.org/pub/nspr/releases/v4.36/src/nspr-4.36.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/nspr-4.36.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/nspr-4.36/nspr
    patch -Np2 -i ${DOWNLOADDIR}/0001-nspr-add-loongarch32-support.patch
    cp ${SYSDIR}/cross-tools/share/automake-*/config.* build/autoconf/
    ./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
                --host=${CROSS_TARGET} --with-mozilla \
                --with-pthreads
    make CC="gcc" -C config
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### NSS
　　https://archive.mozilla.org/pub/security/nss/releases/NSS_3_111_RTM/src/nss-3.111.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/nss-3.111.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/nss-3.111/nss
    sed -i "s@ uname -m@ cross-uname -m@g" coreconf/arch.mk
    make CC="gcc" -C coreconf/nsinstall BUILD_OPT=1 \
         CPU_ARCH="loongarch64" CROSS_COMPILE=1 NSS_ENABLE_WERROR=0 OS_TEST="loongarch64" ${JOBS}
    make NATIVE_CC="gcc" CC="${CROSS_TARGET}-gcc" CCC="${CROSS_TARGET}-g++" \
         BUILD_OPT=1 CPU_ARCH="loongarch64" CROSS_COMPILE=1 \
         USE_SYSTEM_ZLIB=1 NSS_USE_SYSTEM_SQLITE=1 NSS_ENABLE_WERROR=0 \
         NSPR_INCLUDE_DIR=${SYSDIR}/sysroot/usr/include/nspr OS_TEST="loongarch64" ${JOBS}

    cat pkg/pkg-config/nss-config.in | sed -e "s,@prefix@,/usr,g" \
        -e "s,@MOD_MAJOR_VERSION@,$(cat lib/util/nssutil.h \
            | grep "#define.*NSSUTIL_VMAJOR" | awk '{print $3}'),g" \
        -e "s,@MOD_MINOR_VERSION@,$(cat lib/util/nssutil.h \
            | grep "#define.*NSSUTIL_VMINOR" | awk '{print $3}'),g" \
        -e "s,@MOD_PATCH_VERSION@,$(cat lib/util/nssutil.h \
            | grep "#define.*NSSUTIL_VPATCH" | awk '{print $3}'),g" \
        > ${SYSDIR}/sysroot/usr/bin/nss-config

    cat pkg/pkg-config/nss.pc.in | sed -e "s,%prefix%,/usr,g" \
        -e 's,%exec_prefix%,${prefix},g' -e "s,%libdir%,/usr/lib32,g" \
        -e 's,%includedir%,${prefix}/include/nss,g' \
        -e "s,%NSS_VERSION%,$(cat lib/util/nssutil.h \
            | grep "#define.*NSSUTIL_VERSION" | awk '{print $3}'),g" \
        -e "s,%NSPR_VERSION%,$(cat ${SYSDIR}/sysroot/usr/include/nspr/prinit.h \
            | grep "#define.*PR_VERSION" | awk '{print $3}'),g" \
        > ${SYSDIR}/sysroot/usr/lib32/pkgconfig/nss.pc
popd
pushd ${BUILDDIR}/nss-3.111/dist
    install -v -m755 Linux*/lib/*.so ${SYSDIR}/sysroot/usr/lib32
    install -v -m644 Linux*/lib/libcrmf.a ${SYSDIR}/sysroot/usr/lib32
    install -v -m755 -d ${SYSDIR}/sysroot/usr/include/nss
    cp -v -RL {public,private}/nss/* ${SYSDIR}/sysroot/usr/include/nss
    chmod -v 644 ${SYSDIR}/sysroot/usr/include/nss/*
    install -v -m755 Linux*/bin/{certutil,pk12util} ${SYSDIR}/sysroot/usr/bin
popd
```

#### ICU4C
　　https://github.com/unicode-org/icu/releases/download/release-77-1/icu4c-77_1-src.tgz

```sh
tar xvf ${DOWNLOADDIR}/icu4c-77_1-src.tgz -C ${BUILDDIR}
pushd ${BUILDDIR}/icu/source
	touch config/icucross.mk
	touch config/icucross.inc
	sed -i '/^PKGDATA/s@$(TOOLBINDIR)@/bin@g' data/Makefile.in
	sed -i '/INVOKE/s@$(TOOLBINDIR)@/bin@g' data/Makefile.in extra/uconv/Makefile.in
	sed -i '/INVOKE/s@/bin/icupkg@/sbin/icupkg@g' data/Makefile.in
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET} --with-cross-build=${PWD}
	sed -i '/INVOKE/s@$(TOOLBINDIR)@/bin@g' data/rules.mk
	sed -i '/INVOKE/s@/bin/icupkg@/sbin/icupkg@g' data/rules.mk
	sed -i '/INVOKE/s@/bin/gensprep@/sbin/gensprep@g' data/rules.mk
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Meson

```sh
tar xvf ${DOWNLOADDIR}/meson-1.8.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/meson-1.8.0
    ${SYSDIR}/cross-tools/bin/python3 setup.py build
    ${SYSDIR}/cross-tools/bin/python3 setup.py install --root=${SYSDIR}/sysroot --prefix=/usr
    sed -i "s@${SYSDIR}/cross-tools@@g" ${SYSDIR}/sysroot/bin/meson
popd
```

#### Ninja
　　https://github.com/ninja-build/ninja/archive/v1.12.1/ninja-1.12.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/ninja-1.12.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/ninja-1.12.1
    CXX="${CROSS_TARGET}-g++" AR="${CROSS_TARGET}-ar" \
    ${SYSDIR}/cross-tools/bin/python3 configure.py
    ninja
    install -vm755 ninja ${SYSDIR}/sysroot/usr/bin/
popd
```

#### Perl5
```sh
tar xvf ${DOWNLOADDIR}/perl-5.40.2.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/perl-5.40.2
	sh Configure -des -Dprefix=/usr -Dvendorprefix=/usr \
	             -Dprivlib=/usr/lib/perl5/5.4x/core_perl \
	             -Darchlib=/usr/lib32/perl5/5.4x/core_perl \
	             -Dsitelib=/usr/lib/perl5/5.4x/site_perl \
	             -Dsitearch=/usr/lib32/perl5/5.4x/site_perl \
	             -Dvendorlib=/usr/lib/perl5/5.4x/vendor_perl \
	             -Dvendorarch=/usr/lib32/perl5/5.4x/vendor_perl \
	             -Dman1dir=/usr/share/man/man1 -Dman3dir=/usr/share/man/man3 \
	             -Dpager="/usr/bin/less -isR" -Duseshrplib -Dusethreads \
	             -Dusecrosscompile
	cp ${DOWNLOADDIR}/perl-5.x-loongarch32-config.sh ./config.sh
	sed -i "/^cc=/s@'cc'@'${CROSS_TARGET}-gcc'@g" config.sh
	sed -i "/^ld=/s@'cc'@'${CROSS_TARGET}-gcc'@g" config.sh
	./Configure -S
	make depend
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### XML-Parser
```sh
tar xvf ${DOWNLOADDIR}/XML-Parser-2.47.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/XML-Parser-2.47
    ${SYSDIR}/cross-tools/bin/perl Makefile.PL CC=${CROSS_TARGET}-gcc LD=${CROSS_TARGET}-ld
    sed -i "/^INSTALL/s@${SYSDIR}/cross-tools@/usr@g" Makefile Expat/Makefile
    sed -i "/^PERL_INC/s@${SYSDIR}/cross-tools@${SYSDIR}/sysroot/usr@g" Makefile Expat/Makefile
    sed -i "/^LDDLFLAGS/s@/usr/local/lib@${SYSDIR}/sysroot/usr/lib32@g" Makefile Expat/Makefile
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### URI
```sh
tar xvf ${DOWNLOADDIR}/URI-5.32.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/URI-5.32
    ${SYSDIR}/cross-tools/bin/perl Makefile.PL CC=${CROSS_TARGET}-gcc LD=${CROSS_TARGET}-ld
    sed -i "/^INSTALL/s@${SYSDIR}/cross-tools@/usr@g" Makefile
    sed -i "/^PERL_INC/s@${SYSDIR}/cross-tools@${SYSDIR}/sysroot/usr@g" Makefile
    sed -i "/^LDDLFLAGS/s@/usr/local/lib@${SYSDIR}/sysroot/usr/lib32@g" Makefile
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libgpg-error
　　https://www.gnupg.org/ftp/gcrypt/libgpg-error/libgpg-error-1.55.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/libgpg-error-1.55.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/libgpg-error-1.55
	cp ${SYSDIR}/sysroot/usr/share/automake-*/config.* build-aux/
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libgpg-error*.la
popd
```

#### Libgcrypt
　　https://www.gnupg.org/ftp/gcrypt/libgcrypt/libgcrypt-1.11.1.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/libgcrypt-1.11.1.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/libgcrypt-1.11.1
	CFLAGS="${CFLAGS} -DGPGRT_ENABLE_ES_MACROS" \
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libgcrypt*.la
	cp -av ${SYSDIR}/sysroot/bin/libgcrypt-config ${SYSDIR}/cross-tools/bin/
popd
```


#### Libxml2
　　https://download.gnome.org/sources/libxml2/2.14/libxml2-2.14.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libxml2-2.14.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libxml2-2.14.3
	mkdir cross-build
	pushd cross-build
		../configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
			--host=${CROSS_TARGET} --with-history --with-icu --with-http \
			--with-python=${CROSSTOOLS_DIR}/bin/${CROSS_TARGET}-python3 \
			--with-python_install_dir=/usr/lib32/python3.13/site-packages \
			PYTHON=${CROSS_TARGET}-python3
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
		rm ${SYSDIR}/sysroot/usr/lib32/libxml2.la
	popd
popd
```

#### Libxslt
　　https://download.gnome.org/sources/libxslt/1.1/libxslt-1.1.43.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libxslt-1.1.43.tar.x -C ${BUILDDIR}
pushd ${BUILDDIR}/libxslt-1.1.43
	rm config.{sub,guess}
	automake -a
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET} PYTHON=${CROSS_TARGET}-python3
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### GPM
　　https://github.com/telmich/gpm/archive/1.20.7/gpm-1.20.7.tar.gz

　　GPM软件包提供了在文本环境下使用鼠标的工具。

```sh
tar xvf ${DOWNLOADDIR}/gpm-1.20.7.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/gpm-1.20.7
	patch -Np1 -i ${DOWNLOADDIR}/gpm-1.20.7-consolidated-1.patch
	patch -Np1 -i ${DOWNLOADDIR}/gpm-1.20.1-weak-wgetch.patch
	./autogen.sh
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 --sysconfdir=/etc \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	ln -sfv libgpm.so.2.1.0 ${SYSDIR}/sysroot/usr/lib32/libgpm.so
popd

cat > ${SYSDIR}/sysroot/etc/sysconfig/mouse << "EOF"
MDEVICE="/dev/input/mice"
PROTOCOL="imps2"
GPMOPTS=""
EOF

cat > ${SYSDIR}/sysroot/usr/lib/systemd/system/gpm.service << "EOF"
[Unit]
Description=Console Mouse manager

[Service]
ExecStart=/usr/sbin/gpm -m /dev/input/mice -t exps2
Type=forking
PIDFile=/run/gpm.pid

[Install]
WantedBy=multi-user.target
EOF
```

#### Libevent
　　https://github.com/libevent/libevent/archive/release-2.1.12-stable/libevent-release-2.1.12-stable.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libevent-2.1.12-stable.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libevent-2.1.12-stable
	rm build-aux/config.{guess,sub}
	automake -a
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Links
　　http://links.twibright.com/download/links-2.30.tar.bz2

　　Links是一个文本环境下简易的互联网浏览器。

```sh
tar xvf ${DOWNLOADDIR}/links-2.30.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/links-2.30
	CC="${CROSS_TARGET}-gcc" \
	./configure --prefix=/usr --libdir=/usr/lib32 --mandir=/usr/share/man \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Doxygen
　　https://www.doxygen.nl/files/doxygen-1.13.2.src.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/doxygen-1.13.2.src.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/doxygen-1.13.2
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release \
		      -DCMAKE_INSTALL_PREFIX=/usr -Wno-dev ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### Git
　　https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.49.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/git-2.49.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/git-2.49.0
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--with-gitconfig=/etc/gitconfig --with-python=python3 --without-iconv \
		ac_cv_fread_reads_directories=yes ac_cv_snprintf_returns_bogus=no
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot perllibdir=/usr/lib/perl5/5.3x/site_perl install
popd
```

#### Ctags
　　https://sourceforge.net/projects/ctags/files/ctags//5.8//ctags-5.8.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/ctags-5.8.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/ctags-5.8
	patch -Np1 -i ${DOWNLOADDIR}/ctags-5.8-fix_form_fedora.patch
	patch -Np1 -i ${DOWNLOADDIR}/ctags-5.8-for-gcc_12.patch
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Inih
　　https://github.com/benhoyt/inih/archive/r60/inih-r60.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/inih-r60.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/inih-r60
	mkdir build
	pushd build
		meson --prefix=/usr --libdir=/usr/lib32 \
		      --buildtype=release --cross-file=${BUILDDIR}/meson-cross.txt ..
		ninja
		DESTDIR=${SYSDIR}/sysroot ninja install
	popd
popd
```

#### Dosfstools
　　https://github.com/dosfstools/dosfstools/releases/download/v4.2/dosfstools-4.2.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/dosfstools-4.2.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/dosfstools-4.2
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-compat-symlinks --mandir=/usr/share/man
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Userspace-RCU
　　https://lttng.org/files/urcu/userspace-rcu-0.15.2.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/userspace-rcu-0.15.2.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/userspace-rcu-0.15.2
    ./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} --host=${CROSS_TARGET}
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
    rm -v ${SYSDIR}/sysroot/usr/lib32/liburcu*.la
popd
```

#### Xfsprogs
　　https://mirrors.edge.kernel.org/pub/linux/utils/fs/xfs/xfsprogs/xfsprogs-6.14.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/xfsprogs-6.14.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/xfsprogs-6.14.0
    patch -Np1 -i ${DOWNLOADDIR}/0001-Fix-for-cross-build.patch
    patch -Np1 -i ${DOWNLOADDIR}/0002-Fix-for-gcc-13.patch
    CC=${CROSS_TARGET}-gcc ./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --with-systemd-unit-dir=/usr/lib/systemd/system --with-udev-rule-dir=/usr/lib/udev/rules.d
    make -j
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libaio
　　https://ftp.debian.org/debian/pool/main/liba/libaio/libaio_0.3.113.orig.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libaio_0.3.113.orig.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libaio-0.3.113
	make CC="${CROSS_TARGET}-gcc" ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot libdir=/usr/lib32 install
popd
```

#### Mdadm
　　https://mirrors.edge.kernel.org/pub/linux/utils/raid/mdadm/mdadm-4.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/mdadm-4.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/mdadm-4.3
    make CC="${CROSS_TARGET}-gcc" CXFLAGS="${CFLAGS} -Wno-error=unterminated-string-initialization -Wno-error=format" ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot PKG_CONFIG=${CROSS_TARGET}-pkg-config install
popd
```
　　Mdadm软件包使用CXFLAGS来传递CFLAGS参数，以保证其内部参数传递正确。

#### LVM2
　　https://sourceware.org/ftp/lvm2/LVM2.2.03.32.tgz

```sh
tar xvf ${DOWNLOADDIR}/LVM2.2.03.32.tgz -C ${BUILDDIR}
pushd ${BUILDDIR}/LVM2.2.03.32
    ./configure --prefix=/usr --libdir=/usr/lib32 --with-usrlibdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --enable-cmdlib --enable-pkgconfig --enable-udev_sync \
                ac_cv_func_malloc_0_nonnull=yes ac_cv_func_realloc_0_nonnull=yes
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### PCRE
　　https://sourceforge.net/projects/pcre/files/pcre//8.45//pcre-8.45.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/pcre-8.45.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/pcre-8.45
	rm config.{sub,guess}
	automake -a
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET} --enable-unicode-properties \
		--enable-pcre16 --enable-pcre32 \
		--enable-pcregrep-libz --enable-pcregrep-libbz2 \
		--enable-pcretest-libreadline
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### PCRE2
　　https://github.com/PCRE2Project/pcre2/releases/download/pcre2-10.45/pcre2-10.45.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/pcre2-10.45.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/pcre2-10.45
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET} --enable-unicode --disable-jit \
		--enable-pcre2-16 --enable-pcre2-32 --enable-pcre2grep-libz \
		--enable-pcre2grep-libbz2 --enable-pcre2test-libreadline
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Packaging
　　https://files.pythonhosted.org/packages/source/p/packaging/packaging-25.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/packaging-25.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/packaging-25.0
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
	CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
	${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user packaging --root=${SYSDIR}/sysroot --prefix=/usr
popd
```

#### Glib

```sh
tar xvf ${DOWNLOADDIR}/glib-2.84.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/glib-2.84.1
    mkdir build
    pushd build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dtests=false --default-library=both \
              -Dman=true -Dselinux=disabled -Dintrospection=disabled -Dman-pages=false \
              --cross-file=${BUILDDIR}/meson-cross.txt \
              ..
        ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### VIM
　　https://github.com/vim/vim/archive/v9.1.1382/vim-9.1.1382.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/vim-9.1.1382.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/vim-9.1.1382
	echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
cat > src/auto/config.cache << EOF
	vim_cv_getcwd_broken=no
	vim_cv_toupper_broken=no
	vim_cv_terminfo=yes
	vim_cv_tgetent=zero
	vim_cv_stat_ignores_slash=no
	vim_cv_memmove_handles_overlap=yes
	ac_cv_small_wchar_t=no
EOF
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}  --with-tlib=ncurses --enable-gui=no
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot STRIP=${CROSS_TARGET}-strip install
	ln -sv vim ${SYSDIR}/sysroot/usr/bin/vi
popd
```

　　VIM制作过程中也需要设置一些参数避免自动探测错误，但VIM的参数设置文件是有默认文件路径的即“src/auto/config.cache”，在文件中写入参数和取值即可，configure配置脚本会自动从该文件中读取。

　　在安装完VIM后，我们可以配置VIM的默认设置文件，设置步骤如下：

```sh
cat > ${SYSDIR}/sysroot/etc/vimrc << "EOF"
let skip_defaults_vim=1
set nocompatible
set backspace=2
set mouse=

if (&term == "xterm") || (&term == "putty")
  set background=dark
endif
EOF
```
　　改设置内容主要是设置了一些基本的界面和操作特性，如Tab转换成几个空格显示，不同的终端下背景颜色等等。

#### UnRAR
　　https://www.rarlab.com/rar/unrarsrc-7.1.6.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/unrarsrc-7.1.6.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/unrar
	sed -i "s@-march=native -O2@${CXXFLAGS}@g" makefile
	make CXX="${CROSS_TARGET}-g++" STRIP=${CROSS_TARGET}-strip -f makefile ${JOBS}
	install -v -m755 unrar ${SYSDIR}/sysroot/usr/bin
popd
```

#### Zip
　　https://sourceforge.net/projects/infozip/files/Zip%203.x%20%28latest%29/3.0/zip30.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/zip30.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/zip30
	make -f unix/Makefile CC="${CROSS_TARGET}-gcc -DLARGE_FILE_SUPPORT" generic ${JOBS}
	make prefix=${SYSDIR}/sysroot/usr MANDIR=${SYSDIR}/sysroot/usr/share/man/man1 \
	     -f unix/Makefile install
popd
```

#### UnZip
　　https://sourceforge.net/projects/infozip/files/UnZip%206.x%20%28latest%29/UnZip%206.0/unzip60.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/unzip60.tgz -C ${BUILDDIR}
pushd ${BUILDDIR}/unzip60
	sed -i "s@-DNO_LCHMOD@@g" unix/configure
	make -f unix/Makefile CC="${CROSS_TARGET}-gcc \
	     -DLARGE_FILE_SUPPORT -DUNICODE_WCHAR -DUNICODE_SUPPORT -std=gnu17" generic ${JOBS}
	make prefix=${SYSDIR}/sysroot/usr MANDIR=${SYSDIR}/sysroot/usr/share/man/man1 \
	      -f unix/Makefile install
popd
```

#### CPIO
　　https://ftp.gnu.org/gnu/cpio/cpio-2.15.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/cpio-2.15.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/cpio-2.15
	sed -i '/The name/,+2 d' src/global.c
	CFLAGS="${CFLAGS} -Wno-implicit-function-declaration -std=gnu17" \
	./configure --prefix=/usr --build=${CROSS_HOST} \
		--host=${CROSS_TARGET} --enable-mt \
		--with-rmt=/usr/libexec/rmt
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libmnl
　　https://netfilter.org/projects/libmnl/files/libmnl-1.0.5.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/libmnl-1.0.5.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/libmnl-1.0.5
	./configure --prefix=/usr --libdir=/usr/lib32 --build=${CROSS_HOST} \
		--host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Ethtool
　　https://mirrors.edge.kernel.org/pub/software/network/ethtool/ethtool-6.14.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/ethtool-6.14.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/ethtool-6.14
	./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Boost
```sh
tar xvf ${DOWNLOADDIR}/boost_1_88_0.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/boost_1_88_0
    patch -Np1 -i ${DOWNLOADDIR}/0001-boost-add-loongarch32-support.patch
    ./bootstrap.sh ICU_ROOT=${SYSDIR}/sysroot/usr --prefix=/usr --libdir=/usr/lib32 --with-python=python3
    sed -i "/using gcc/s@using gcc@& : loongarch32 : ${CROSS_TARGET}-gcc@g" project-config.jam
    sed -i "/using python/s@${CROSSTOOLS_DIR}@${CROSSTOOLS_DIR}/bin/${CROSS_TARGET}-python3@g" project-config.jam
    sed -i "s@mips @mips1 @g" libs/log/build/log-arch-config.jam
    ./b2 stage threading=multi link=shared address-model=32 toolset=gcc-loongarch32 linkflags="-lstdc++"
    rm -rf ${SYSDIR}/sysroot/usr/lib/cmake/[Bb]oost*
    ./b2 install --prefix=${SYSDIR}/sysroot/usr --libdir=${SYSDIR}/sysroot/usr/lib32 \
             threading=multi link=shared address-model=32 toolset=gcc-loongarch32 linkflags="-lstdc++"
popd
```

#### Libpng
　　https://sourceforge.net/projects/libpng/files/libpng16/1.6.48/libpng-1.6.48.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libpng-1.6.48.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libpng-1.6.48
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libpng16.la
popd
```

#### LibJPEG-Turbo
　　https://sourceforge.net/projects/libjpeg-turbo/files/3.0.1/libjpeg-turbo-3.0.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libjpeg-turbo-3.0.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libjpeg-turbo-3.0.1
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=${COMMON_DIR}/cmake-cross.cmake \
		      -DCMAKE_INSTALL_PREFIX=/usr \
		      -DWITH_JPEG8=ON \
		      -DCMAKE_INSTALL_DEFAULT_LIBDIR=lib32 ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### TIFF
　　https://download.osgeo.org/libtiff/tiff-4.7.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/tiff-4.7.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/tiff-4.7.0
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=${COMMON_DIR}/cmake-cross.cmake \
		      -DCMAKE_INSTALL_PREFIX=/usr \
		      -DCMAKE_INSTALL_LIBDIR=lib32 -Dtiff-docs=OFF ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
		sed -i /Version/s/\$/$(cat ../VERSION)/ \
		       ${SYSDIR}/sysroot/usr/lib32/pkgconfig/libtiff-4.pc
	popd
popd
```

#### LCMS2
　　https://sourceforge.net/projects/lcms/files/lcms/2.17/lcms2-2.17.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/lcms2-2.17.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/lcms2-2.17
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### OpenJPEG
　　https://github.com/uclouvain/openjpeg/archive/v2.5.3/openjpeg-2.5.3.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/openjpeg-2.5.3.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/openjpeg-2.5.3
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=${COMMON_DIR}/cmake-cross.cmake \
		      -DCMAKE_INSTALL_PREFIX=/usr \
		      -DOPENJPEG_INSTALL_LIB_DIR=lib32 ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### Jasper
　　https://github.com/jasper-software/jasper/archive/version-4.2.5/jasper-4.2.5.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/jasper-4.2.5.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/jasper-4.2.5
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=${COMMON_DIR}/cmake-cross.cmake \
		      -DCMAKE_INSTALL_PREFIX=/usr \
		      -DCMAKE_SKIP_INSTALL_RPATH=YES \
		      -DJAS_CROSSCOMPILING=True -DJAS_STDC_VERSION=201710L \
		      -DJAS_ENABLE_DOC=NO ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### LibRaw
　　https://www.libraw.org/data/LibRaw-0.21.4.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/LibRaw-0.21.4.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/LibRaw-0.21.4
	autoreconf -ifv
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-jpeg --enable-jasper --enable-lcms
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libmng
　　https://sourceforge.net/projects/libmng/files/libmng-devel/2.0.3/libmng-2.0.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libmng-2.0.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libmng-2.0.3
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### FreeType
　　https://sourceforge.net/projects/freetype/files/freetype2/2.13.3/freetype-2.13.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/freetype-2.13.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/freetype-2.13.3
	sed -ri "s:.*(AUX_MODULES.*valid):\1:" modules.cfg
	sed -r "s:.*(#.*SUBPIXEL_RENDERING) .*:\1:" \
	    -i include/freetype/config/ftoption.h
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-freetype-config
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
sed -i -e "s@${SYSDIR}/cross-tools/bin@/usr/bin@g" \
       -e "s@${CROSS_TARGET}-@@g" \
       ${SYSDIR}/sysroot/usr/bin/freetype-config
```

#### Gobject-Introspection
```sh
tar xvf ${DOWNLOADDIR}/gobject-introspection-1.84.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gobject-introspection-1.84.0
    sed -i -e "/gircompiler_command/s@gircompiler,@'${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-g-ir-compiler',@g" \
           -e "/g-ir-scanner/s@'g-ir-scanner'@'${CROSS_TARGET}-g-ir-scanner'@g" \
           gir/meson.build
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dgi_cross_use_prebuilt_gi=true \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### Glib （再次编译）
　　再次编译Glib的目的是为了增加对Gobject-Introspection的支持。

```sh
tar xvf ${DOWNLOADDIR}/glib-2.84.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/glib-2.84.1
    mkdir build
    pushd build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dtests=false --default-library=both \
              -Dman=true -Dselinux=disabled -Dman-pages=false \
              --cross-file=${BUILDDIR}/meson-cross.txt \
              ..
        ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
echo '#!/bin/bash -e
qemu-loongarch32 ${SYSDIR}/sysroot/usr/bin/glib-compile-resources "$@"' > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-glib-compile-resources
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-glib-compile-resources
cp -a ${SYSDIR}/cross-tools/bin/{${CROSS_TARGET}-,}glib-compile-resources
echo '#!/bin/bash -e
qemu-loongarch32 ${SYSDIR}/sysroot/usr/bin/glib-compile-schemas "$@"' > ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-glib-compile-schemas
chmod +x ${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-glib-compile-schemas
cp -a ${SYSDIR}/cross-tools/bin/{${CROSS_TARGET}-,}glib-compile-schemas
cp -a ${SYSDIR}/sysroot/usr/bin/glib-mkenums ${SYSDIR}/cross-tools/bin/
```

#### HarfBuzz
　　https://github.com/harfbuzz/harfbuzz/releases/download/11.2.1/harfbuzz-11.2.1.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/harfbuzz-11.2.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/harfbuzz-11.2.1
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dbenchmark=disabled -Dintrospection=enabled -Dgraphite2=disabled \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        sed -i -e "s@\(${SYSDIR}/\)sysroot/usr\(.*\)\(g-ir-compiler\|g-ir-scanner\)@\1cross-tools\2${CROSS_TARGET}-\3@g" build.ninja
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### Graphite
　　https://github.com/silnrsi/graphite/archive/1.3.14/graphite-1.3.14.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/graphite-1.3.14.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/graphite-1.3.14
	patch -Np1 -i ${DOWNLOADDIR}/0001-graphite-1.3.14-fix-for-gcc-15.patch
	sed -i "/mfpmath/d" src/CMakeLists.txt
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -DCMAKE_INSTALL_PREFIX=/usr -DLIB_SUFFIX=32 \
		      -DCMAKE_BUILD_TYPE=Release ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### HarfBuzz(第二次)
　　这次编译是加入对Graphite的支持。

```sh
tar xvf ${DOWNLOADDIR}/harfbuzz-11.2.1.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/harfbuzz-11.2.1
    mkdir cross-build-2
    pushd cross-build-2
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dbenchmark=disabled -Dintrospection=enabled -Dgraphite2=enabled \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        sed -i -e "s@\(${SYSDIR}/\)sysroot/usr\(.*\)\(g-ir-compiler\|g-ir-scanner\)@\1cross-tools\2${CROSS_TARGET}-\3@g" build.ninja
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### Brotli
　　https://github.com/google/brotli/archive/v1.1.0/brotli-1.1.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/brotli-1.1.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/brotli-1.1.0
	mkdir build
	pushd build
		CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
		cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=${COMMON_DIR}/cmake-cross.cmake \
		      -DCMAKE_INSTALL_PREFIX=/usr -DLIB_SUFFIX=32 \
		      -DCMAKE_BUILD_TYPE=Release ..
		make ${JOBS}
		make DESTDIR=${SYSDIR}/sysroot install
	popd
popd
```

#### FreeType(第二次)
　　这次编译是加入对HarfBuzz的支持。

```sh
tar xvf ${DOWNLOADDIR}/freetype-2.13.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/freetype-2.13.3
	sed -ri "s:.*(AUX_MODULES.*valid):\1:" modules.cfg
	sed -r "s:.*(#.*SUBPIXEL_RENDERING) .*:\1:" \
	    -i include/freetype/config/ftoption.h
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-freetype-config
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libfreetype.la
popd
```

#### Fontconfig
　　https://www.freedesktop.org/software/fontconfig/release/fontconfig-2.16.0.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/fontconfig-2.16.0.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/fontconfig-2.16.0
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--sysconfdir=/etc --localstatedir=/var --disable-docs
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libfontconfig.la
popd
```

#### Fribidi
　　https://github.com/fribidi/fribidi/archive/v1.0.16/fribidi-1.0.16.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/fribidi-1.0.16.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/fribidi-1.0.16
	mkdir build
	pushd build
		meson --prefix=/usr --libdir=/usr/lib32 \
		      --buildtype=release \
		      --cross-file=${BUILDDIR}/meson-cross.txt ..
		ninja
		DESTDIR=${SYSDIR}/sysroot ninja install
	popd
popd
```

#### Nettle
　　https://ftp.gnu.org/gnu/nettle/nettle-3.10.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/nettle-3.10.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/nettle-3.10.1
	cp ${SYSDIR}/sysroot/usr/share/automake-*/config.* ./
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### GC
　　https://www.hboehm.info/gc/gc_source/gc-8.2.8.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/gc-8.2.8.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/gc-8.2.8
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET} \
		--enable-cplusplus --with-libatomic-ops=none
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libgc*.la
popd
```

#### Guile
　　https://ftp.gnu.org/gnu/guile/guile-3.0.10.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/guile-3.0.10.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/guile-3.0.10
	CFLAGS="${CFLAGS} -std=gnu17" \
	./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --with-libgmp-prefix=${SYSDIR}/sysroot/usr/lib32 \
                --with-libunistring-prefix=${SYSDIR}/sysroot/usr/lib32
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libguile*.la
popd
```

#### Libtasn1
　　https://ftp.gnu.org/gnu/libtasn1/libtasn1-4.20.0.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libtasn1-4.20.0.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libtasn1-4.20.0
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	rm -v ${SYSDIR}/sysroot/usr/lib32/libtasn1.la
popd
```

#### P11-Kit
　　https://github.com/p11-glue/p11-kit/releases/download/0.25.5/p11-kit-0.25.5.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/p11-kit-0.25.5.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/p11-kit-0.25.5
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release \
              -Dtrust_paths=/etc/pki/anchors \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### GnuTLS
　　https://www.gnupg.org/ftp/gcrypt/gnutls/v3.8/gnutls-3.8.9.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/gnutls-3.8.9.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/gnutls-3.8.9
    ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --enable-openssl-compatibility --enable-ssl3-support \
                --with-default-trust-store-pkcs11="pkcs11:" \
                --with-libz-prefix=${SYSDIR}/sysroot/usr \
                --disable-guile --disable-doc --enable-static
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
rm -v ${SYSDIR}/sysroot/usr/lib32/libgnutls*.la
```

#### Vala
```sh
tar xvf ${DOWNLOADDIR}/vala-0.56.18.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/vala-0.56.18
    CFLAGS="${CFLAGS} -Wno-incompatible-pointer-types" \
    ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --disable-valadoc
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### LibUSB
　　https://github.com/libusb/libusb/archive/v1.0.28/libusb-1.0.28.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libusb-1.0.28.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libusb-1.0.28
    ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET}
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### USBUtils
　　https://mirrors.edge.kernel.org/pub/linux/utils/usb/usbutils/usbutils-018.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/usbutils-018.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/usbutils-018
   ./configure --prefix=/usr --build=${CROSS_HOST} --host=${CROSS_TARGET} \
                --datadir=/usr/share/hwdata
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
    mkdir -v ${SYSDIR}/sysroot/usr/share/hwdata/
    wget http://www.linux-usb.org/usb.ids -O ${SYSDIR}/sysroot/usr/share/hwdata/usb.ids
popd
```

#### Json-Glib
　　https://download.gnome.org/sources/json-glib/1.10/json-glib-1.10.6.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/json-glib-1.10.6.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/json-glib-1.10.6
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Dgtk_doc=disabled \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        sed -i -e "s@\(${SYSDIR}/\)sysroot/usr\(.*\)\(g-ir-compiler\|g-ir-scanner\)@\1cross-tools\2${CROSS_TARGET}-\3@g" build.ninja
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### LibGUSB
　　https://github.com/hughsie/libgusb/archive/0.4.9/libgusb-0.4.9.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libgusb-0.4.9.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libgusb-0.4.9
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 \
              --buildtype=release -Ddocs=false \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        sed -i -e "s@\(${SYSDIR}/\)sysroot/usr\(.*\)\(g-ir-compiler\|g-ir-scanner\)@\1cross-tools\2${CROSS_TARGET}-\3@g" build.ninja
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### LibGUdev
　　https://download.gnome.org/sources/libgudev/238/libgudev-238.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libgudev-238.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libgudev-238
    mkdir cross-build
    pushd cross-build
        meson --prefix=/usr --libdir=/usr/lib32 --buildtype=release  \
              --cross-file=${BUILDDIR}/meson-cross.txt ..
        sed -i -e "s@\(${SYSDIR}/\)sysroot/usr\(.*\)\(g-ir-compiler\|g-ir-scanner\)@\1cross-tools\2${CROSS_TARGET}-\3@g" build.ninja
        CC="${CROSS_TARGET}-gcc" ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
```

#### DHCP
　　https://ftp.isc.org/isc/dhcp/4.4.3-P1/dhcp-4.4.3-P1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/dhcp-4.4.3-P1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/dhcp-4.4.3-P1
        CFLAGS="${CFLAGS:--g -O2} -Wall -fno-strict-aliasing -Wno-incompatible-pointer-types -std=gnu17 \
        -D_PATH_DHCLIENT_SCRIPT='\"/usr/sbin/dhclient-script\"' \
        -D_PATH_DHCPD_CONF='\"/etc/dhcp/dhcpd.conf\"' \
        -D_PATH_DHCLIENT_CONF='\"/etc/dhcp/dhclient.conf\"'" \
        ./configure --prefix=/usr --host=${CROSS_TARGET} \
            --sysconfdir=/etc/dhcp --localstatedir=/var \
            --with-randomdev=no \
            --with-srv-lease-file=/var/lib/dhcpd/dhcpd.leases       \
            --with-srv6-lease-file=/var/lib/dhcpd/dhcpd6.leases     \
            --with-cli-lease-file=/var/lib/dhclient/dhclient.leases \
            --with-cli6-lease-file=/var/lib/dhclient/dhclient6.leases
        make -j${JOBS} BUILD_CC=gcc LIBS="-latomic"
        make -C client DESTDIR=${SYSDIR}/sysroot install
        install -v -m755 client/scripts/linux ${SYSDIR}/sysroot/usr/sbin/dhclient-script

install -vdm755 ${SYSDIR}/sysroot/etc/dhcp &&
cat > ${SYSDIR}/sysroot/etc/dhcp/dhclient.conf << "EOF"
 # Begin /etc/dhcp/dhclient.conf
 #
 # Basic dhclient.conf(5)

 # prepend domain-name-servers 127.0.0.1;
request subnet-mask, broadcast-address, time-offset, routers,
        domain-name, domain-name-servers, domain-search, host-name,
        netbios-name-servers, netbios-scope, interface-mtu,
        ntp-servers;
require subnet-mask, domain-name-servers;
 # timeout 60;
 # retry 60;
 # reboot 10;
 # select-timeout 5;
 # initial-interval 2;

EOF

install -v -dm 755 ${SYSDIR}/sysroot/var/lib/dhclient
popd
```

#### BusyBox
　　https://busybox.net/downloads/busybox-1.37.0.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/busybox-1.37.0.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/busybox-1.37.0
	make mrproper
	make defconfig
	sed -i "/^# CONFIG_STATIC=/d" .config
	echo "CONFIG_STATIC=y" >> .config
	sed -i "/CONFIG_TC=y/s@=y@=n@g" .config
	make -j${JOBS} CROSS_COMPILE=${CROSS_TARGET}-
	cp -v busybox ${SYSDIR}/sysroot/usr/bin/
popd
```

#### Libksba
　　https://www.gnupg.org/ftp/gcrypt/libksba/libksba-1.6.7.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/libksba-1.6.7.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/libksba-1.6.7
         ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET}
        CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
        make DESTDIR=${SYSDIR}/sysroot install
        rm -f ${SYSDIR}/sysroot/usr/lib32/*.la
popd
```

#### Npth
　　https://gnupg.org/ftp/gcrypt/npth/npth-1.8.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/npth-1.8.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/npth-1.8
        ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} \
	        --enable-install-npth-config --enable-malloc0returnsnull
        CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
        make DESTDIR=${SYSDIR}/sysroot install
        rm -f ${SYSDIR}/sysroot/usr/lib32/*.la
        cp -a ${SYSDIR}/sysroot/usr/bin/npth-config ${SYSDIR}/cross-tools/bin/
        sed -i "/^prefix/s@=\/usr@=${SYSDIR}\/sysroot\/usr@g" ${SYSDIR}/cross-tools/bin/npth-config
        sed -i "/^libs/s@-L\/usr@-L${SYSDIR}\/sysroot\/usr@g" ${SYSDIR}/cross-tools/bin/npth-config
popd

```

#### Libassuan
　　https://www.gnupg.org/ftp/gcrypt/libassuan/libassuan-3.0.2.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/libassuan-3.0.2.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/libassuan-3.0.2
	./configure --prefix=/usr --libdir=/usr/lib32 \
                    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	make ${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### HWData
　　https://github.com/vcrhonek/hwdata/archive/v0.395/hwdata-0.395.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/hwdata-0.395.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/hwdata-0.395
	./configure --prefix=/usr --libdir=/usr/lib32 \
                    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd

```

#### Dejavu-Fonts
　　https://sourceforge.net/projects/dejavu/files/dejavu//2.37//dejavu-fonts-ttf-2.37.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/dejavu-fonts-ttf-2.37.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/dejavu-fonts-ttf-2.37
	cp fontconfig/*.conf ${SYSDIR}/sysroot/usr/share/fontconfig/conf.avail/
	install -dv ${SYSDIR}/sysroot/usr/share/fonts/DejaVu/
	cp -v ttf/* ${SYSDIR}/sysroot/usr/share/fonts/DejaVu/
popd
```

#### LZO
　　http://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/lzo-2.10.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/lzo-2.10
	./configure --prefix=/usr --libdir=/usr/lib32 \
		    --enable-shared \
                    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libnftnl
　　https://www.netfilter.org/pub/libnftnl/libnftnl-1.2.9.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/libnftnl-1.2.9.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/libnftnl-1.2.9
	./configure --prefix=/usr --libdir=/usr/lib32 \
                    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### Libedit
　　https://www.thrysoee.dk/editline/libedit-20250104-3.1.tar.gz

```sh
tar xvf ${DOWNLOADDIR}/libedit-20250104-3.1.tar.gz -C ${BUILDDIR}
pushd ${BUILDDIR}/libedit-20250104-3.1
	./configure --prefix=/usr --libdir=/usr/lib32 \
                    --build=${CROSS_HOST} --host=${CROSS_TARGET}
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
popd

```
#### Nftables
　　https://www.netfilter.org/pub/nftables/nftables-1.1.3.tar.xz

```sh
tar xvf ${DOWNLOADDIR}/nftables-1.1.3.tar.xz -C ${BUILDDIR}
pushd ${BUILDDIR}/nftables-1.1.3
	./configure --prefix=/usr --libdir=/usr/lib32 \
		--build=${CROSS_HOST} --host=${CROSS_TARGET}
	CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" make -j${JOBS}
	make DESTDIR=${SYSDIR}/sysroot install
	pushd py
		CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
		${SYSDIR}/cross-tools/bin/pip3 wheel -w dist --no-build-isolation --no-deps ${PWD}
		CC=${CROSS_TARGET}-gcc CXX=${CROSS_TARGET}-g++ _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__linux_${CROSS_TARGET} \
		${SYSDIR}/cross-tools/bin/pip3 install --no-index --find-links dist --no-cache-dir --no-deps --ignore-installed --no-user nftables --root=${SYSDIR}/sysroot --prefix=/usr
	popd
popd
```

#### Firewalld
　　https://github.com/firewalld/firewalld/releases/download/v2.3.0/firewalld-2.3.0.tar.bz2

```sh
tar xvf ${DOWNLOADDIR}/firewalld-2.3.0.tar.bz2 -C ${BUILDDIR}
pushd ${BUILDDIR}/firewalld-2.3.0
    ./configure --prefix=/usr --libdir=/usr/lib32 \
                --build=${CROSS_HOST} --host=${CROSS_TARGET} --disable-docs \
		PYTHON=${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-python3
    make ${JOBS}
    make DESTDIR=${SYSDIR}/sysroot install
popd
```

#### LLVM

```sh
pushd ${BUILDDIR}
git clone https://github.com/heiher/llvm-project.git --depth 1 -b la32
pushd llvm-project
    mkdir cross-build
    pushd cross-build
        CC="${CROSS_TARGET}-gcc" CXX="${CROSS_TARGET}-g++" \
        cmake .. -G Ninja -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release \
              -DBUILD_SHARED_LIBS:BOOL=OFF -DLLVM_ENABLE_LIBCXX:BOOL=OFF \
              -DCMAKE_C_FLAGS="-DNDEBUG" -DCMAKE_CXX_FLAGS="-DNDEBUG" \
              -DLLVM_BUILD_RUNTIME:BOOL=ON -DLLVM_ENABLE_RTTI:BOOL=ON \
              -DLLVM_ENABLE_ZLIB:BOOL=ON -DLLVM_ENABLE_FFI:BOOL=ON \
              -DLLVM_ENABLE_TERMINFO:BOOL=OFF \
              -DLLVM_TABLEGEN:PATH=${SYSDIR}/cross-tools/bin/llvm-tblgen \
              -DLLVM_BUILD_LLVM_DYLIB:BOOL=ON \
              -DLLVM_LINK_LLVM_DYLIB:BOOL=ON -DLLVM_BUILD_EXTERNAL_COMPILER_RT:BOOL=ON \
              -DLLVM_INSTALL_TOOLCHAIN_ONLY:BOOL=OFF \
	      -DLLVM_VERSION_SUFFIX='' \
	      -DLLVM_ENABLE_PROJECTS='llvm;clang;lld' \
	      -DCLANG_DEFAULT_LINKER=lld \
	      -DLLVM_HOST_TRIPLE=${CROSS_TARGET} \
              -DLLVM_DEFAULT_TARGET_TRIPLE=${CROSS_TARGET}
        ninja
        DESTDIR=${SYSDIR}/sysroot ninja install
    popd
popd
rm -rf llvm-project
popd
```

#### LLVM Runtimes

```sh
pushd ${BUILDDIR}
git clone https://github.com/heiher/llvm-project.git --depth 1 -b la32
pushd llvm-project
    pushd compiler-rt
        mkdir cross-build
        pushd cross-build
            CC="clang" CXX="clang++" \
            cmake .. -G Ninja -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release \
                  -DLLVM_CONFIG=${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-llvm-config \
                  -DLLVM_LIBDIR_SUFFIX=32 \
                  -DCMAKE_AR="${SYSDIR}/cross-tools/bin/llvm-ar" \
                  -DCMAKE_RANLIB="${SYSDIR}/cross-tools/bin/llvm-ranlib" \
                  -DCMAKE_C_FLAGS="-DNDEBUG -target ${CROSS_TARGET}" \
                  -DCMAKE_CXX_FLAGS="-DNDEBUG -target ${CROSS_TARGET}" \
                  -DCMAKE_ASM_FLAGS="-target ${CROSS_TARGET}" \
                  -DCMAKE_MAKE_PROGRAM:PATH=${SYSDIR}/cross-tools/bin/ninja \
                  -DPython3_EXECUTABLE=${SYSDIR}/cross-tools/bin/${CROSS_TARGET}-python3 \
                  -DCMAKE_SYSROOT=${SYSDIR}/sysroot \
                  -DCMAKE_FIND_ROOT_PATH=${SYSDIR}/sysroot/usr \
                  -DLLVM_USE_SANITIZER:BOOL=ON \
                  -DLLVM_LINK_LLVM_DYLIB:BOOL=ON \
                  -DLLVM_HOST_TRIPLE=${CROSS_TARGET}
            ninja
            DESTDIR=${SYSDIR}/sysroot ninja install
        popd
    popd
popd
rm -rf llvm-project
popd
```

## 6 设置目标系统
　　目标系统的软件包制作过程已经完成，接下来就是对目标系统进行必要的设置，以便目标系统可以正常的启动和使用。

### 创建用户文件

　　创建基本的用户名，这些用户名大多数在启动过程中会用到，步骤如下：

```sh
cat > ${SYSDIR}/sysroot/etc/passwd << "EOF"
root::0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/var/run/dbus:/bin/false
lp:x:9:9:Print Service User:/var/spool/cups:/bin/false
polkitd:x:27:27:PolicyKit Daemon Owner:/etc/polkit-1:/bin/false
rsyncd:x:46:46:rsyncd Daemon:/home/rsync:/bin/false
sshd:x:50:50:sshd PrivSep:/var/lib/sshd:/bin/false
lightdm:x:65:65:Lightdm Daemon:/var/lib/lightdm:/bin/false
sddm:x:66:66:Simple Desktop Display Manager:/var/lib/sddm:/sbin/nologin
colord:x:71:71:Color Daemon Owner:/var/lib/colord:/bin/false
systemd-bus-proxy:x:72:72:systemd Bus Proxy:/:/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/bin/false
systemd-network:x:76:76:systemd Network Management:/:/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/bin/false
systemd-coredump:x:79:79:systemd Core Dumper:/:/bin/false
systemd-oom:x:80:80:systemd Userspace OOM Killer:/:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF
```


### 创建组文件

　　创建基本用户组，大多数都是系统必须的，步骤如下：

```sh
cat > ${SYSDIR}/sysroot/etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
lpadmin:x:19:
systemd-journal:x:23:
input:x:24:
polkitd:x:27:
mail:x:34:
rsyncd:x:46:
sshd:x:50:
kvm:x:61:
lightdm:x:65:
sddm:x:66:
colord:x:71:
systemd-bus-proxy:x:72:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
systemd-coredump:x:79:
systemd-oom:x:80:
saslauth:x:81:
wheel:x:97:
nogroup:x:99:
users:x:1000:
EOF
```

### 创建用户环境设置文件

```sh
echo '# /etc/bashrc

pathmunge () {
    case ":${PATH}:" in
        *:"$1":*)
            ;;
        *)
            if [ "$2" = "after" ] ; then
                PATH=$PATH:$1
            else
                PATH=$1:$PATH
            fi
    esac
}

pathmunge /usr/sbin

for i in /etc/profile.d/*.sh /etc/profile.d/sh.local ; do
    if [ -r "$i" ]; then
        if [ "${-#*i}" != "$-" ]; then
            . "$i"
        else
            . "$i" >/dev/null
        fi
    fi
done

unset i
unset -f pathmunge
' > ${SYSDIR}/sysroot/etc/bashrc
```

```sh
echo '# /etc/profile
. /etc/bashrc
' > ${SYSDIR}/sysroot/etc/profile
```

```sh
echo "PS1='[\u@\h \W]\\\$ '
export PS1
export LC_ALL=zh_CN.UTF-8
" > ${SYSDIR}/sysroot/etc/profile.d/default-event.sh
```

```sh
echo '
if [[ $(find /opt -maxdepth 1 -type d) ]]; then
    for i in $(find /opt -maxdepth 1 -type d | sort); do
        if [ -d $i/bin ]; then
            pathmunge $i/bin after
        fi
    done
fi
' > ${SYSDIR}/sysroot/etc/profile.d/add-opt-path.sh
```


```sh
mkdir -pv ${SYSDIR}/sysroot/etc/skel
echo '
if [ -f /etc/bashrc ]; then
    . /etc/bashrc
fi
' > ${SYSDIR}/sysroot/etc/skel/.bash_profile
cp -v ${SYSDIR}/sysroot/etc/skel/.bash{_profile,rc}
```

### 创建动态库搜索配置

```sh
cat > ${SYSDIR}/sysroot/etc/ld.so.conf << "EOF"
include /etc/ld.so.conf.d/*.conf

EOF
mkdir -pv ${SYSDIR}/sysroot/etc/ld.so.conf.d
```


### 创建nsswitch文件

```sh
cat > ${SYSDIR}/sysroot/etc/nsswitch.conf << "EOF"
passwd: files
group: files
shadow: files

hosts: files dns
networks: files

protocols: files
services: files
ethers: files
rpc: files

EOF
```

### 网络设置

```sh
cat > ${SYSDIR}/sysroot/etc/systemd/network/10-eth-dhcp.network << "EOF"
[Network]
DHCP=ipv4

[DHCP]
UseDomains=true
EOF
```

### 创建域名服务文件

```sh
ln -sfv /run/systemd/resolve/resolv.conf ${SYSDIR}/sysroot/etc/resolv.conf
```

### 创建主机名

```sh
echo "Sunhaiyong" > ${SYSDIR}/sysroot/etc/hostname
```
这里根据需要可以更换其他名字。


### 创建主机列表文件

```sh
cat > ${SYSDIR}/sysroot/etc/hosts << "EOF"
127.0.0.1 localhost.localdomain localhost
::1       localhost ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters
EOF

```

### 设置默认语言环境

```sh
cat > ${SYSDIR}/sysroot/etc/locale.conf << "EOF"
LANG=zh_CN.UTF-8
LC_ALL=zh_CN.UTF-8
LC_CTYPE=zh_CN.UTF-8
LC_NUMERIC=zh_CN.UTF-8
EOF
```

### 创建输入配置文件

```sh
cat > ${SYSDIR}/sysroot/etc/inputrc << "EOF"
set horizontal-scroll-mode Off
set meta-flag On
set input-meta On
set convert-meta Off
set output-meta On
set bell-style none
"\eOd": backward-word
"\eOc": forward-word
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert
"\eOH": beginning-of-line
"\eOF": end-of-line
"\e[H": beginning-of-line
"\e[F": end-of-line
EOF
```
　　通过创建输入配置文件，可以使终端输入时更加符合常见系统中的习惯，不创建该文件也不会对系统造成影响。

### 设置时间文件
```sh
cat > ${SYSDIR}/sysroot/etc/adjtime << "EOF"
0.0 0 0.0
0
LOCAL
EOF
mkdir -pv ${SYSDIR}/sysroot/var/lib/hwclock
ln -sv /etc/adjtime ${SYSDIR}/sysroot/var/lib/hwclock/adjtime
```
　　这里设置为使用BIOS提供的时间，如果使用UTC时间，可以将文件中的“LOCAL”改成“UTC”。

### Shell
```sh
cat > ${SYSDIR}/sysroot/etc/shells << "EOF"
/bin/bash
EOF
```

### 分区挂载文件
```sh
cat > ${SYSDIR}/sysroot/etc/fstab << "EOF"
# file system        mount-point    type     options     dump  fsck_order

# /dev/<root_dev>      /            xfs      defaults     0       0
# /dev/<boot_dev>      /boot        ext2     defaults     0       0
# /dev/<efi_dev>       /boot/efi    vfat     defaults     0       0
# /dev/<swap_dev>      swap         swap     pri=1        0       0

EOF
```

### 创建首次运行脚本

```sh
echo '#!/bin/bash
if [[ $(find /etc/first-run.d/ -maxdepth 1 -name "*.sh") ]]; then
    for i in $(ls /etc/first-run.d/*.sh | sort -n ) ; do
        if [ -f "$i" ]; then
            chmod +x $i
            $i
            if [ "$?" == "0" ]; then
                if [ -f "$i" ]; then
                    rm $i
                fi
            fi
        fi
    done
fi

if [ -x /usr/bin/run-startx.sh ]; then
    /usr/bin/run-startx.sh &
fi
' > ${SYSDIR}/sysroot/etc/rc.local
chmod +x ${SYSDIR}/sysroot/etc/rc.local

mkdir -pv ${SYSDIR}/sysroot/etc/first-run.d/

cat > ${SYSDIR}/sysroot/etc/first-run.d/000-os-first.sh << EOF
#!/bin/bash -e
pwconv
grpconv
echo 1 > /var/run/first-run
EOF

cat > ${SYSDIR}/sysroot/etc/first-run.d/999-create-user.sh << EOF
useradd -m loongson -c "默认用户"
echo loongson:loongson | chpasswd
usermod -a -G video loongson
usermod -a -G input loongson
usermod -a -G wheel loongson
EOF
```

### 创建系统信息文件
```sh
cat > ${SYSDIR}/sysroot/etc/lsb-release << "EOF"
DISTRIB_ID="My GNU/Linux System for LoongArch32"
DISTRIB_RELEASE="0.1"
DISTRIB_CODENAME="Sun Haiyong"
DISTRIB_DESCRIPTION="My GNU/Linux System"
EOF
```

```sh
cat > ${SYSDIR}/sysroot/etc/os-release << "EOF"
NAME="My GNU/Linux System for LoongArch32"
VERSION="0.1"
ID=CLFS4LA32
PRETTY_NAME="My GNU/Linux System for LoongArch32 0.1"
VERSION_CODENAME="Sun Haiyong"
EOF
```

## 7 处理目标系统

### 清理符号（symbol）信息
　　目前安装到目标系统中的二进制文件大多带有各种符号信息，这些信息不影响执行，但是占用了大量的存储空间，如果没有调试相关的需求，可以将这些信息清理掉以减少存储空间。

　　清理符号信息可以使用strip命令，但strip必须能够处理目标平台二进制，所以我们可以使用交叉编译工具链中的strip命令来操作，操作步骤如下：

```sh
pushd ${SYSDIR}/sysroot
	find usr/lib{,32} -type f -name \*.a -exec ${CROSS_TARGET}-strip --strip-debug {} ';'
	find usr/lib{,32} -type f -name \*.so* -exec ${CROSS_TARGET}-strip --strip-unneeded {} ';'
	find usr/{bin,sbin,libexec} -type f -exec ${CROSS_TARGET}-strip --strip-all {} ';'
	find opt -type f -exec ${CROSS_TARGET}-strip --strip-unneeded {} ';'
popd
```

　　这里我们发现strip使用的参数有多种，这里简单的说明一下：  
　　* `--strip-debug`，仅去掉调试相关的符号信息，该参数适合用于静态库文件，对于链接过程需要的信息是不会去掉的。  
　　* `--strip-unneeded`，删除所有与重定位无关的所有符号信息，该参数不能用于静态库文件，否则会导致静态链接时无法使用处理过的静态库。  
　　* `--strip-all`，该参数代表所有能去掉的符号信息都尽量去掉，该参数不建议用于库文件，特别是静态库文件。

### 清除.la文件
```sh
find ${SYSDIR}/sysroot/usr/lib32/ -type f -name "*.la" -exec rm '{}' ';'
```

### 打包系统
　　制作完成后就可以退出制作用户环境了，使用命令:

```sh
exit
```


### 打包系统

　　接着可以使用root权限对目标系统进行打包，打包步骤如下：

```sh
pushd ${SYSDIR}/sysroot
	sudo tar --xattrs-include='*' --owner=root --group=root -cjpf \
			${SYSDIR}/loongarch32-clfs-system-0.1.tar.bz2 *
popd
```

## 附录

### 参考资料
《手把手教你构建基于LoongArch64架构的Linux系统》 https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md
