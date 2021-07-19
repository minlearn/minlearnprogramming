发布一统tinycolinux，带openvz，带pelinux,带分离目录定制（1）
=====

__本文关键字：tinycolinux上装第二套非标准路径下的gcc toolchain__

在前面《在tinycolinux上组建可子目录引导和混合32位64位的rootfs系统》一文中，我们讲到了定制可启动linux rootfs结构的可能：我们测试的是最简的情况，发现仅是修改busybox和启动脚本就可以达到目的，，，由于linux根文件系统大多基于busybox有同样的启动脚本流程，，于是在那文的文尾我们还谈到这种思路可以用于一般linux，比如tinycorelinux，只是在tinycolinux中，预想除了busybox,inital scripts，可能还要涉及到从kernel到glibc,甚至一些系统必要工具中的硬编码路径的修改（因为tinycolinux默认最简情况下由这些组成）。下面，就让我们来完成这个定制，并记录下整个细节。最终我们要得到从/system中启动的tinycolinux的发行版。

而且，，，而且我们还将一并完成某些额外工作，比如运用liveos技术运用在基于这个rootfs的一个linux发行版中，并希望整合openvz到这个定制内核。

为什么要这么做呢？如果一直关注我的《bcxszy part2》->《xaas》系列，读者一定会发现这是由colinux as xaas到纯tinycolinux路线的改变，比如原来由winpe(winpe+tinycolinux下busybox维护硬盘分区)变成了这里的统一live tinyclinux+tinycolinux hd，除了装机方案还有cui toolchain-----我们deprecated了《hostguest nativelangsys及uniform cui cross compile system》换成了统一的tinycolinux内的toolchain《在tinycolinux32上装tinycolinux64 kernel和toolchain》，且不久前还谈到《DISKBIOS：一个统一的实机装机和云主机装机的虚拟机管理器方案设想》，而在这里，我们还将进一步完善这个统一的tinycolinux内的rootfs和toolchain建设,且整合进openvz，使之变成一个高可用的综合xaas体。

所以最终，我们完成得到的是一个带pelinux和分离目录定制的rootfs和带虚拟化openvz的tinycolinux发行。

首先我们来准备这个liveos环境和一个带gcc443的tinycolinux硬盘系统环境，liveos部分会不断在接下来的定制rootfs的过程中被调用：

>按《将tinycolinux以硬盘模式安装到云主机》的方法安装一套硬盘tinycolinux 3.8.4到硬盘，menuentry tinycolinuxhd(/)启动脚本可写成：

>set root=(hd0,msdos1) set prefix=(hd0,msdos1)/boot/grub linux /boot/bzImage ro root=/dev/vda1 local=vda3 home=vda3 opt=vda3 tce=vda3

>准备microcore.gz一份也放到/boot下，作为liveos在grub启动脚本中被启动,menuentry "tinycolinuxlive"：

>set prefix=(hd0,msdos1)/boot/grub linux /boot/bzImage ro initrd /boot/microcore.gz

>有了这个livecd和二个分区，然后我们启动进liveos把整个/mnt/vda1下的/打包起来放进/mnt/vda3下的某个tar.gz中，在增删hd rootfs时有出错了我们随时可以在livecd中恢复回来，命令就不用提供了吧。

>再来准备我们要测试的对象rootfs：

>把备份的xxx.tar.gz解压一份到/system下，并用如下脚本设置启动menuentry "tinycolinuxhd(/system)"：

>set prefix=(hd0,msdos1)/boot/grub linux /boot/bzImage base ro root=/dev/vda1 local=vda3 home=vda3 opt=vda3 tce=vda3 init=/system/init

>注意它与上面grub条目的区别，这里是启动/system/init下的rootfs，这样，系统中就有二套hd rootfs,/system下为我们要得到的rootfs，这个/system我们要让它成为根目录下唯一文件夹也能启动tinycolinux

目前为止，这2个hd rootfs都可能启动，我们首先为这个新rootfs准备toolchain和基本的openssh支持。，过程可能要求你在前二个系统中不断reboot

编译另一套安装到/system下的工具链
-----

先按《在tinycolinux32上装tinycolinux64 kernel和toolchain》一文中准备bootstrap new gcc用的gcc443，并编译新的GCC，由于这里是在32位准备另一套装到/system的gcc不是那文中的装64位toolchain，，所以这里与那文中的步骤有类似但不完全相同，主要三件套中的某些步骤会有所不同（比如去掉了x86_64相关的参数，省掉了glibc三步步骤为一步），依次完成下列工作：

安装make.tcz 381,perl5.tcz
binutils : sudo ../configure --prefix=/system --disable-multilib
linux header : sudo make INSTALL_HDR_PATH=/system headers_install
gcc:sudo ../configure --prefix=/system --enable-languages=c,c++ --disable-multilib
export PATH=$PATH:/system/bin

定制glibc 2.12.1源码中的路径，将其中引用/lib和/lib64的部分改成引用/system/lib,/system/lib64，和一些由/移到/system的通用路径修正:

```
glibc-2.12.1\sysdeps\generic paths.h,ldconfig.h
glibc-2.12.1\sysdeps\unix\sysv\linux\x86_64 ldconfig.h
glibc-2.12.1\elf readlib.c
glibc-2.12.1\sysdeps\unix\sysv\linux\ia64 ldconfig.h 
```

以上更改主要是让新的glibc的ldconfig会发现/system下的lib。当然我们现在还处在旧rootfs中，还没有条件来测试具体引用过程。

编译时先在源码目录下mkdir b，cd b,需要建立一个configparms引导接下来的configure，文件内容为CFLAGS += -march=i486 -mtune=i686 -pipe，否则会出问题__sync_bool_compare_and_swap_4问题。

然后sudo ../configure --prefix=/system --disable-multilib sudo make install

然后就是整个gcc了。重新cd gcc-4.4.3/b sudo make sudo make install

由于libz极为基本，binutils中的as和openssh都要引用到libz，所以需要编译这个先，从3.x的release->src下取得libz的src，编译这个libz：

sudo ./configure CC="gcc -Wl,--rpath=/system/lib -Wl,--dynamic-linker=/system/lib/ld-linux.so.2" --prefix=/system

(注意不要拼错。否则出错，最好直接在winputty中直接右键粘贴，上面这句是用原rootfs中的gcc，新gcc中的库来产生libz)

然后是make381 src，同样用上面的configure同样构建make到/system下

然后依次是:openssl,openssh,openssh需要openssl,我们下载的是《在tinycolinux上安装sandstorm davros》一文中的版本，openssl的configure文件较特殊我们需要把其中的gcc:批量替换成gcc -Wl,--rpath=/system/lib -Wl,--dynamic-linker=/system/lib/ld-linux.so.2，然后

cd openssl-1.0.1/ sudo ./Configure（大写的） CC="gcc -Wl,--rpath=/system/lib -Wl,--dynamic-linker=/system/lib/ld-linux.so.2" --prefix=/system linux-elf sudo make install

openssh的编译 cd openssh-5.8p1/ sudo ./configure CC="gcc -Wl,--rpath=/system/lib -Wl,--dynamic-linker=/system/lib/ld-linux.so.2" --with-ssl-dir=/system --prefix=/system/usr sudo make 

这样整个新的GCC就好了，这样其它的扩展都可以由这个新GCC而来的而不再需要上面的一系列参数了。当然，这建立在一个假设上，我们最终要能boot进/system rootfs，且运行起/system下的gcc来编译程序。而我们还没能达到这个目的 - 我们还没有一个这样的环境。

我们先进入liveos打包一下/system到/mnt/vda3/systemgcc.tar.gz，然后我们重新启动到原rootfs，我们先来完成第一步，为能boot进/system new rootfs使用GCC和openssh改一些路径，注意以下工作依然是在原rootfs中完成的。

建立能用openssh和新gcc toolchain的新rootfs环境
-----

继续《在tinycolinux上组建子目录引导和混合32位64位的rootfs系统》下的工作，修改busybox中的硬编码路径，安装到/system下与toolchain一起，然后是大量的脚本，基本上，把能看到的原/下所有硬编码的路径都加个/system，这个工作量比较大，找到文本批量替换搞一下，注意能启动openssh的那些脚本要改好路径。修改后reboot到/system hd rootfs下，当然这样你是看不到失败的，因为这个/system依然引用大量原/ rootfs中的所有目录。我们的最终目的是脱离所有的/下的文件夹能启动。

注意到lib是openssh和gcc都要引用的目录，这是我们消除的原rootfs下所有目录的第一个目录。以便做到删除原rootfs/lib也能启动openssh和使用gcc，

我们先来完全删除/lib，发现进入/system rootfs失败，没事，失败了可以reboot进livecd还原。

最终，精简/lib至安全版本，仅含几个文件：ld-2.11.1.so,ld-linux.so.2,libc.so.6,libc-2.11.1.so,libnsl.so.1,libnss-2.11.1.so,libnss_dns.so.2,libnss_dns-2.11.1.so,libnss_files.so.2,libnss_files-2.11.1.so

我们也不知道这些文件为何还在引用，无论如何，新的/system rootfs终于能进了能tc登录了，openssh终于可以成功了，不过有好多errors和warnings。

先用起new GCC来，方法是：重新解压那个第一次制作的系统打包，并解压systemgcc.tar.gz到其中，这样就是一个全新的系统了。 再删除原来的/usr/local下的gcc，/usr/bin/ar的链接去掉，去掉旧的GCC引用，这样，新的gcc在export PATH=$PATH:/system/bin后终于可以用了。

当务之急是，利用这个新GCC，进去再编译几个工具。

利用新gcc编译tinycolinux rootfs必要工具
-----

以下源码全在3.x release src下下载，以下编译过程会自动使用新的/system/lib：

sudo gcc rotdash.c -o rotdash sudo gcc autoscan-devices.c -o autoscan-devices

e2fsprogs的编译:cd /system/mnt/vda3/tce/e2fsprogs-1.41.11/ sudo ./configure --prefix=/system sudo make install

sudo的编译:cd /system/mnt/vda3/tce/sudo-1.7.2p6/ sudo ./configure --prefix=/system --without-pam (注意这个withoutpam) sudo make install

最后是udevm这个东西，比较烦琐，基本上这个引用是这样的：udevm需要gperf,acl需要attr,attr需要gettext

a) 第一次尝试configure udevm，发现不了gperf
cd gperf-3.0.4/ sudo ./configure --prefix=/system sudo make install 
b) 第二次尝试configure udevm 
cd gettext-0.17/ sudo ./configure --prefix=/system sudo make install
 
由于attr,acl的configure风格比较相似，文件都不够聪明，所以我们需要指定它引用到的工具路径给它
sudo unsquashfs -f -d / /system/mnt/vda3/tce/gccextras/libtool.tcz

cd attr-2.4.43/
sudo ./configure --prefix=/system MAKE="/usr/local/bin/make" LIBTOOL="/usr/local/bin/libtool" MSGFMT="/system/bin/msgfmt" MSGMERGE="/system/bin/msgmerge" XGETTEXT="/system/bin/xgettext"

并且它的安装要二步完成:sudo make install sudo make install-dev 

好了，由于我们把attr库安装到了system/lib,system/include下，所以:

cd acl-2.2.52/ sudo ./configure --prefix=/system LDFLAGS="-L/system/lib" CPPFLAGS="-I/system/include" 

但是这样是不行的，我们还是乖乖地将attr多加一次安装到/usr/下以便现在这个gcc发现。

c) 第三次尝试configure udevm，需要lusb 

cd libusb-1.0.8/ sudo ./configure --prefix=/system cd usbutils-002/ sudo ./configure --prefix=/system

发现不了pci-ids数据文件，但usb-ids有了,直接下载3.x中的lpci.tcz解压上传吧。

lusb和lpci风格比较相似，

sudo ./configure --prefix=/system LIBUSB_LIBS=/usr/lib LIBUSB_CFLAGS=/usr/include LIBUSB_LIBS=/usr/lib/libusb-1.0.la LDFLAGS=-lusb-1.0

发现不了usb.h，下载lusb的tcz，把usb.h放进来 

d) 第四次尝试configure udevm，发现需要glibo 

sudo unsquashfs -f -d /system /system/mnt/vda3/tce/libffi.tcz 
sudo unsquashfs -f -d /system /system/mnt/vda3/tce/gobject-introspection.tcz
sudo unsquashfs -f -d /system /system/mnt/vda3/tce/glib2.tcz

可见，如果不将一切都编译到/system/lib，会导致对原/lib和原rootfs依然有依赖。

e) 第五次尝试,完

sudo ./configure --prefix=/system LIBUSB_LIBS=/usr/lib LIBUSB_CFLAGS=/usr/include LIBUSB_LIBS=/usr/lib/libusb-1.0.so LDFLAGS=-lusb-1.0 --disable-extras

还有一个工具rzcontrol不弄了

基本上，这样之后，/system roots能很好启动了（较以前少很多errors,warnings）。

到现在为止,/lib依然没有没完全精简掉。

你需要像对待精简/lib的过程一样对待其它文件夹。一步一步来。先找出它的所有硬编码可能存在于原rootfs中的某处依赖。改正它。尝试删除它，反现不行就在liveos中改回来继续尝试。

最终完成精简掉整个原rootfs /下文件夹的目的。 

----------------

好了，其它的工作就是逐步修改，不断reboot to liveos备份增量集成修改的过程了，最终你会得到一个无误完善版本的rootfs

我发现这个新rootfs，以前登录winscp或winputty会停顿的现象现在不会了哈哈，不知是不是编译openssh时disable pam了的结果。。

下一篇我们将在这个tinycolinux hd中装openvz，另外，克服程序员技术更新过快和后天健忘（要记的东西太多了）的特点：只能是写博记录。或微博记事。

其它的留到本文第二篇再写吧。

关注我



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337218/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




