一个统一的parallel bootloader efi设想:免PE，同时引导多个系统
=====

__本文关键字：Multi loader vs parallel booter__

一直以来，我们都是使用各种bootloader来处理与不同OS的安装和启动任务的，如简单的单OS单loader（比如ntldr这种）,多OS的multi loader(比如grub这种)，这些loader往往与OS和磁盘模式(mbr?gpt?fat32?ext4?)相对。

刚开始的bootloader作为BIOS到OS的bootstraper，而且仅负责磁盘部分。它们工作在实模式，在《在阿里云上安装黑苹果（2）：虚拟机方案研究和可行性参考》中我们谈到，由于BIOS是16位的，与当前32/64的硬件管理程序不搭，所以出现了新的固件规范EFI。后来在新的EFI规范中，EFI自带磁盘BOOT，还负责其它功能比如还带驱动作为OS为硬件认证。管理CMOS,etc…..

>> PS：其实EFI指的是固件本身，这些东西是硬件的，它们不可更改，而clover这些东西不是EFI，是EFI的软件部分。EFI整个删掉并不会影响主板是不是EFI属性，不会删掉内置在各硬件中的firmware。这些软件部分的EFI可以驱动硬件（它们另有意义，如做硬件检测），但并不是OS驱动层的驱动意义（实际驱动硬件），实际上EFI中的驱动运行在DEX中不运行在CPU中，而且EFI中的驱动跟OS中的驱动没有承接关系，EFI规范只是被设计用来启动和引导期的特殊任务，并不影响OS的地位。。

无论如何，作为复杂的预处理系统。此时的loader是一个关于EFI的全部生态。完成更多的任务。实际上复杂的EFI也带工具(efi shell,gui,etc..)。甚至可以浏览网页……俨然是一个小PE了。

而我们谈到的PE其实更偏向指群晖webassist,winpe，苹果 recovery这些。其实它们跟正常OS一样，也包括完整由内核组成的系统，也是由上述各种loader启动的。

parallel boot设想:同时引导多个系统
-----

那么既然有更复杂的EFI，而且存在可能将其发展得越来越多高级，那么可以在loader中直接发展Preinstall PE，或当recovery(post install环境)吗，不搭配内核和工具不组建一个OS,不走普通PE的路子，单loader本身可以复杂到如此吗？—— 甚至，能在其中集成虚拟机管理系统吗，这样我们就可以parallel boot同时启动多个OS了。那么，还有没有虚拟机和实体通用的这种loader呢。

这些设想从何而来呢，我们知道，一台OS是独占整个机器的。在机器完成引导化之后。这个OS就独占了机器的全部资源，安装在硬盘上的多系统引导实际上只是multi bootloader，而并非parallel bootloader，如果EFI可以从一套机器硬件组合中按配额来划分它们组成2个/或多个子机器表示。那么，这样的parallel bootloader将不难于实现。因为我们可以在每一个子机器表示下安装不同的OS，实现多个系统的同时启动。

而这些做带来的意义是很巨大的，我们知道，虚拟化从来都集成在系统引导之后，exsi等裸金属虚拟化方案，是在HOST系统里搭虚拟机管理软件hypervisor。它是涉及到OS的。一些工具级的虚拟化软件如virtualbox其实也本质上是这么回事。在实机上，我们从来都是单个时刻只运行一个OS。再在这个OS里各种分裂化。不能以硬件本身作虚拟化，去掉HOST。

最基本的意义。上述方案的成功，可以使得在一个PC上安装多个OS，按常规/而非虚拟化的方式，就能同时使它们运行变得可能。—— 而且不需要涉及到集成一个与OS同质化的PE或RECOVERY。使之变成通用计算机的标配EFI。

市面上有几种特殊的接近这种多样化用途的loader
-----

在xhyve中有user space的grub2,在vmlite中有能在实机引导vhd的loader，在《在阿里云上安装黑苹果（2）：虚拟机方案研究和可行性参考》中我们谈到模拟层的OVMF。CLOVER本身也是模拟层的。

这些都可以成为parallel booter inside efi的实现参考。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340282/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




