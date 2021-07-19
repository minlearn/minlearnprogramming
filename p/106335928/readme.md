将virtio集成slipstream到windows iso,winpe – 原生方法和利用0pe
=====

__本文关键字：集成srs到windows镜像,slipsrteam driver to windows iso,集成virtio到winpe,winpe集成virtio__

目标：
-----

制作一个winpe，使得其在virtio主机环境中可用，比如阿里云ECS或WEST263，godaddy云主机。

这种技术的本质是将srs驱动带入/植入需要这种驱动的镜像，让其在安装/启动过程中发现硬盘不蓝屏，且辅助其它安装过程继续到完成。这一般有二个步骤，第一步是将驱动手术式的植入iso，实现安装程序的对应这种srs设备的内部驱动支持和发现。

>srs驱动是boot time driver中一类特殊的驱动，因为安装过程往往需要接触到硬盘对其识别，否则会BOSC，且往往是txt mode 的驱动，有别于pnp把inf和driver .sys直接往windows\下一抛就可以的方式，那属于次级驱动，即进入系统后增强系统的后期 设备驱动。在windows安装过程中，这类srs驱动往往是已经存在于注册表和文件系统配置文件中的，所以为了增加一个SRS，我们 必须手动slipstream相应的文件和设置（甚至要修改注册表到.hiv文件）。
 
>如果ISO能自发现带入的驱动 — 比如它是一个安装iso,安装过程有F3选择驱动接口，这种情形比较干净，可以实现从外部将驱动带入到ISO而不需要修改它（当然你也完全可以事先集成它实现自动发现，这样情况跟要谈到的livecd是一样的了），但如果是livecd iso – winpe则是一种windows livecd,则需要集成这个驱动，且保证正确加载驱动后的镜像结果是预期正确能工作的–对应能驱动你那个需要驱动的安装盘）

第二步，实现外部发现光驱镜像过程，即用grub4dos配合winvblk从外部带入光驱镜像。

可见植入srs,与配合grub4dos+winvblk导入光盘镜像是这二大通用过程，这种情况下0pe就是一个强有力的工具。因为它几乎是专门针对这个问题提出的一个整合方案。当然也可书写grldr菜单和手动集成驱动。

注意：目前所提到的全部针对windows iso。因为它识别grub4dos+winvblk带入的驱动。你可以参照其它尚不支持这种方案的ISO带入SRS驱动的方案，比如本站reactos0.4.x增强系列。

下面继续，我们将在winxpsp2下植入virtio驱动到deepinxp iso来描述这个过程,植入到winpe是同样过程，只是pe加载的grldr稍不同：

值入驱动到ISO
-----

手动方法：

修改镜像中的txtsetup.sif（WINPE和windows安装盘中的i386中），加入驱动盘中的对应txtsetup.oem中的chunks到txtsetup.sif，一般有files段，scsi段，scsi.load段，hardwareiddatabases段。注册表的部分好像并不需要（hivexxx.inf->setup.hiv）。把驱动放到system32\drivers下。保存改过的iso.

自动方法，准备素材，利用工具：

* Deepin XP SP3 完美精简版 V6.2 ISO文件，11/20/2013,51.65.104.7400版virtio netkvm和viostor驱动for winxp
* 其它工具：nlite,grub4dos，WinBuilder0.78.exe和定制的vistape脚本，还有一些for linux，在linux下将ext变成windows winpe盘的工具，在wwwroot下
* 准备补丁：对于for deepinxpsp3.iso的补丁：HIVESFT.INF,LAYOUT.INF,SETUPREG.HIV,etc..由于下载来的deepin iso信息不完整,winbuilder处理它时会出现好多信息不全的情况出现，包括一些文件大小写错误，故需要修正。请下载全部工具尝试得出修正差异。

处理方法：

* 解压deepinxpsp3.iso到一个目录，比如我这里是D盘，利用nlite工具，将virtio 驱动 slipstream到镜像中。
* 打开winbuilder，生成winpe。

grub4dos+winvblk引导
-----

准备peboot，文件组织情况参照提供的peboot.rar，注意的是引导文件中的这几条：

```
title (Winvblock) Boot RamPE From ISO -- filename 0pe.iso
find --set-root /boot/imgs/0pe.iso
map --mem /boot/imgs/winvblock.img.gz (fd0)
map --mem /boot/imgs/0pe.iso (0xff)
map --hook
chainloader (0xFF)/I386/SETUPLDR.BIN

title (Winvblock) Boot WindowsSetup From ISO -- the 1st step,filename winxpsp3.iso
map --mem /boot/imgs/winvblock.img.gz (fd1)
map --mem (md)0x6000+800 (fd0)
find --set-root /boot/imgs/winxpsp3.iso
map /boot/imgs/winxpsp3.iso (0xff)
map --hook
dd if=(fd1) of=(fd0) count=1
chainloader (0xff)

title (Winvblock) Boot WindowsSetup From ISO -- the 2st step,filename winxpsp3.iso
map --mem /boot/imgs/winvblock.img.gz (fd1)
map --mem (md)0x6000+800 (fd0)
find --set-root /boot/imgs/winxpsp3.iso
map /boot/imgs/winxpsp3.iso (0xff)
map --hook
chainloader (hd0)+1
```

使用0pe
-----

在0pe中植入srs驱动我们用它的自动选择方案，即在ope\srs\FREQUENT\放一个viostor.sy_，0pe\src\CHKPCI.TXT放一条(具体值即打开viostor\txtsetup.oem查看)

$PCI\VEN_1AF4&DEV_1001&SUBSYS_00021AF4
VIOSTOR
在CHKPCIDB.GZ->PCIDEVS.txt中你也可看到0pe对redhat virtio有支持。

至于引导过程，它在加载驱动后会发现optdesk.wim，然后继续winpe的加载，最终完成进入过程。当然你也可以用下一步菜单实现二步安装windows，或自写更多的菜单实现更多功能（这完全是grldr编辑问题。）



对比virtio winpe，与众不同的是virtio 0pe版本的winpe可以借助netkvm连网。且有更多外置工具可用。

而virtio winpe支持将linux winpe盘变成windows filesystem的winpe盘。

注意事项
-----

使用w2k或winxp内核产生的winpe在进入系统时，鼠标可能会出现不能使用的情况（这好像是虚拟机USB驱动冲突通用情况）。至于netkvm完全不必像viostor那样麻烦完成可以采用pnp的方式把对应inf和sys放到光盘驱动镜像中。

 

生成的virtio winpe：

http://www.shaolonglee.com/owncloud/index.php/s/yeKnfbK67f4MXo8

0pe virtio winpe:

http://www.shaolonglee.com/owncloud/index.php/s/dD8dm8c9FPcAUje



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106335928/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




