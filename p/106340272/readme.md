Boot界的”开源os“ : coreboot，及再谈云OS和本地OS统一装机的融合
=====

__本文关键字:firmware as service, linux as boot，，boot as infrastructure，real cloud os and app，云操作系统选型，实机装机/裸金属通用的云OS。hypervisor/vm通用的云OS。__

Firmware编程是计算机（软件）编程的最初一种形式，也是OS的雏形。没错，它是针对硬件的”软件“编程。这个层面最初形成了我们看到的OS。最明显的层面就是OS中的HAL层面。

历史上。bios是这个规范，如今，UEFI，各种各样复杂的loader也出现了。甚至还有linux as loader，即把linux作为开机出厂程序。用户装的OS作为第二级OS。

那么这种情况合理吗？其实所谓UEFI只是机器引导之后的一个特殊执行体而已，这个执行体在没有进入OS之前，可以是其它任何软件。甚至一个极简的linux rootfs。所以，loader as uefi跟loader as linux没有任何区别-它们共享一个相同的最基础部分:就是那个机器引导，之后同质之前同源。

这样有什么好处吗？我们知道UEFI中有很多驱动。UEFI的作用就在firmware中提出一种新的地址模式和能够使用现代语言的编程方式来处理硬件的改变。可是，这不只越来越复杂的UEFI能办到。在一种现代OS中这样的东西中也能办到，而且天然就是现成的，不用重造轮子。比如linux，它类UEFI本身就是高级语言的地址空间产物，它的宏内核设计使得它的驱动天生就是内含的。其中的很多drivers也不用在UEFI中重写呢。而且它够小。它作为一级主板预置程序可以发挥像WINPE，LINUXPE,syno web assist,osx recovery一样的作用，而且，除此之外，os还能兼能提供现成的工具，如busybox等。这样，linux loader既是recovery，也是firmware.等等，它还有其它作用（稍后就说到）

为什么会有这种需要和改变呢？这种需要的出现是因为cloud computering出现了。它要求对现有的PC，裸金属机器。进行新的一轮upwards streaming抽象。使OS的概念重新分层分级，包含单机，实现，云主机，各种寄宿有OS的地方 —— 形成一种多用的云OS，既然这样，何妨让Cloud Computing也配一个抽象Firmware不就好了么。。

所有会有我们今天谈到的firmware as serice，coreboot就是相关的产品。

Coreboot：抽象firmware as serice,云host os
-----

它的前身是linuxbios，不过后来变成了现在的coreboot,libreboot是它的一个极力去除闭源驱动的开源版本。

>>coreboot performs a little bit of hardware initialization and then executes additional boot logic, called a payload.

>>With this separation of hardware initialization and later boot logic, coreboot can scale from specialized applications run directly from firmware, operating systems in flash, and custom bootloaders to implementations of firmware standards like PCBIOS and EFI without having to carry features not necessary in the target application, reducing the amount of code and flash space required.


记得当时刷chromebook用的就是这个coreboot。它对各种设备，包括云主机都有payloads.

而linux as payload更有趣。

>>Two aspects emphasized by proponents of Linux-as-a-payload are the availability of well-tested, battle-hardened drivers (as compared to firmware project drivers that often reinvent the wheel) and the ability to define boot policy with familiar tools, no matter if those are shell scripts or compiled userland programs written in C, Go or other programming languages.


所以，接上面“稍后就说到”：它不光是advanced firmware, recovery，还可以做虚拟机管理器，这样买来的预置了linuxbios的机器永远不怕坏了。因为可以在其中安装多OS。而且这些OS是linuxbios的二级OS，它们是用户OS。只是一些类似虚拟机guest os的主机（所有的guest都是平等的，而且可以parallel booting）。Splashtop产品就是这样的一种设备。Avatt:all virtual all the time就是基于coreboot，在linux里提供了ovz, kvm虚拟机管理器，可以允许客户安装自己免坏的OS，类似exsi，但是可以作为日常用机而不是服务器。———— 当然，这种firmware是需要刷机的。

而正是这个虚拟机管理器功能，就带来了一个更为巨大的意义：——— 我们一直知道，虚拟机，虚拟OS，虚拟化是云计算的主要概念和手段，虚拟机的运用放在今天实在太重要了，因为它不但是装机/运维问题，开发/devops问题，virtual appliance/appcontainer问题，也是云计算和云开发的基础课题。云主机一般是虚拟机。但实机，自从coreboot它使得本地也可以更方便地云化，从此也可以通过虚拟机装机。而不必仅限于传统的host内使用guestOS的方式和结构。—— 作为一种从最开始处：从类传统PC的BOOT处，对云OS装配firmware，的”OS”，它已经天然是个“实机云HOST OS”了。

unikernel和云guest OS
-----

那么guest os呢？还要有unikernel+guest os的设计。

什么是guest os，就是我们通过云在其中运行应用的那类OS，宠统来讲，我们最常接确到的“云OS”而不是什么hyperior，其实我们之前文章一直在探讨，什么是云OS，在不同的层次上，有很多OS，甚至APP都称自己为云OS。

比如，对于装机和开发，在谈群晖的系列文章中时，我们谈到docker as devops，qnap的qvpc，群晖是一种云OS，对于云APP，lamp是一种云OS(with appstacks)，cloudwall是一种云OS (with db and sync only)，Tumblr 它也是一种云OS，因为它提供了一个聚合功能。

所以我们最后结结实实地总结得过一句:云OS从来没有自己的专用OS，云APP也没有本质上专用的云APP。都是现在的本地OS和本地APP的分布式的叠加。所有这一切，都造成了抽象的过度堆彻。

那么现是在思考解决这样问题的时候了。拿webos来说，Webapp,webappstack虽然是打洞主义，然而它的碎片化特征却是独一无二的（超链本身就是一种碎片化,webapp是page,是applet，非常碎片化，通过超链的表现相连）。现在，OS的分布化和碎片化也要发展起来了。

基本的思路就在这里，基本的解决问题的产品就是unikernel。将OS也像APP一样碎片化。就有了我们即将谈到的云OS（关于云APP。也有碎片化的云APP）。而unikernel。它对传统OS也进行了裁剪和重抽象，它也通过碎片化解决了抽象的过度堆彻。

Unikernels are single address space library operating systems. An application compiled into a unikernel only has the required functionality of the kernel and nothing else. Such a stripped-down kernel makes unikernels extremely lightweight, both in terms of image size and memory footprint, and also can lead to security benefits due to a reduced attack surface. There are many such lightweight unikernel implementations, e.g, LING, IncludeOS, and MirageOS. LING’s website takes 25 MB of memory because it runs on top of the LING unikernel. IncludeOS’s base VM starts at 1MB and a DNS server running on MirageOS compiles into a 449 KB image.

（others : 容器OSCore os, smart os,vOS, etc..）

Unikernel往往去掉了传统OS kernel中的某些hal部分。因为它是hyperior的和hostos的。而且它形成了一个操作系统配一个进程的构架。

而其实，一个操作系统一个进程，内核和应用都在一个地址空间内。一个APP配一个OS，是云计算中理想的Guest OS。这有点像user mode linux和colinux,这样的一套OS+APP就形成一个Virtual Appliance，谈到appliance，这是虚拟化自己的APPMODEL，类似web的web app,native的native app,mobile的mobileapp，virtual appliance最初的形式是vmlite的是appliance，Microsoft vpc的xp融合模式，就是不透出guest os，OS作为守护，仅把融合在host os的app透出来。—— 所以，它是对virtual appliance的增强。

Unikernel也是对容器和容器OS的强化。也是真正的applvl的虚拟化。见《打造一个Applevel虚拟化，内置plan9的rootfs:goblin（1） 》。因为它不是用容器管理器的方案来处理host/guest关系的，而是实在的把OS包含其中。

也是对appmodel的增强，一个APP配一个OS，就没有了server, 它是对servless app的增加。

可以说，配有coreboot的云主机+配有unikernel的guest os，是包含了对云OS装机， devops,applicance 容器。的4 in 1所有路径上的解决方案了。

——


无论如何，以后统一用虚拟机装实机/虚拟机，云主机，开发，作容器，的梦想可以融合了，可以实现了。这是我们前面诸多文章的目标。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340272/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



