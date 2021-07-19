在tinycolinux32上装tinycolinux64 kernel和toolchain
=====

__本文关键字：高版本gcc cross compile 交叉编译低版本gcc,boostrap,为tinycolinux低版本linux kernel生成gcc，在32位linux cross build gcc target for linux64 execution，32位64位混合rootfs制作，运行cross build的应用。__

在《为tinycolinux创建应用包-toolchain和编译方法》中我们谈到gcc作为一套完善工具链的中心（编译套件），它从源码级支持被boostrap构建，和被外来地cross compile构建，一般地，当制作一个linux可用发版时toolchain支持是必要的。当然它也是难于构建的，它难于被构建是因为它绑定了binutils,kernel,libc这样的东西。且这些东西分别分布于三个关联平台，即：build平台,host平台,target平台 ----- build,host,target是GCC的三元组，它表达了这样一种通用工具链产生流程：用a gcc编译出能运行在b上的gcc，且这个gcc能产生c上运行的代码，这里abc依次即为build,host,target，GCC自举统一使用build平台已有的binutils,kernel,libc，且通常build=host=target。这里没有任何涉及到先有鸡还是先有蛋的问题（当然第一个GCC肯定不是GCC产生的，这其实是个演化问题），但是一旦涉及到cross compile（cross compile正是通用流程，bootstrap build只是个例）这里的复杂关联就无穷无尽了：这三个平台的位数，CPU类型，OS类型，OS版本，都不相同，具体GCC也要求不同具体版本的bin和平台上的header,和来自平台调用到libc的封装，GCC产生的程序需要运行在配有当初与GCC一起产生的binutils中的LD的host平台中运行等,如此种种,etc........ 没有一个single gcc source tree that with binutils,kernel,libc in a boundle能覆盖这些(当然指定版本到足够相符程度还是可以的)，通常情况下，我们都是依赖配置生成具体三位组所指的GCC TOOLCHAIN。考虑到复杂性，这也是为什么GCC这样的基础套件一般被设计成极度selfcontained的-仅引用binutils,kernel,libc，在以前的文章中我们还谈到升级libstdc++的情况，但要升级libc这基本很难，这也是因为作为基础的它们越基础的东西绑定越紧的原故，就像kernel一样。

好了，在以前的文章中我们一直使用的是3.x的tinycolinux32，现在，我们编译tinycolinux3.x 64和其完善的toolchain支持。其中，我们会涉及到比较多的坑。

cross compile tinycolinux3.x 64
-----

我们是在一台ubuntu14.04 64bit上的gcc485交叉编译出如下tinycolinux 3.x 64的(不要直接用tinycolinux上32位的gcc编译这个kernel):

参照《将tinycolinux以硬盘模式安装到云主机》一文的相似做法，我们从http://mirrors.163.com/tinycorelinux/3.x/release/src/kernel/下载64位的src和patch，打开virtualization中的virtio pci选项，编译进virtio block和network驱动，轻易在arch/x86/boot下得到bzimage，放到boot中启动。发现可以跟原有的rootfs一起正常启动。uname -m显示x86_64。file /boot/bzimage，显示x86 bootable kernel。猜这是因为在.config文件中同时开启了32和64支持，32位程序能运行在64位上，且原来的rootfs中的32位binutils和gcc未变。

如果把64位某linux的程序拷进来file它显示64bit elf，执行它会提示not found，这是因为它依赖的binutils ld没有，调用gcc -o helloworld.c -64m，提示unimplemented,这是因为3.x的rootfs是没有对应的GCC 64的。接下来需要cross compile一个：

在32 tinycolinux上boostrap compile gcc 4.4.3 64 三件套 for tinycolinux 3.x 64 target
-----


一般地，GCC支持从高向低crosscompile，反过来要难一点。所以这里我们采取最简单的方法：从同版本的32位GCC bootstrap编译出同版本的GCC，采用本地的32位gcc bootstrap式cross compile出64位的gcc，不再使用外来cross compile的方案（直接那样也行）。当然这种方案是设想了tinycolinux上本来就存在GCC的事实基础上，如果追求更通用的实践目的，还是从外面的系统cross compile进来好。

这样产生出来的GCC仅是一个target到x86_64-pc-linux-gnu的gcc 443版本，因为在本机上构建，所以这个build和host都不变，为本机系统HOST，但是并不影响我们的工作继续，至于以后你要用这个GCC作鸡生蛋蛋生鸡的事，比如可以用这个再次自举GCC443到host也为443的版本inplace覆盖，这都是以后的事。GCC支持从32到64或反过来的交叉构建。

我们选用2.x repos的make.tcz(3.81版，为什么不使用3.x的make 382接下来会涉及到)和选用3.x repos的gcc443 32位(为什么不用4.x的gcc471：因为4.x后采用eglibc，在编译很多程序时会遇到重复定义错误，这个时候就应该想到是版本问题)，走从GCC443 32位编译出GCC443 64的方案，要保证系统绝对干净，否则可能会遇到各种坑(比如cant computer object file prefix,etc..)，介绍一下制作纯净tinycolinux系统的方法：

按《在硬盘上安装tinycolinux》的方法重新安装rootfs，相当于重装系统，除了保留第一步的64 bzimage在boot下引导不变，你可能需要额外安装openssh。然后下载3.x的toolchain并安装:

 
```
sudo unsquashfs -f -d / /tce/gccbase/gmp.tcz
sudo unsquashfs -f -d / /tce/gccbase/libmpc.tcz
sudo unsquashfs -f -d / /tce/gccbase/mpfr.tcz
sudo unsquashfs -f -d / /tce/gccbase/ppl.tcz
sudo unsquashfs -f -d / /tce/gccbase/cloog.tcz
sudo unsquashfs -f -d / /tce/gccbase/binutils.tcz
sudo unsquashfs -f -d / /tce/gccbase/bison.tcz
sudo unsquashfs -f -d / /tce/gccbase/diffutils.tcz
sudo unsquashfs -f -d / /tce/gccbase/file.tcz
sudo unsquashfs -f -d / /tce/gccbase/findutils.tcz
sudo unsquashfs -f -d / /tce/gccbase/flex.tcz
sudo unsquashfs -f -d / /tce/gccbase/gawk.tcz
sudo unsquashfs -f -d / /tce/gccbase/gcc.tcz
sudo unsquashfs -f -d / /tce/gccbase/grep.tcz
sudo unsquashfs -f -d / /tce/gccbase/m4.tcz
sudo unsquashfs -f -d / /tce/gccbase/make.tcz
sudo unsquashfs -f -d / /tce/gccbase/patch.tcz
sudo unsquashfs -f -d / /tce/gccbase/pkg-config.tcz
sudo unsquashfs -f -d / /tce/gccbase/sed.tcz
sudo unsquashfs -f -d / /tce/gccbase/base-dev.tcz
#sudo unsquashfs -f -d / /tce/gccbase/gcc_libs.tcz
#sudo unsquashfs -f -d / /tce/gccbase/linux-headers-2.6.33.3-tinycore.tcz
```

然后下载以下并准备，都解压到一个目录。

http://mirrors.163.com/tinycorelinux/3.x/release/src/compiletc_other/ (mpfr-2.4.2.tar.xz,gmp-4.3.2.tar.xz,binutils-2.20.tar.xz)

http://mirrors.163.com/tinycorelinux/3.x/release/src/glibc-2.11.1.tar.bz2 (不用2.11.1了，到ftp.glibc.gnu.com下载2.12.1，以后有用)

http://mirrors.163.com/tinycorelinux/3.x/release/src/gcc-4.4.3.tar.bz2(从GCC-4.3起，安装GCC将依赖于GMP-4.1以上版本和MPFR-2.3.2以上版本。如果将这两个软件包分别解压到GCC源码树的根目录下，并分别命名为"gmp"和"mpfr" )

1）首先编译binutils:

```
cd binutils-2.20 && sudo make b && cd b
sudo ../configure --prefix=/usr/local/gcc443 --target=x86_64-pc-linux-gnu --disable-multilib
```

高版本GCC可加--disable-werror以免导致各种警告错误

sudo make
sudo make install

2）然后导出linux头文件到工具链: 

```
cd linux-2.6.33.3
sudo make ARCH=x86_64 INSTALL_HDR_PATH=/usr/local/gcc443/x86_64-pc-linux-gnu headers_install
```

要用到perl.tcz

3) 构建GCC工具框架，不带任何库。

```
cd gcc-4.4.3 && sudo make b && cd b
sudo ../configure --prefix=/usr/local/gcc443 --target=x86_64-pc-linux-gnu --enable-languages=c,c++ --disable-multilib
```

如果使用的3.x的make 3.8.2会出现configure错误：mixed rule

sudo make all-gcc
sudo make install-gcc

4) 生成glibc的基础部分

第三步已经将工具生成了，现在最重要的基础库的基础部分，注意还不是整个glibc

预先export PATH=$PATH:/usr/local/gcc443/bin帮助接下来的configure找到新编译出的x86_64-pc-linux-gnu-gcc，虽然configure会自动找到，手动一下更保险

```
cd glibc-2.12.1 && sudo make b && cd b
sudo ../configure --prefix=/usr/local/gcc443/x86_64-pc-linux-gnu --build=$MACHTYPE --host=x86_64-pc-linux-gnu --target=x86_64-pc-linux-gnu --with-headers=/usr/local/gcc443/x86_64-pc-linux-gnu/include --disable-multilib libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes
```

$MACHTYPE在正常的linux32上会输出i686-pc-linux-gnu字样，在tinycolinux上输出为空，继续

如上语句在tinycolinux上一次通过，但在普通linux上configure似首很容易把glibc源码目录被破坏，即使是cd 到b中，比如你也许会碰到：cannot compute suffix of object files或者： invalid host type: $CXX unregconnize -c，并不网上说的解决办法能解决的，往往重新准备glibc源码目录重新按上面的configure来配置就好了，在普通linux上，glibc源码目录下的scripts/gen-sorted.awk 19行以后会出现需要将\/[^/]+$ 改成 \/[^\/]+$的BUG，修正就好了。

然后就是make了：

a) sudo make install-bootstrap-headers=yes install-headers

在tinycolinux上一次通过，在普通linux上，你或许需要在make后额外加CFLAGS="-O2 -U_FORTIFY_SOURCE"  cross-compiling=yes以分别应付下列可能出现的错误。

cross-compiling=yes : No rule to make target `elf/soinit.os' error
CFLAGS=02 : glibc cant continues without opt error
-U_FORTIFY_SOURCE : inlining failed in call to ‘syslog’ error

如果在不纯净的tinycolinux上执行a)，可能会出现需要tls support error

b) sudo make csu/subdir_lib

如果在不纯净的tinycolinux上执行b)，会继续出错

c) install csu/crt1.o csu/crti.o csu/crtn.o /usr/local/gcc443/x86_64-pc-linux-gnu/lib

d) x86_64-pc-linux-gnu-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /usr/local/gcc443/x86_64-pc-linux-gnu/lib/libc.so

e) touch /usr/local/gcc443/x86_64-pc-linux-gnu/include/gnu/stubs.h

如果在纯净的tinycolinux上，可以无误一直执行到e)，一般到这接下来二步都能完成。

5)生成GCC的LIBGCC

重新cd gcc-4.4.3/b 
sudo make all-target-libgcc
sudo make install-target-libgcc

6)最后一步，生成完整的glibc和gcc中的stdc++lib 

重新cd glibc-2.12.1/b 
sudo make
sudo make install

重新cd gcc-4.4.3/b 
sudo make
sudo make install

其实如上三部曲的编译还有很多联合构建的选项。但是本文不深究了。

测试编译64位程序并运行
-----

 先写一个CPP的helloworld,test.cpp

```
#include <stdio.h>
int main() {
    printf("Hello World");
    return 0;
}
```

然后分别/usr/local/gcc443/bin/x86_64-pc-linux-gnu-g++  test.cpp -o a,/usr/local/gcc443/bin/x86_64-pc-linux-gnu-g++  -static test.cpp -o b,file a,file b，发现都是64位程序，我们发现b可以直接运行，而a显示not found，跟文章开头说的没有64位的GCC和binutils一样原因，那么现，我们讨求用新编译出的工具链让它运行的方法：

其实原因就是找不到共享库，error cant find share libs, ELF64CLASS，我们不能用32位的LDD分析它的依赖关系，但我们可以cd a所在的目录，x86_64-pc-linux-gnu-readelf -a ./a | grep "Shared"或x86_64-pc-linux-gnu-objdump -p ./a | grep NEEDED的方式查看，发现它引用了libc.so.6(->libc-2.12.1.so),libstdc++.so.6(->libstdc++.so.6.0.13),libgcc_s.so.1,libm.so.6(->libm-2.12.1.so)，当然了，这些库都要从新编译出的工具链的lib或lib64中找，放到a的目录，然后在a的目录下写个runa，加起执行权限，内容为：

```
D=$(dirname $0)
$D/ld-linux-x86-64.so.2 --library-path $D:$D/lib:$D/usr/lib ./a $@
(ld-linux-x86-64.so.2也是从工具链中找到的，它其实可以被执行，你也可以定制上面的--library-path)
```

执行./runa，输出跟静态b一样的结果。

这从理论上说明，只要系统支持默认的ld-linux-x86-64，它就支持运行一切由这个新工具链产生的程序。

-------------

现在，64位的kernel有了，生成64位程序的toolchain有了（它本身还是32位程序只是也能处理64位生成的事），但是整个ROOTFS还是基本上32上的，连运行它生成的64位程序的事都管不了，应该要重新编译busybox让它支持新的64位ld。

还有，可以以这个TOOLCHAIN为基础，不断bootstrap高版本的GCC，或者inplace覆盖，或变动BUILD，HOST进行，产生新的toolchains.

关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336718/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





