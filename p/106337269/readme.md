发布一统tinycolinux，带openvz，带pelinux,带分离目录定制（2）
=====


__本文关键字：从0开始搭建linux rootfs,在tinycolinux上安装openvz，openvz源码编译任意linux__

在前面《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制（1）》中，我们在tinycolinuxhd上集成了pelinux，使之成为一个pre install的linux维护环境，而且我们还改造了其rootfs，使之成为一个文件结构更合理化的tinycolinux发行版，在本篇我们将实现将openvz也集成到这个tinycolinux，实现将其基本变成一个完全适合用单机和集群服务器环境的tinycolinux的承诺，由于进行了深度定制，一路上碰到的大小坑不断，不过实践证明这完全是可行的。不费话了，继续吧：

更彻底的 clean /system rootfs
-----

1)去掉/lib的方法：

前文我们谈到那个/lib中至少需要保留几个文件，这是因为busybox有一条配置CONFIG_USE_BB_PWD_GRP:

If you enable CONFIG_USE_BB_PWD_GRP , BusyBox will use internal functions to directly access the /etc/passwd, /etc/group, and /etc/shadow files without using NSS . This may allow you to run your system without the need for installing any of the NSSconfiguration files and libraries.

只要将它打开编译进BB，就可以删掉这些文件了，纯粹以/system/lib下的文件启动。只是/etc下的passwd,shadow还在被引用，待以后处理。

2)去掉/bin的方法:

登录tc时login总是引用/bin/sh，而不是/system/bin/sh 

这是因为没有用上/system/etc/skel中的登录脚本，在其中设置好PATH变量到/system/bin,/system/sbin，并保证内核传过的home=vda3正确挂载在了/mnt/vda3/home/tc（保证其中的内容是由/system/etc/skel中复制过来的），然后系统会自动用上/system/bin/sh

以后，我们将陆续去掉/下其它文件夹。

但本文以测试openvz为优先，接下来的测试，会有tcz不断被安装到/usr/local/bin。所以这一节与接下来的一节可以不相关。用户最好在原来的/ rootfs的tinycolinuxhd中进行，接下的讲解也是默认以原先的/ rootfs的测试环境为准进行的。

构建openvz内核和支持工具
-----

谈到从源码构建openvz2.6 series针对任意发行版本，网上的文章都是从https://openvz.org/Download/kernel/2.6.32/2.6.32-feoktistov.1/下载的，有完整的patch打包和.config配置示例，不过我下载的时候都显示404，就连各个mirrors中的这些文件都失效，只好从它的git中下载了https://src.openvz.org/projects/OVZL：这个版本中的kernel是经过patch的，我们需要linux-2.6.32-openvz 74c87ab8a48 ,ploop d42955558fb,vzctl 8b9e1c15fce,vzpkg a05e9f503f9,vzquota 5834f7a1fcb，这几个repos，注意二点，1） 一定要git，不要下到windows打包再上传到linux，这是为了防止稍后编译kernel会出现unkown option error，2） 由于/system rootfs的tinycolinux没有了tmp，而kernel编译脚本中默认还是使用/ rootfs中tmp的，所以暂时恢复建立tmp否则gen initramfs_data.cpio error 1

首先来编译kernel，我们需要tinycolinux 2.6.33中的.config，为了免除更多工作，我们没有将tinycolinux 2.6.33中的patchs搬过来打补丁，实际上tinycolinux对原版vault的kernel 2.6.33也没进行太多影响大局的修正。因此，我们完全可以使用openvz的kernel在其上工作，好了，现在进入src root,make menuconfig，进入发行openvz选项，加载2.6.33的.config，virtio按以前文章中的处理方式处理编译到内核，openvz中的项是星是模块的都不要改动，因为openvz tools会默认以加载模块的方式来启动openvz，sudo make开始编译kernel，完成后在arch/x86/boot中找到bzImage，复制到boot，写个menuentry让其在下一节稍后启动,sudo make modules_install，把安装到lib/modules/tinycolinux-2.6.32.8中的kernel文件夹改为build，因为编译出的kernel要从那里找modules

然后是工具部分，准备安装autotools包含libtool.tcz，vzctl需要用到，这个libtool-dev.tcz也需要，否则在编译vzctl时./autogen.sh时，会出现LToptions not found，sudo ./autogen.sh开始配置过程，发现需要libxml2-dev.tcz,liblzma.tcz，将libxml2.tcz和lzma unsquashfs -f -d安装/进去sudo ldconfig (以前文章中我们不断提到安装了*-dev.tcz的tinycolinux需要重启使lib文件生效，但实际上仅需要这一句就可免重启)，继续，发行还需要ploop，编译ploop不成功。但是没关系。仅需要头文件。继续，还需要libcgroup，但是好像vzctl内含libcgroup，我下载的是sourceforge中的libcgroup-0.41。cgroup配置过程中需要yacc，用flex,bisn也可以，make install后sudo ldconfig

好了，现在可以安心编译vzctl了，sudo ./autogen.sh，需要gmp,pam，一一解决，直到出现配置正确完成。make install会安装主体vzctl文件和支持脚本,make install-debian会安装vzctl控制openvz自启的脚本，由于debian最接近tinycolinux，所以我们安装这个。然后是接下来的vzpkg和vzquota，相对比较简单，直接sudo make install就可以了

到现在为止，内核和工具部分都编译完成了，由于tinycolinux是特殊版，为了稍后能启动，所以我们还需要稍微定制一下：

lsmod在bin下不在sbin下,将它复制到sbin下, 手动建一个/var/lock/subsys,coreutils.tcz中含的stat(coreutils需要libcap,acl,attr,gmp，下载到的attr不含so，按前文的方法编译得到so或直接复制这个so和二个链接出来到/lib下),bash.tcz(直接打开提取上传到/usr/bin),iproute中的ip command都需要否则会出现ip command not found,/etc/init.d/vz脚本也稍微修改一下，将其中sysctl -p后面的-p去掉，/opt/bootlocal.sh加一段使vz随系统自启的脚本，与openssh自启条目放一起，内容为：

```
rm -rf /var/lock/subsys/*
/etc/init.d/vz start
```

现在重启。进之前为新kernel建的grub menuentry，让系统加载新kernel并尝试接下来启动整个openvz

启动openvz并建立一个vps
-----

好了，如果重启后显示running kernel is not an openvz kernel就表明没有进入正确的kernel，反之输出openvz的一系列信息直到显示openvz is running(上面bootlocal.sh脚本中可以加个>null使之不显)则可以继续下一步测试：

下载一个template，我选择的是ubuntu-12.04-x86-minimal.tar.gz，上传到/vz/template/cache,然后安装它：sudo vzctl create 101 --ostemplate ubuntu-12.04-x86-minimal --layout simfs --config basic，发现busybox默认提供的gzip和tar都缺少选项功能，显示invalid option --l等错误。从http://ftp.gnu.org/gnu/下载tar-1.30和gzip-1.9.tar,编译tar ./configure FORCE_UNSAFE_CONFIGURE=1，和gzip替换busybox中的旧版本，再次执行上面的sudo vzctl create，正确完成！

启动sudo vzctl start 101,sudo vzctl enter 101，显示ubuntu的root用户的命令行。完工！

 ------------

我们其实可以把vz映射到/mnt/vda3/vz，像tinycolinux的local,tce,usr/local,opt文件夹一样。我们还可以定义自己的os template，这个os template同样可以是基于tinycolinuxhd的。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337269/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



