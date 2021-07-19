使用群晖作mineportalbox（3）：在阿里云上单盘安装群晖skynas
=====

__本文关键字：将群晖安装在阿里云主机上，skynas系统盘和数据盘同盘。mineportal3硬件选型,webpe，提取群晖webpe，使用webpe代替diskbios__

在前面我们讲到省事直接使用白群晖的方式，和提到一些需要hacker的地方，有些时候，正规一成不变的方式和硬件的方式有时的确是难于忍受的。从本篇开始我们探究一些群晖的奇特面，比如将其安装在云主机上的方式。比如阿里云。

阿里云上的群晖官方发布了一个skynas，我们知道群晖是靠卖软件卖硬件的，那么阿里云当硬件的情况下，它只能对镜像收费了，阿里云上的skynas是收费的，且套件支持不完全（比如非常适合当php虚拟主机管理器的webstation都没有。），且正常使用除系统盘外还需要至少一个数据盘.其里面使用的引导技术，其实跟其它群晖是一样的（白群晖硬件引导器是集成到机器ROM上的uboot，黑群则是U盘上的xpenboot之类的东西）。这种利用引导器来安装恢复系统live linuxpe,平时自身却是个dsm引导器（它有二部分，如果/dev/sda5不能引导，它会自动安装install/upgrade，因为集成好多SRS驱动所以通用性很强）的思路，就是前面我们提到的DISKBIOS的效用之一（这实际上也是我们前面追求的类苹果在线恢复系统式的PE），我们将其称为web装机系统，webpe(群晖叫法webassistant)，其一般思路就是，

在引导器下，系统dsm实际上是一个完整的linux发行版升级包。是系统也是数据，引导器会划分第一块硬盘空间，将DSM挂载到根下。这里有一段复杂的启动脚本完成挂载和升级转接（直接以tar为系统镜像进行恢复或全盘增量式升级）。分启动和安装，第一次安装系统和以后升级系统甚至引导都是同一个过程。。它仅需要这个第一块硬盘为系统盘和数据盘安装系统（对的，这里的数据盘说法上严格来说是volume1，白群和黑群的引导器可以独立于这块硬盘外置USB等方式启动，直接在onthefly环境下操作这块盘，而aliyun ecs环境特殊，引导器要事先集成安装在40G系统盘下，我们得不到一个类外置启动U盘之类的环境，引导器不能对这块系统盘直接作分区操作完成DSM安装/升级，所以一切都是事先集成好的，引导器内部的逻辑不同）。

失败的尝试，我曾想通过安装virtio类grub4dos仿真光盘的方式加载黑群引导器/skynas引导器（将其加载到纯内存），企业制造类黑白群晖的装机环境，但是不行，问题有2：
1，发现不了硬盘，黑群的xpenboot能认virtio盘，然而其将阿里云盘读成vda而非sda，引导器根本发现不了磁盘。
2，我将提取的skynas引导器做成ISO，它有认盘然而在操作硬盘分区时出现35错误，同样失败，意料之中，因为winvblk这样的东西只对windows镜像有效，是它们的驱动而非linux认识的。

如何提取skynas引导器：从这里下载它的系统http://update.synology.com/autoupdate/genRSS.php,搜索alidsm，发现DSM_SkyNAS_15254.pat并下载，7z打开，我们需要的二个引导文件是zImage和rd.gz,在ubuntu 14.04 下创建grub2可引导的iso，准备grub2的文件，将这几个文件和boot/grub文件夹放进一个文件夹假设是test，然入这里，将其做成黑群式的iso：grub-mkrescue -o test.iso test
如出现：grub-mkrescue: warning: Your xorriso doesn’t support `–grub2-boot-info’. Some features are disabled. Please use xorriso 1.2.9 or later..
安装apt-get install xorriso

好了，因为重新编译黑群晖引导器和调整其发现硬盘的逻辑目前还没有尝试，下面来探究正确安装skynas的方式：

警告，以下过程为了学习起见，不要用于将其用于其它目的！！

准备测试环境
-----


我们先准备测试环境(注意这个接下来的linuxpe并不是webpe，我们用webpe指syno live bootstraper)，为什么要准备这个测试辅助环境，因为webpe的ssh是进不去的我们不能直接在里面工作，我们采用从《使用virtiope安装iso》《在硬盘上安装tinycolinux as linxpe》中的方式在云主机上装上这个环境。并为这个linuxpe准备openssh和lvm支持，将其打造成实用的linuxpe版本。主要就是采用《为tinycolinux创建应用》的方式，在live tinycolinux的microcore.gz中加入这几个3.x的应用包gcc_libs,tcz,openssl-0.9.8.tcz,openssh.tcz,/ ncurses.tcz,readline.tcz,udev-lib.tcz,libdevmapper.tcz,raid-dm-2.6.33.3-tinycore.tcz,liblvm2.tcz,lvm2.tcz。并把shadow处理好，openssh运行一次配置文件也集成到这个PE中去。

好了，最终启动这个live linxpe，我们可以通过ssh进入并作lvm分区操作，如果将其做成上面的可启动ISO，这样就不再需要《利用virtiope加colinux noimage完成云主机linux的动态无损分区》这样的课题了，而我们的目的要稍微轻量一点，我们不打算利用这个PE创LVM分区，而只是复原启动器能用的磁盘结构：

现在上传2个skynas启动器文件（而并非与grub2一起做成ISO）并加入skynas启动器的启动，进入linuxpe,其启动菜单与tinycolinux livepe类似，加一条进去到/boot/grub/grub.cfg，类似

```
menuentry “skynas bootstraper webpe” –unrestricted {
set root=(hd0,msdos1)
set prefix=(hd0,msdos1)/boot/grub
linux /boot/zImage ro root=/dev/sda5 (硬性指定从sda5启动)
initrd /boot/rd.gz
boot
}
```

启动它，访问云主机ip:5000出来web assitant，我们发现它依然不能对内置第一块硬盘进行格式化，这是因为启动器只认集成了分区布局的情况（如果认到，它将不尝试分区），除非，那真是一块在启动器眼中“干净”的内置硬盘（很明显地，启动器也在这里，所以它不算干净）。

备份这二个文件和boot/grub/grub.cfg到其它区。

准备分区布局
-----

然后，准备分区布局，进入linuxpe,fdisk /dev/vda,先键入u，由柱面计算方式换成sector，删除所有现有结构，新建下列布局（不必要求大小一一对应甚至不用格式化，其实布局对了webpe就能继续）

```
Device Boot Start End Sectors Size Id Type
/dev/sda1 * 2048 34815 32768 16M 83 Linux
/dev/sda2 34816 239615 204800 100M 83 Linux
/dev/sda3 239616 20964824 20725209 9.9G 5 Extended
/dev/sda5 239679 9676862 9437184 4.5G 83 Linux （扩展分区第一个分区号永远都是扩展分区号+1之后的计数得来）
/dev/sda6 9676926 19114109 9437184 4.5G 82 Linux swap / Solaris
```

除了把备份的webpe和启动逻辑应用到第一分区sda1，其它分区甚至不需要格式化，如上所讲，正常安装选择在线更新（你也可以上传那个DSM_SkyNAS_15254.pat），你就会发现安装过程就会继续，大小也不一定一定相同，布局相同就会继续了。完工！

你可以利用lvm把剩余空间用起来。

数据盘的问题
-----


进入dsm，你会发现，webstation没有，套件很有限，而且最最关键的一个问题，数据盘所在的volume1，除非另加一个阿里云盘，在系统盘上，不管你在上面的布局如何新增sda7,etc..，都没有在这里被识别为volume1.

启动过程中可以看到一条：volumemamnager.cpp no target disks to be creTED AS VOLUMES AT Vdsm bootup。这尚不能确定除了内核定制，在脚本层就能改变判断volume1的逻辑，如果能做到，那么就能成功。
skynas的安装逻辑主要在，rd.gz\rd\usr\syno\sbin\installer.sh,upgrade.sh，还有/etc/synogrinst.sh,rc.volume等文件中，里面有一条Create data=no的判断。

也可以参照黑群在同一个系统分区上可创建volume1的方式来修改。只是网上有为kernel改virtio为sda的破解，有重新打包pat的破解。定制xpenboot的源码却找不到。

以后解决。

——————

接下来的文章我们要为这个skynas准备一些重要的套件了



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337807/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



