在tinycorelinux上安装lxc，lxd (1)
=====

__本文关键字，在tinycorelinux上安装lxc，lxd,gcc4.4 self-reference struct typedef__

在前面的文章中我们讲到过内置虚拟化的os设计，它可以使包括裸金属，云主机在内的等所有虚实主机实现统一装机，且统一各层次的虚拟化(os容器/app容器一体化)，这就是diskbios->cloudbios的设想，在一些文章中我们还讲到利用这样的一套架构打造vps server farm(最初我们尝试的是在windows上利用colinux打造vps farm),甚至打造一个portable cloud environment image file的思想。

那时我们考虑的主要是单纯的xaas：类coreos，但是更偏向接近native的去虚拟化，我们为此建立了一个极小的linux distro，在《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制》系列中我们实现了这样一个linux distro的基础部分：dbcolinux。

在更稍晚的文章中，我们发现了devops,如，《利用packer把dbcolinux导出为虚拟机格式和docker格式》谈到的Provision a dbcolinux img。。我们在《hyperkit：一个full codeable,full dev support的devops,及cloud appmodel》还谈到一种利用轻量hypervisor来做虚拟层的工具和devops OS设计：bhyve。我们甚至还谈了支持编程扩展DSL和devops的语言。

最后，考虑到虚拟化和devops结合，结合四大件，我们承诺实现一个带xaas,langsys,domainstack的全功能到busybox的设想，这里的思想是这样的：—— 基本上，只要把这些xaas,devops尽早尽量上升到上流，而且保持尽量小。就可以在下流domainstacks,apps中再度被抽象。而busybox设想的最终目的，是要实现一个native/cloud可以本地远程无差运维开发的appstack,这是后话。

但是现在，为了得到一个这样可用的东西，我们会采用一些更为实际的方法。—— 比起使用真正的hypervisor，我们可能会继续使用ovz这样的东西，这是因为深思之后考虑到：一些云主机没有intelV硬件功能不可再虚拟化，而且，hypervisor它也比较重。虽然bhyve比较轻量不过比起os级的虚拟(openvz,etc)来说还是比较重，而且它只工作在freebsd，

> 分清二种平台虚拟化containerisation vs virtualisation：

> 拿bhyve的衍生品smartos来说。

> SmartOS is a specialized Type 1 Hypervisor platform based on illumos.  It supports two types of virtualization:
> OS Virtual Machines (Zones): A light-weight virtualization solution offering a complete and secure userland environment on a single global kernel, offering true bare metal performance and all the features illumos has, namely dynamic introspection via DTrace
> KVM Virtual Machines: A full virtualization solution for running a variety of guest OS's including Linux, Windows, *BSD, Plan9 and more

Ovz做的主要是第一种，而kvm,bhyve是第二种，我们偏向采用os level的虚拟化，因为它也能devops，而OVZ的devops功能不足。所以我们考虑用lxc/lxd来代替ovz，它的优点有：

1,lxc兼容linux 2.6之上,利用linux本身机制，与docker技术统一。2,lxc作为操作系统级的containerisation技术，它的使命却在于模拟普通虚拟机和hypervisor。3,它也有LXD这样的上层，LXD is a container "hypervisor"。可以Provision生成，甚至休眠。缺点：资料少,有一些与虚拟机的功能不能一一对应，缺失

我们先来讲在dbcolinux安装它，好了，开始吧。

基础工作，安装toolchain增强工具
-----

按《在tinycolinux上编译seafile》的方法，安装3.x的autotools，包括autogen,automake,autoconf,libtool,libtool-dev,etc..
还要安装tclsh.tcz

如果允许，你也可以把下面的给做了

```
File systems  --->
   Pseudo filesystems  --->
   [*] /proc file system support
That enables the /proc virtual filesystem; read the help file for more on that. Then enable the following:
General setup  --->
   [*] Kernel .config support
   [*]   Enable access to .config through /proc/config.gz

When editing the file by hand, say Y to CONFIG_PROC_FS, CONFIG_IKCONFIG, and CONFIG_IKCONFIG_PROC.
```

重编内核不是必要工作，这是测试lxc需要的。本篇只讲编译。

编译lxc
-----

然后下载lxc-lxc-2.0.11.tar.gz的src,2.0.x是lxc2,选择2是因为它从linux kernel 2.6.32开始，与系统所用kernel接近

1,错误：expected specifier-qualifier-list before sa_family_t
在macro.h中，把所有 include linux/*.h 放include directory的最后
2,错误：ms_shared undeclared here
在conf.c中引用这二个头文件，linux/fs.h and limits.h，也放后面

这里的问题大部分都是因为我们所用的gcc443是c99以内的标准，而lxc源码用了部分c11，所以需要如上的workarounds，实际上编译lxd的时候也会看到好几种相似的情形。需要一一针对处理。

Sudo ./autogen.sh,sudo ./configure,sudo make install

—————

lxd放在接下来一篇讲，因为lxd编译要复杂得多，而且它们二者应该分开，因为lxd作为管理可以不跟lxc一样集成在host中而是一个guest中。有没有。。
至于运用lxc和lxd provision的方式，这些都在网上可以找到。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337413/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





