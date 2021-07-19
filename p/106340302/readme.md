Linuxbootlinux as UEFI,linux over UEFI
=====

__本文关键字：linux as UEFI，linux over UEFI，qube-os__

在前面《将虚拟机集成在BIOS和EFI层，硬件融合的新起点：虚拟firmware，avatt的编译》及前面一些关于coreboot的文章中，我们提到coreboot是一种开源的boot+firmware，它是从硬件加载到firmware（能够存在于flash的其它非hardware initial必要部分）全包的，coreboot聪明地分开了这二者，将firmware部分视为payload。我们还提到coreboot最初的payload就是linux，除此之外，它还支持UEFI，任何可以在硬件加载完后可执行的东西。

这就是说，coreboot的硬件加载部分依然是需要自实现的。面对一块新主板新平台，开发上，除了软件部分（可以用QEMU调试），硬件上它要求刷机有时甚至要求烧录和焊接技能。—— 这对大部分工作来说都是很麻烦的，写好了也不便于安装。

而UEFI是一种开放的firmware标准（虽然大部分UEFI是闭源的），而且适用于现大部分INTEL平台，为了在平台上开发使用UEFI，照样需要完成硬件加载和firmware衔接工作，和重点的fimware编写工作（DXE），TianoCore是一种开源的UEFI，coreboot的硬件加载部分可以结合tinacore使用。视TianoCore为payload。—— 虽然还是有很多工作，不过，在这种方案下，coreboot的硬件加载可以直接采用该平台开放的部分，然后接入自己的UEFI逻辑，而intel vendor UEFI firmware平台随处可见，是用户可得的。

那么对于我们追求的Linux as firmware，有没有一种更干净的连接起硬件加载部分和firmware的机制呢？并在UEFI上使用 LINUX的技术呢？

这就是linux as UEFI或linux over UEFI。Linuxboot就是这类技术实现的代表

LINUX BOOT
-----


The LinuxBoot project https://github.com/linuxboot/linuxboot (formerly NERF) is a collaboration between Google, Facebook, Horizon Computing Solutions, and Two Sigma that aims to build an open, customizable, and slightly more secure firmware for server machines based on Linux. It supports different runtimes, like the Heads firmware or Google's NERF.

Unlike coreboot, LinuxBoot doesn't attempt to replace the chipset initialization code with opensource. Instead it retains the vendor PEI (Pre-EFI environment) code as well as the signed ACM (authenticated code modules) that Intel provides for establishing the TXT (trusted execution environment). The LinuxBoot firmware replaces the DXE (Driver Execution Environment) portion of UEFI with a few open source wrappers, the Linux Kernel and a flexible initrd based runtime.



1,它首先是一种UEFI实现，跟TianoCore一样，用来代替闭源驱动，相当于coreboot+uefi payload。不过它的工作比coreboot+uefi payload更少。
2,由于它也是全包的解决方案,所以也是一种coreboot代替品，而且它面向使用linux这点与coreboot也不同，coreboot中并没有使用linux当uefi的部分。
3,linuxboot+runtime可以采用cb的硬件加载，也可以使用自己的硬件加载，其linux kernel部分是不变的。
4,它可以实现在硬盘的UEFI分区上直接置换firmware。这样至少对于安装是很方便的。
5,以上其实最终实际上也符合我们在《将虚拟机集成在BIOS和EFI层，硬件融合的新起点：虚拟firmware，avatt的编译》文提到的理论基础，即：linux本身与UEFI技术中的驱动本来就存在重复工作，所以wrapper一下得到的结果也能很好融入UEFI。而linux作为开机预加载环境，可以实现kexec直接加载linux（kernel inside bootloader，不需要专门的bootloader），如果在其中加入虚拟机管理器，可以实现前述我们提到的并启和多OS运行。甚至，虚拟化firmware，可以让黑苹果，黑ios这样的东西变得现实 ——— 这些我们前面都有例子。

Linux boot唯一需要你定制和附加的，就是initrd制作部分，也就是它要接入的runtime，vs Coreboot and Coreboot payload，linuxboot将定制firmware的工作压缩到了仅需要定制rootfs的程度。

Linux boot runtime
-----

Linux boot支持Coreboot runtime（采用其硬件加载部分），linuxboot其前身NERF也是一种runtime，不过像coreboot一样，linuxboot发展到了支持多种其它runtime。如除了coreboot，还有Heads firmware，https://github.com/osresearch/heads

还有这里提到的，https://github.com/u-root/u-root

u-root is an embeddable root file system intended to be placed in a flash device as part of the firmware image, along with a Linux kernel. Unlike most embedded root file systems, which consist of large binaries, u-root only has five: an init program and four Go compiler binaries.


基本上,u-root非常接近我们要得到的goblinux:go,busybox,linux as hyperisor inside boot, linux as user rootfs 。一份及早封装到上游的抽象打包linux for init programmers。

—————

在linuxboot的runtime中集成虚拟机管理器，就得到了我们前面提到的并启虚拟化支持。Linuxboot它主要面向服务器主板作虚拟化。Qubes os就是另外一种主要面向PC主板虚拟化的。
 

-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340302/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



