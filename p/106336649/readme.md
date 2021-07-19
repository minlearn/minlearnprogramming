将tinycolinux以硬盘模式安装到云主机
=====

__本文关键字：tinycolinux安装到阿里云主机，tinycolinux实机硬盘模式安装,vs Frugal tinycorelinux Scatter模式安装,重新定制tinycolinux的rootfs__

在《发布tinycolinux代替docker》一文中，我们将colinux和tinycorelinux结合，打造了一个tinycolinux并装在了windows host上，只要主机装了windows，那么实际上就可以装tinycolinux as guest，这对实机和云主机是无区别的，因为二者都可以装windows。

那么tinycorelinux，如何实现其在实机/云主机以standalone模式安装呢？

在《为tinycolinux发布应用中》我们提到tinycolinux的rootfs:microcore.gz，那里我们对它有一些优化意见，但在那里我们还不想定制它，那么现在我们要面临这个实际问题了。

测试livecd模式和寻址问题
-----

实际上参照tinycolinux as guest for windows的方案和《利用tinycolinux在云主机上为linux动态分区》一文中安装grub2的过程，我们已经有思路了，即我们完全可以在云主机上创建一个包含microcore.cpio内容的grub2 as bootload的分区，然后参照windows host/colinux guest中利用vmlinux和microcore.gz的方式去尝试驱动它，实际上这是完全可能的。

我们先来说livecd模式安装，，即tinycolinux的frugal模式安装。因为在这个基础上可以一步一步很好测试以后的scatter模式是否能成功：

即按《利用tinycolinux在云主机上为linux动态分区》一文中利用virtiope+tinycolinux no image的方法分二个区，第一个区作为系统区并bootice刻上grub2的mbr，然后解压g2files.tar.gz，做/boot/grub的文件结构，把http://mirrors.163.com/tinycorelinux/3.x/archive/3.8.4/distribution_files/下到的bzImage和microcore.gz放进/boot，grub.cfg菜单就写成：

```
menuentry "tinycolinux" --unrestricted {
set root=(hd0,msdos1)
set prefix=(hd0,msdos1)/boot/grub
linux /boot/bzImage ro root=/dev/vda1 local=vda3 home=vda3 opt=vda3 tce=vda1         ;云主机发现2个盘
initrd /boot/microcore.gz
boot
}
```

这样是完全可以驱动进云主机的，但我们很快发现这始终只能让initrd中的内容成为根，进一步上传从microcore.gz中解压出来的microcore.cpio到/mnt/vda1/，cd /boot/，cpio -idmv < microcore.cpio,这时vda1中已经有可以工作的文件系统了，但是重启，去掉或保留那条initrd /boot/microcore.gz ，都不能使菜单中的root=/dev/vda1起作用（去掉会让云主机提示cant mount vfs as root，找不到盘启不动，而本来linux是可以不用initrd启动的.）。目前为止这样的livecd于主机来说不实用，由于livecd写入到根的东西都是占内存的，且由于一些未知的原因（我是不想追究了），我们发现GCC是无法在这种livecd中运行的。

这是因为vmlinux开机时发现不了virtio云硬盘，所以不能这样启动，不同于其在windows hosted的情况下可以在配置文件中直接定义/dev/cobd1=/dev/disk/partion,root=/dev/cobd1。

我尝试用bootloader grub来启动vmlinuz,即在grub.cfg中set GRUB_CMDLINE_LINUX="root=/dev/vda1 rootfstype=ext3"，同样发现不行，看来， 

要寻求传统的硬盘根文件系统启动的方式，scatter模式，只能寄希望于先编译出一个支持virtio inside，能在开机时就能发现硬盘并挂载的vmlinuz：

编译virtio驱动模块到tinycorelinux bzimage/vmlinuz
-----

我使用的版本是tinycorelinux 3.8.4，从http://mirrors.163.com/tinycorelinux/3.x/release/src/kernel/处下载config-2.6.33.3-tinycore和linux-2.6.33.3-patched.tbz2

由于在config中集成驱动，各个选项有复杂的依赖关系，是不能直接修改.config文件的。所以进make menuconfig，未尾加载那个config-2.6.33.3-tinycore，按一下/，输入virtio查看依赖关系，发现跟virtualization有关，好了，进入打开，如果你直接在network driver中打开virtio network的y选项会提示有依赖关系，block driver中的virtio block driver也一样，只有解决了依赖才能进行。 

然后make mrproper（如果你进行了多次构建尝试，执行一个这个比较好）由于我在gcc481下编译的，所以vdso makfile会提示找不到i386等等，此时按《在colinux上编译openvpn》上处理的方法一样将里面的某句改成m32,m64，继续，得到bzimage在/arch/x86/boot。改个名放进livecd模式下的/boot/中，sudo reboot，在系统启动时进入grub命令行，改菜单，去掉initrd，用新的bzimage名代替bzimage，提示发现vda1，但又出现：runaway loop modprobe binfmt-464c的问题，无论如何，我们问题完成了一半。

网上说这可能是位数冲突，可能我使用编译bzimage的是个64位的ubt主机导致的，于是换回colinux+gcc461编译： 

colinux下make meunconfig会用到term设置：export $TERMINFO=/usr/share/terminfo，且要安装ncurse和perl5.tcz，安装，重复make menuconfig继续编译得到bzimage，继续上传放进云主机/boot中测试，问题解决！！

然后就是那个定制cpio的问题了
-----

其实这在硬盘模式下可以直接定制根文件系统逻辑了，对于打包的microcore.gz，则可以这样定制再打包，这称为remaster：

```
cd /mnt/vda1/boot/test
ls . | sudo sh -c 'cpio -oH newc -d > ../test.cpio'
不要在boot目录使用find ./test，会保留test
sudo gzip ../test.cpio
```

如果不用sh -c，会出现sudo之后依然无权限，我的busybox cpio是version v1.19.0，仅支持使用以上newc格式。。

------------------

其实，我刚一开始的解决方案是企图通过定制microcore.gz来改那个/etc/init.d/tc-config，把挂载到livecd根下的所有目录通过类opt,home,usrlocal,tce的方式挂载到目标硬盘，这样在有硬盘时使用硬盘，没有硬盘时就使用livecd。livecd模式也不是完全没用的，livecd模式也称为装机模式，，忘掉它云端模式的另一个名字吧我感觉没什么大用，只在这个装机模式下，它整个都在内存，所以即使tinycorelinux initrd所在硬盘可以拿来格掉/覆盖，且tinycorelinux比起virtiope来还有一个联网功能，云端模式的名字就来源于此，其实完全可以代替virtiope的所有功能，打造类mac电脑在线恢复系统的功能，我未来会把它做进diskbios--一个整合化的装机系统。

还有，linux kernel一个微小中心+shell脚本的多元发行设计使得其发行版很常见，运用到语言的设想就是用terralang这样的东西组装langtech级可定制组装/剪裁的发行版语言系统，这样就不需要聚集于传统的库方式，也不需要大量非C的脚本DSL了，当然，当这个terralang是terracling的时候就是这样，因为terralang中的lua是非C的。。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336649/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



