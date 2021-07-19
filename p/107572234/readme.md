一个设想，在统一bios/uefi firmware，及内存中的firmware中为pebuilder.sh建立不死booter
=====

__本文关键字：firmware in RAM' replacements for UEFI firmware，虚拟efi,编译类colinux的linuxboot__

在《云主机装黑果实践》上我们反复提到一种在bios上也能运行的uefi，这就是变色龙和四叶草。它在内存中模拟一份虚拟的firmware和efi机器环境，以对接需要efi环境运行的os（如果是bios机，可以模拟一份全新efi，如果是uefi机，可以替换或修改相关uefi项），------这样无论是legacy系统还是新机器是实机还是云环境，都可以制造一层一致的中间层机器固件环境和功能来定义os运行所需环境 .

在一致的中间层firmware中我们能干啥
-----

在这种”中间层固件，软件化的固件“中，除了硬件加载部分，我们可以写入各种payload（即uefi的app,如一份efi shell，如一份linuxboot中的vmlinux和rootfs，实际上，linuxboot即是这种技术，它保留原vendor firmware的硬件加载部分，把uefi app部分替换成linux+linux rootfs，比如headfirmware：一份集有虚拟机管理器的linux或uboot：一份集有userland golang busybox替代的linux）。，------ 然后，在实机上，这样的一份firmware可以被直接刷入bios rom或spi rom，在云主机或虚拟机中，它们可以被刷入分区代替firmware硬件rom区(相当于强化了的booter,with firmware included)，为bios机重新定义firmware，---- 实际上，群晖中，由initrd和updater.pat组成的二层rootfs，它的第一层initrd即是一个meta rootfs，适合放在这样的firmware区中。所以黑群的引导，从以上的观点中，可以理解为整个引导所在区组成的firmware(也可以理解为linuxpe)，而实际上，这一部分，在白群中，的确是被刷入主板的spi中的。

为pebuilder.sh建立不死booter
-----

而我们最想做的，就是利用上述技术为云主机上pebuilder.sh建立一个不死booter，我们知道，现在都流行一键还原和折腾安装系统，老旧的电脑，一折腾错磁盘就费了，又插usb又插光盘的  到现在为止，这种问题依然没有解决，大家还是靠U盘救电脑，其实新的电脑也一样，如果能做到类mac电脑一键网络恢复的功能，它的原理是将网络支持，网络恢复写进了firmware中就好了，在这里提供系统在线恢复所需的环境(这个环境局限受payload大小限制往往只有几M到十几M。)，，如果可以，这一切可以日后可发展到实机上作为linuxpe（不用这个firmware刷机）。因为linux的srs驱动支持很强大。

如果是刷机方式，这种booter是天然刷不死的。除非spi或bios rom坏了。如果是仅放在引导区，只要严格要求镜像市场中的镜像格式审核，也是可以极大程度以免刷死的。

技术原理上，四叶草它们用开源的tiancore和ovmf(qemu)达成。由于变色龙和四叶草实际上就是tiancore模拟内存中的efi加强化了rUEFit引导osx的特化（以引导的形式存在的虚拟firmware，功能面向为引导osx），而linuxboot也面向产出firmware，却跟clover不同的地方在于：（它不面向引导的形式，而面向产出一个能被刷入的rom，linuxboot功能面向于：自动化保留vender中硬件加部分和替换payload的功能），除此之外它们一样。

所以实际上，也可以按照linuxboot的方式在四叶草中里面集成linux和linux rootfs。

由于qemu环境和云主机环境始终是最容易虚拟得到的环境和最方便接确到的实验体，所以。最开始制造for qemu pc3.5,i440fx等的这种不死booter支持最符合现实。

等条件成熟了就动手做一下吧。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107572234/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>








