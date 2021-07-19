在tinycolinux上组建子目录引导和混合32位64位的rootfs系统
=====


__本文关键字：mount subdirectory as linux root,boot linux from root subdirectory，从子目录引导linux root，separated system and usr extend under linux root__

在前面《在tinycolinux32上装tinycolinux64 kernel和toolchain》中我们讲到了组建一个linux发布版的二大基本部件：kernel和toolchain部分，虽然通常提到linux发行指的是一个包含了所有打包的linux ---- 体积外观上最大的主要是其rootfs部分，即那个/下的部分，，但往往kernel才是一个发行版的表征：它提供了能bootable起硬件使之变成OS的部分，它定义了PC能带起什么硬件，能支持几位程序的部分，基础之二的toolchain是支持这个表征放大的基础：它提供了用户能开发和运行应用以扩展这个OS的支持部分---- 它们属于从外而内产生一个linux发行版的必要和基础部分------- 这二部分要先行从外部cross built而来，相对来说，rootfs只是kernel要搭配起哪些toolchain产生的程序完成什么工作的事，所以可以放在以后，甚至一个至简的rootfs就是一个busybox+一些init脚本就可以了。

本篇我们将讨论这个rootfs组建的过程，以测试运行前文产生的kernel和gcc toolchain产生的程序的过程。最终的目的，将会是一个支持64位/32们混合的文件系统，和一个高度自定义，system和用户扩展文件夹分开的，这样一个linux发行版。

>这究竟会是一个什么样的LINUX呢？ 

>现在的linux发行版，基本是根文件系统挂在/下的，这样一个发行版就占用一整个硬盘分区，外观上也很不雅观，业界竞然也没多少人注意到这个问题，要是能进行一下改造：在不破坏这个根目录是挂不挂在/下这个事实的基础上，如果我们能让系统从/下的一个子目录启动就好了。比如从/system启动。这样有很多好处，外观清爽不说，还可以在一个分区中准备多个发行版并从中引导运行（有没有一点像虚拟化？），每个rootfs对应一个发行版/system1,/system2,etc..，这样还可以被独立打包备份。除了/system1,/system2,我们还可以有与/systemx并列的/usr，/usr就是用户扩展，应用安装后所在的目录，里面会有bin,include,lib这些用户扩展。这样就分开了传统linux发行版将所有一切放到/傻傻不分的情况了，最终/下会有/boot,/system,/usr三个文件夹，如果说我们可以有混合32/64文件系统，那么在这里，我们甚至可以混合使用这三个文件夹的组合，达到一种“system和usr extend能分开能组合的高度自定义化文件系统”。

>上述说法中，承认我们没有破坏根目录挂载在/下的事实是很重要的，因为我们仅是想做个trick，让系统文件归档在/system下使之变得好看，并做到能启动就好了，事实上，这仅是改造busybox的事我们的目的就能达到 - 仅是改造busybox源码中硬编码的文件路径，并不需要关注“改造根目录挂载位置”这样重大的问题，而那其实是一个难度甚高的事情。比如可以利用initramfs+privo root达到。---- 但这样的方案就有点重了。

 好了，让我们来看是怎么回事吧，先来看32/64位混合文件系统。

在tinycolinux上组建32/64位混合文件系统
-----

在《在tinycolinux32上装64位toolchain》文中，我们提到产生的64位程序不能运行，甚至ldd都不能分析出其引用，仅提示wrong elf64class，直接执行也提示not found,这是因为它找不到64位共享库，由于ldd无法使用，我们通过其它手段分析，发现最终原因其实是因为默认64位GCC产生的glibc，将GCC产生的程序对loader，即ld-linux-x86-64.so的引用，放在了/lib64中（至于其它基础库libc-2.12.1.so，libcrypt-2.12.1.so，libm-2.12.1.so，libpthread-2.12.1.so，你可以把它做起对应软链一同放在/lib64中，其实不做也可以，因为它们被引用在了/usr/local/gcc443/x86_64-pc-linux-gnu/lib这个是由编译工具链时hardcoded指定的，还有libstdc++.so.6.13,libgcc_s.so.1也要放/usr/local/gcc443/x86_64-pc-linux-gnu/lib）。因此我们仅需：

首先把这个文件和它引用的真实文件ld-2.12.1.so复制到/lib64下，并把ld-2.12.1.so加起执行权限来（这个至关重要，否则会提示access corrupt shared libraries），然后把上述的文件各自复制到其所在目录。执行64位测试程序，发现能成功运行！

这样，tinycolinux就拥有了二套GCC支持开发和运行的程序，所在的文件系统，一套在/lib下，一套在/lib64下。分别同时支持32位和64位。

在tinycolinux上组建system和usr extend分开的高定文件系统
-----

还记得我们开头谈到至简的rootfs就是busybox+一些init脚本吗，我们不断提到的busybox是一个产生rootfs的基础和中心，总管，它自包含我们建立这个测试环境需要的一切，我们来使用它建立这个最简的rootfs样本：

我们是在tinycolinux本身带有GCC481的环境下测试的，为了方便测试使用云主机，使用快照随时准备备份恢复重来，使用的tinycolinux它自己就有rootfs。根目录下有个init,/bin下有个busybox，注意到这些细节后，我们来构建自己的busybox：

首先下载busybox源码http://mirrors.163.com/tinycorelinux/3.x/release/src/下载busybox-1.19.0.tar.bz2,busybox-1.19.0-config和9个patch并运用到解包后的源文件,按《tinycolinux上硬盘安装》一文准备sudo make menuconfig并运用config，为了我们的分离式文件夹系统，busybox事先是被静态链接的，静态链接可以免去对lib目录的依赖,且编译menuconfig配置时设置了把/system/usr/bin,/system/usr/sbin一起合并到/system/bin,system/sbin中， --- 因为lib我们要将其做在usr文件夹中，与system,boot并列，sudo make install 编译好后复制_install为根目录下的/system，那个/system/linuxrc不要删。

然后我们按照《将tinycolinux安装在硬盘上》一文中的grub启动/boot下的kernel，具体我们测试用的kernel启动参数是：linux /boot/bzImage ro root=/dev/vda1 swapfile=vda1 local=vda3 home=vda3 opt=vda3 tce=vda3 init=/system/init，，，完整的grub菜单文本请参照那文查看。注意到init=/system/linuxrc，这是新加的一条参数。它定义了系统在引导系统时发现root=/dev/vda1后，完成系统将执行权交给PID0来初始化文件系统的那个PID0，root只能是设备，对应文件系统中的/，而init pid0可以是/下任意路径下的一个可执行程序，一段脚本。这段参数其实就是kernel转手给通往rootfs init的连接器(其实你可以patch kernel中的init/main.c让它加载你自己的init)。

业界有很多复杂化的init，如systemvinit等，tinycolinux也定义了它的脚本化init，在tinycolinux中，init是根下的init是一段脚本，但对于简单的init，你可以将它直接链接到busybox中的init，在我们的测试环境，就是这么用的:linuxrc链接到system/bin/init,busybox init仅链接就能作为init使用是因为它其实包含了一系列默认动作，就像传统init能做的那样：它首先会查找etc/inittab，这个文件可以没有，没有的话,busybox init会执行/etc/init.d/rcS，在这里它要执行一些必要工作，所以我们还要准备一些把busybox当init用脚本和作一些初步工作：a)在/system下建立dev，etc，proc,sys四个空目录，b)dev下准备二个设备文件 mknod console c 5 1和mknod null c 1 3，然后: c)etc下提供fstab,inittab,init.d/rcS，其中inittab,rcS都加起执行权限,内容分别为:

fstab:

```
proc /system/proc proc defaults 0 0
sysfs /system/sys sysfs defauts 0 0
```

inittab:

```
::sysinit:/system/etc/init.d/rcS
console::respawn:-/system/bin/sh
::ctrlaltdel:/system/sbin/reboot
::shutdown:/system/bin/umount -a -r
```

init.d/rcS:

```
#!/system/bin/sh
/system/bin/mount -a
```
 

好了，仅是这样就OK了(你可以先不用/system，将上面的rootfs打包成initrd.gz在普通方式下测试，证明这个文件系统是完善的，最终结果是进入无误进入命令行。)。下面我们试着让基于附加了/system的rootfs运行，直接改名原来tinycolinux的/bin/busybox，让新的busybox生效，继续如下测试，如果失败有下列原因之一，在下列失败可能和解决方案循环间不断恢复云主机重新尝试：

1)失败可能:

提示kernel panic，说提供的init不可执行，系统尝试执行tinycolinux /下的默认init

warning:cant start default console

sbin/gettty not found之类之类

可以看到sh，但ls ,which not found

可见光是脚本文件不足于影响busybox的行为，由于我们企图将etc这些东西归类到/system/etc下，所以我们需要定制busybox中的路径硬编码部分以继续测试: 

2)解决方案：

改动源码：

include/libbb.h

```
1690: define bb_default_path      (bb_PATH_root_path + sizeof("PATH=/system/bin:/system/sbin:/sbin:/usr/sbin"))
1717: #define LIBBB_DEFAULT_LOGIN_SHELL  "-/system/bin/sh"
1725: #define CURRENT_TTY "/system/dev/tty"
1726: #define DEV_CONSOLE "/system/dev/console"
```

init/init.c

```
137: #define INIT_SCRIPT  "/system/etc/init.d/rcS"   //为全程定制busybox 
init的行为起见，这句必须要搭配下面一句inittab使用
638: parser_t *parser = config_open2("/system/etc/inittab", fopen_for_read);
684: tty = concat_path_file("/system/dev/", skip_dev_pfx(tty));
996: putenv((char *) "SHELL=/system/bin/sh");
1018: new_init_action(SYSINIT, "mount -t proc proc /system/proc", "");
```

继续编译。

cd /tce/busybox
sudo make clean
sudo make install
sudo cp _install/system/bin/busybox /system/bin

不断测试，最终成功。系统启动过程无误，最终出现正常命令行。当然还有很多需改动使这个rootfs变得更完善的空间。

 ----------------

为了维护这套干净强大的文件系统设计，用户要注意在编译程序时将其产生到/usr下，永远不要采用./configure 默认无prefix的情况。

你可以整合tinycolinux的现有init逻辑，把tinyclinux的根文件系统改造成高定文件系统，以如上在tinycolinux内部循序渐进地改动进行的方式。

关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336743/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




