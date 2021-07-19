阿里云上利用virtiope+colinux实现linux系统盘动态无损多分区
=====

__本文关键字：利用colinx+virtio winpe定制aliyun多分区linux系统盘,在winpe xp winpe中运行colinux,在windows pe下真正操作linux分区，利用colinux作单硬盘分区扩容无损分区， bootice安装grub2-00 到硬盘,云主机越狱装自定义镜像__

在《发表virtiope》《在阿里云上装自定义ISO》和《定制virtio winpe镜像》系列中，我们用类似手机越狱，刷机的hacker workarounds（围绕virtio pe为中心,利用一系列小工具组合）的方式达到了使云主机装机变得像本地PC一样方便流行的维护/管理方式，，那里我们侧重讲解的是windows，任务结果是将自定义ISO装成了系统盘，将系统盘分成二个区并把备份后的镜像装在第二个区内，这样借助virtiope和一系统一数据区的双分区设置可以恢复一个全新的系统。

在《一个设想：共盘文件系统的windows,linux》一文中提到过windows与linux二大OS文件系统存在巨大鸿沟的现状，及我们希望得到一个共盘的uniform windows linux fs的要求。本文中，我们将在装机领域，探索一种“在winpe下自由操作linux分区”的目标与可能。----- 文章最后，探索为单硬盘单分区下的云主机linux分裂为二个分区，打造一个类PC和手机recovery的可恢复rom机制，只要这样，在装机和实用阶段，都能完成某种“共盘，实用的windows,linux融合方案”，那文提到的设想才能基本变得“像那么回事”，也算有技术参考方向。

>在winpe下操作linux分区的难点，在于它不如ntfs受windows中的磁盘工具如diskgen,pqmaigc之类与其结合支持得好，在windows下用此类工具操作EXT3，要么不受支持（需要特定驱动且这类驱动往往很原始），要么能读不能写ext分区，要么能写但是频频蓝屏，更别说动态对其调大小，与类gho方式恢复镜像等（diskgen493开始支持格式化EXT3，也不行，稍后会讲到）。甚至格式化都很久

>关于单分区linux动态扩展出新分区有LVM这样的方案，但是要求在业已分好标识为8e的分区格式的情况下进行。

我们的总目标，还要打造一个windows,linux二合一的pe维护盘(保证一切在该xp based winpe下完全，且不需要二次进不同的ISO环境，比如合盘的windows+linux pe)。这一切我们将在1g内存的阿里云预装了ubuntu14.04 32bit的一台机器上完成。下面开始：

在阿里云上利用noimagecolinux实现linux系统盘的动态分区扩容
-----

这里我们额外用到的virtiope工具有（除了原来封装于virtiope的四个:showdriver,ext234reader,bootice,ramdisk），还有：winpm 7 服务器版本for winpe，它用来分出新ext3区。，还有colinux noimage（busybox我们能用到的工具有mount,tar,cp等等）用来重建系统：众所周知colinux，根据我的《发表colinux》，它被定位于guestos，可是它本身也是工具，colinux可以nomiage的配置形式运行，可加载windows目录为分区也可加载本地硬盘为分区。不加载任何镜像的colinux自带busybox，可以实现在windows下操作linux硬盘分区，实现真正的重新格式化，分区，扩容等效果。最后还需要从网上找一份新grub boot文件包,用来重建grub2.0。

1)准备工作，按照《在阿里云上自定义安装iso》的原系统是linux情况下的方法，将以上几个工具和boot文件包上传放到boot/tools下，然后tar整个根目录

```
cd /
tar -cvpzf backup.tar.gz --exclude=/backup.tar.gz --one-file-system /
```

看到打包后的大小是570m，这个就是原系统镜像。

2)然后，启动进入virtiope，利用ramdisk建立一个590m的内存盘(size=0000250 hex)。利用234extreader将/boot/tools和backup.tar.gz放进来这里的暂存盘是T：(为了操作234extreader你最好要有一个带右键菜单的键盘)，利用winpm删除整个40G分区然后分二个小ext3分区，一个10G用来作新的系统盘，其它30G用作自由空间日后作数据和镜像存放。打开colinux conf文件夹，noimage.conf中设置如下：

```
cobd0="\Device\Harddisk0\Partition1"
cofs0="..\..\..\"   (因为tools与backup.tar.gz并列放在T:中，回退3级才能看到T盘根)
保持mem=128，方便稍后的复制解压，也不能开得过大，因为1G的内存开了用得差不多了
```

现在portable_colinux.bat打开，提示enter激活busybox时，mount 2个盘到noimage colinux：

```
mount /dev/cobd0 /mnt/temp (10g盘)
mount -t cofs 0 /mnt/win (注意cofs与0中间有个空格)
```

(以上2个mnt点是colinux自带的)

3)然后，就是利用busybox中的工具：

```
cp mnt/win/backup.tar.gz mnt/temp/backup.tar.gz
chdir mnt/temp
tar -xvpzf backup.tar.gz -C / --numeric-owner 解压
```

用bootice安装新的mbr grub2.0到硬盘，从网上下载grub的boot文件包替换现有的boot文件夹（除了保留boot下原有的10个内核文件）。

4)，最后重启，进入分区调整后的linux。

如果看到新的grub2启动界面，就说明基本要完成了

```
set root=(hd0,msdos1)
linux /boot/vmlinuz-4.4.0-85-generic ro root=/dev/vda1 （注意阿里云是vda）
initrd /initrd.img
boot
```

进入新的系统，成功！！

打造linux和windows二合一的winpe装机维护方案
-----

一些失败的尝试：

>我曾尝试7zip直接解压或gnu windows tar解压到ext2sd形成的分区中，但都会蓝屏，这就是为什么我开头就说windows下处理linux分区是非原生的。大部分时间它只是辅助用一下。据说比ext2sd,ext2ifs更好的是Paragon_ExtFS之类，但是上传后无法运行，也无心去试了。不过(要是virtiope日后直接集成了ext2sd就不用这步了)这倒是另外一个极好的尝试方向.

>我也曾试过diskgen是4.9.3的（4.9.3的开始支持对ext2/3的分区，它虽然比较大，但是它综合了bootice,234extreader的全部，且鼠标操作好。），跟上面一样它们甚至在xp winpe上无法运行。只有这个winpm 7 服务器版本for winpe很好支持手标操作。

>我曾试过mount  -t tmpfs -o size=590m  tmpfs /mnt/tmp，内部fdisk,直接DD，等等，都不够直观或根本行不通。

其余，就可以不断发挥了吧。比如，前面还有《我们有第二开源操作系统吗》和《一个设想：共盘windows,linux中》，后来我们还提到了《reactos可用版》《colinux as xaas》等等，如果说这篇文章是装机领域的这类使linux,windows天然融合的努力，那么那四篇文章纯粹是是实用领域的融合windows,linux的尝试。

恩恩写够多了，就这样了


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336019/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



