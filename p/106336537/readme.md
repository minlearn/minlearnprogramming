能装机，能在无光驱的实机稳定启动的reactos版本
=====

__本文关键字: reactos 实机安装,reactos 实机测试__

reactos 0.4.x发布并提供了livecd和bootcd二个版本，，可是都会出现不能在无光驱，实机启动/安装的问题（其实即使有光驱也会出现跟启动介质有关问题导致不能启动），由于livecd和bootcd中的setupldr.sys都支持从grldr 本机LLB bootstrap，所以对于无光驱启动，网上的解决方法一般是在linux下通过grub2来map 或map –mem到仿真光驱进行的（在windows下可结合winvblk这样的东西将光驱带入启动期）。

ps:grub支持file-backed仿真盘，和加了mem到内存的ramdisk盘，测试可用hd32之后的符号代表CDrom。如果不以mem方式加载cd，那么要求cd一定要连续。基本过程是：map –mem (hdr0)xx (hdv0)后，root(hdv0,0)显示虚拟后的C盘信息，再chainloader上面的启动文件（如果这个盘是直接可启动的，就直接chainloader整个盘符,比如(hdv0,0),,,这里r0代表虚拟前的镜像文件所在第一个盘符，v0代表虚拟后的第一个）。

且winvblk模拟的盘+grubdos模拟的光驱，能仿真到真实光驱的同时把它带给winpe。即把firadisk.ima或winvblk用map –mem (pd)firadisk.ima (fd0)这一句加载到虚拟软驱后，其实质是在PE中就可以看到用grub4dos创建的所有虚拟磁盘,我们可理解为这个friadisk给了PE系统一个启动时的注入的一个驱动，它将虚拟盘在启动时全部列出来，所以它与PE内启动机制中加载boot driver过程是绑定的。
可是，即使这样也并不意味着问题已解决，上面的方案看似能工作实际却不行。本文正是探讨一种能让能这二个iso在实机稳定启动，达到正常测试和安装的目的。

注意：以下涉及到的讲解和测试，以下探讨和测试在windows下完成，且均要求freeldr.ini所在的盘是FAT/FAT32.

livecd
-----

livecd通过自带的菜单（grldr直接chainloader freeldr），或通过(grldr map –mem)，或grldr结合winvblk通过它带入，都会出现如问题：

这是目前的一个bug，见图：


——–

IopCreateArcNames failed，大意是指不能分配一个盘符。当然这只是表象，那么导致问题发生的本质是什么呢？我们当然无时间从源码去归结原因，大致几种原因猜测如下：

启动介质光盘冲突？不能发现光驱？这应该是livecd不认识winvblk驱动？

网上有禁用floppy.sys，禁用/拔掉usb，置换uniata.sys，但都不行。

——-

可是所幸freeldr.sys原生支持ramdisk cd，它能在通过freeldr.sys启动时通过option将iso exportascd，加载到ramdisk到固定X：的方式。类似winpe。我测试了下，外置freeldr.sys+freeloadr.ini到fat32根，里面用ramdisk /exportascd选项，全部成功。

ps:原理是因为ramdisk能文件解压到了X盘。然后直接从x盘展开的文件系统中通过setupldr启动。ramdisk是与OS绑定的，用的是Os的loader+启动配置文件中的option选项，看起来与grldr模拟的ramdisk效果相同但实质不同，（能模拟ramdisk效果但不同比如不是X盘）.

bootcd
-----

对于bootcd，通过实机或grub4dos仿真盘，bootcd会黑屏。

那么通过1类ramroslivecd的ramdisk方式行不行呢，不行，通过freeldr（指定freeldr.ini中参数到ramdisk(0)，注意freeldr.ini也支持systempath,kernel和hal参数），加载bootdriver过程黑屏，通过grldr+winvblk通常也不行。

ps:live系统和bootcd系统对于winvblk仿真盘的需求也不一样，这也是本文分开说这二块的原因，但其原理都是一样，就是类似0pe的那一套，可在kernel和boot driver和hal后，启动一个系统的子集。但是livecd明显将所有文件都在第一次启动时展开，bootcd就展开最小的，然后其它的靠后期安装到硬盘比里CD里面那个大cab文件。这导致二者处理方式不一样。

网上解决方法一般是在硬盘上开辟二个fat32,一个放安装镜像（即bootcd iso中提取出来的全部文件和另设的一个那个setupldr），一个放安装到的fat32 目标reactos系统分区。另外一个是生成img，再改造llb+loader，需要重新从外部做硬盘的安装镜像。这二种方法都有缺点，也破坏了源码生成iso的事实,特别后一种方法。依然是制造虚拟硬盘镜像，还要改造二段式boot过程。还是不够方便。

我还测试了下其它方案，比如利用grub+map –mem .img虚拟硬盘文件加载到hd31，但都有各种问题（路径是对的但不能加载hive，etc..）。直接map (w/o –mem)的方式没尝试，因为要求镜像连续。

最后尝试用raw filesystem的方式成功。即直接将整个rosbootcd解压，外置freeldr.ini，在配置文件boottype=reactossetup下面加一条systempath=安装文件路径，即可。

安装结束后不要选择安装bootloader(skip it)，用混合freeldr.ini菜单运行ramlivecd或安装，及启动安装到硬盘后的系统。

下载地址（站内下载）：

http://www.shaolonglee.com/owncloud/index.php/s/VntoC8nlCNGjYY9/authenticate




-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336537/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



