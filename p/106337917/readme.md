Dsm as deepin mate(3):离线编辑初始镜像，让skynas本地验证启动安装/升级
=====


地球变得越来越危险了，20年前地球大部分地区可以用荒凉来形容，而现在每一寸都被开发，疫情，自然灾害频发，如果不及时止损，估计100年都用不了，人类只需要20年就能从气候和生态上毁灭地球

__本文关键字：啥是真正的黑群，压缩skynas磁盘布局为5G内__

在《dsm as Deepin mate(2)》中，我们讲到了使skynas镜像脱离aliyun ecs真正能运行起来的方法。但是我们用的是倒叙的手法，我们还没到得到安装镜像，这里继续。讲解离线创建和编辑纯净镜像，使之能像普通黑群一样启动，完成安装/升级的方法。

Ps:啥是真正的黑群

————
所谓黑群晖,是由于群晖基于linux gpl要求开源了早期内核和toolchain在sourceforge：http://sourceforge.net/projects/dsgpl/files/，而它的rd.gz,pat升级/安装包是很容易解包打包的https://github.com/andy928/synochecksum。因此。各种黑群技术有机会修改zImage,rd.gz和重新打包定制pat。你可以在这里定制内核插入你的驱动，作各种脚本增强，使群晖适用于更多机型，甚至通用硬盘。
故，其实正真被黑的部分并不是bootloader。正真被黑的应该是部分内核zImage与位于initrd ramdisk中的各种外挂模块。至于系统中的安装包/升级包中的正常具体逻辑，是不需要破解的。虽然如此，大部分黑群版本命名大都是xxxboot之类。
具体技术方面，（1）破解的第一关是rd.gz中的synobios：群晖是通过一个叫做synobios.ko的外挂模块集中处理硬件绑定问题的。一旦破解了它,很容易解除硬件绑定关系。所谓的破解，主要就是改一下这个接口的代码，使他在黑群晖的硬件上也能返回一个合法的机型代码（例如DS3612xs的代码是42）。因此，大家现在所安装的黑群晖，实际上synobios.ko文件已经修改过了，镜像pat也用的是重新打包过的pat，synochecksum-emu1 hda1.tgz rd.gz updater VERSION zImage >checksum.syno这里有一个对pat内文件验证打包的过程。(2)破解的第二关，群晖在sourcefourge的开源内核驱动仅支持官方，本身也是有点错误的，需要一定定制，加入自己的驱动，直接编译后在引导系统的时候会出现很多错误，例如加载synobios时候出现找不到符号的错误、synoacl实现并没有完全开放等等。因此，需要对synology开放的内核代码做一些修改，目前网上能够找到的对内核的修改只有旧内核的(3.2.11等）。加入自己的驱动，等等。（3）还有一些边角的破解或增强，如修改u盘vid和pid,填写正确的U盘 VID和PID可以在DSM界面隐藏U盘，修改sn和mac地址可以用于"洗白”。可见sn破解是无关紧要的，除非你要用上官方的服务如quickconnect,ffmpeg缩略生成等（见前文后面，本地验证版黑群，官方只是没有对这些假sn开刀而已）。
http://xpenology.com/wiki/en/building_xpenology可以说是一篇最详细和权威的资料（编译内核的内容，还包括了synobios的破解和rd.gz的制作，etc..），不过它是是针对4.1的老版本的。不过其用于指导黑群制作的思路是不变的。
视上面二方面，加入了不同驱动，作了不同增强，因此有很多黑群版本，虽然定制synobios是不可避免的，但有的版本，如gnoboot支持使用官方的PAT文件（依然是xpenology的思路），gnoboot编写了一组脚本，巧妙的实现了在rd.gz引导系统后自动替换需要破解的文件，不仅如此，gnoboot还包含了大量的驱动，并且可以很方便的加载和卸载驱动。
—————

问题回到出发点，对于skynas,破解目标是什么?对应上面提到的(1)（2）（3）有哪些地方需破解呢？第一个问题，不经过破解直接运行的结果，就是用你提取的安装镜像换机安装时，不会启动sda5中的系统安装无法继续，而是进入web assit重装界面（即用当初收费镜像机dd出来的机器镜像，换机后也有绑定过期验证不通过退回webassit重装界面），这个绑定在哪？是不是验证？这就是第二个问题。

作初步猜想，可能在grub的那个checksum（zimage,rd.gz,update.pat/hda1.tgz三个都是阿里云绑定的因为其内都有aliyun_xxx相关的脚本文件,不排除是这个原因导致的安装验证不通过。rd.gz/synobios应该不需要破解因为都是阿里云机。那么那个checksum呢？），也有可能在sn绑定（因为skynas是个在线机器,有先sn网络验证后安装的条件，不过群晖并不常用sn验证安装过程,物理机上的黑群boot通过了就通过了）。

无论如何，skynas用的并不是严格普通黑群的那套(不是synobios过了，pat就会过)。而是只有zimage,rd.gz,update.pat三个都过了，才过，所以破解会是（1），（3）我们需要测试验证，下面会讲到。

镜像及镜像离线编辑测试环境
-----

我们先来尝试建立一个纯净镜像，去阿里云开一台系统盘为20g的skynas615-15254(选择可卸载)按量机器作测试环境（为什么不在本地虚拟机测试呢？因为我们还没有精力去考虑因换本地可能带来的其它情况），开机后关机，恢复系统盘初始状态为skynas615-15254选择不重启，建立快照。 —— 这样我们就得到了一个初始skynas61515254的快照。重置系统为ubuntu18，并启动，为只有一个系统盘的系统建立一个20G的数据区，建立时恢复skynas快照并mount。

反转数据区与系统区启动逻辑(为什么要反转呢，因为我们未来要在系统区安装skynas运行测试镜像，启动到数据区上安装的ubunt18里离线编辑，grub1,2可以互通)，在ubt18中初步编辑skynas/vda/boot/grub.cfg建立启动到第二硬盘的菜单，像这样：

timeout由15改为1500,如果这个不改，从vnc重启，如果过快，会默认启动到skynas。
加一个菜单boot ubt
set root=(hd1,msdos1) 这里是vdb中的ubt,从grub1 chainload grub2，反之需要insmod ext2
linux /initrd
initrd /vmlinux
boot

删掉原来快照重新建立数据区的新快照 ——— 这样我们得到一个初步修正了的skynas的快照。同时，系统区也做快照。—— 这样我们得到了一个ubuntu18的快照。

现在，使用阿里云最新的功能卸载系统盘和数据盘，为没有任何盘的系统重新建立一个20G的系统和50G的数据（这里为什么先前要卸载系统盘呢，因为不是新建的盘不可以从快照中恢复，我们也不想使用镜像功能因为那会涉及到计费），建立时恢复新skynas的镜像，如上所述，我们将在这个ubunt18里离线编辑(为什么ubuntu18里面不同时装我们以前文章中处处使用到的tinycorelinuxpevirtio呢因为tcpe毕竟有些工具版本有限不太方便)，在skynas盘运行测试镜像（为什么不在数据盘里运行skynas呢，因为skynas kernel的root必须是sda系列，而ubt怎么都可以），这样我们就得到一个离线编辑运行镜像的测试环境了。

vnc重启，进入skynas的菜单，选择启动到vdb上的ubuntu,在ubunt里，dd( + status=progress可看进度)数据区到系统区root下形成img.gz文件。得到一体化灌装好的一个初始镜像615-15254.gz，这也是阿里云初始sky nas 镜像用的方式，(并非用任意pat通过webassit直装)，mount /vdb5可以看到它是如下这样组织的，像这样：

.SynoUpgradeIndexdb.txz
.SynoUpgradePackager       猜出现在package center的是丢在pat/package升级包中提取出来的
.SynoUpgradeSynohdpackImg.tcz
.NormalShutdown
…..
然后是hda1.tgz释放到整个sda5中的内容

其实不嫌麻烦，你可以利用前几篇文章中的方法或重新建立分区，研究手动释放DSM_SkyNAS_15254.pat升级包到vdb5得到如上组织的方式。


绕过验证逻辑
-----

这样三者(kernel,rd.gz,pat)都有了，synobios略过，来看grub中的checksum，它相当于物理机中pat中的checksum，checksum是check rd.gz+zimage的。如果修改rd.gz，checksum会failed，但它会不会影响rd.gz中的逻辑呢。如果验证逻辑主要在rd.gz中呢？（这个在gui桌面环境下用7z就能打开查看,里面有大量的可能验证逻辑），不论如何，先验证一下修改rd.gz使checksum失效吧，看会有什么影响。

我们首先测试来为rd.gz中的root去掉内置默认密码，以证实我们的想法和利于接下来的分析，:etc/passwd去掉第一对::中的x,/etc/shadow去掉第一对::中的*，如下作重新打包即可：

(我们现在在/root，mount /dev/vdb2 mnt/vdb2, cd mnt/vdb2,vdb2空间足够，可以直接在这里修改和重打包,首先把rd.gz改名为rd.gz.xz,不然接下来xz解压不成功)

```
(解压)
mkdir rd.dir
cd rd.dir
xz -d -c -k ../rd.gz.xz  | cpio -id

(压缩)
cd rd.dir
find . | cpio -o -H newc | xz -9 --format=lzma > ../rd.gz
```

（测试1）
重启测试新的rd.gz,即boot sda2启动菜单中去掉checksum和vendor。此时进去/sda5/etc/会发现大部分rc都是加密的。原来checksum只够影响这里。

（测试2）
重启测试新的rd.gz,即boot sda2启动菜单中保留checksum和vendor。此时进去/sda5/etc/会发现rc都是正常的，只是显示checksum处显示failed，而且linuxrc.syno failed on 4。但此时系统依然无法启动。，启动后。依然如意料只会停留在web assit界面。

看来checksum并不是最终验证或唯一的过程。使得这个升级过程不能继续。我们需要探索更多rd.gz中的地方。接下来就是初步分析整个rd.gz，去得到绕过验证的逻辑，然后动态修改rd.gz，并用新的rd.gz用在系统区中启动。一点一点测试，这是我们的新思路。

我们结合dmesg,从linuxrc.syno来一步一步分析。这里的流程是rd.gz/linuxrc.syno(rd.subr)->/etc/synogrinst.sh(rd.gz/usr/syno/sbin/installer.sh,update.sh)->sda5/usr/syno/etc.default/aliyun_sn_init.sh,sn_generater在第一次启动后（无论成功或失败）会自删掉

我们从上面找到了，从rd.gz到sda5 privot root的地方(rd.gz/linuxrc.syno，rd.gz/etc/rc中的mount /dev/root /)。

貌似在upgrade输出，准备升级开始的地方。
如果找不到/dev/root就找不到/tmp/root等
如果找不到分区，就找不到/dev/sda
找不到/dev/sda，针对sda就全盘cfgen不了，就导致linuxrc.syno启动失败(upgrade过程)
如果upgrade不了，就形成不了整个有效可启动的/dev/sda5

正是这里导致了linuxrc.syno failed on 4，最终就只能进入rd.gz

因此，所有的问题都变成，如何触发 让rd.gz过程找到/dev/root(就会启动/dev/sda,事实上绕过了验证)

（最终成功测试）
解发能找到分区的关键是，安装过程中要有一个/dev/sdb，跟前文一个思路和技术。

(最后来说那个sn)

其实那个序列号只是系统内一个叫sn_generater的本地工具随机生成的。有没有那个sn没关系,系统可以以空sn安装启动
那个aliyun_sn_init.sh和aliyun_sn_generater是临时生成的。


封装和后期离线编辑过程
-----

最后是封装：dd到ubut分区中,及后期离线编辑：

```
（没有kpartx，我们可以用loop)
losetup -f
losetup -o 1048576 /dev/loop0 skynasinit
mount /dev/loop0 /mnt/volume1
losetup -o 17825792 /dev/loop1 skynasinit
mount /dev/loop1 /mnt/volume2
losetup -o 122715648 /dev/loop2 skynasinit
mount /dev/loop2 /mnt/volume5
umount /volume5 losetup -d /dev/loop2
上面-o 之后的数据就是skynas的四个分区开始乘于512得到的

(如果volume5 umount不了,我们可以找出占用volume1的在执行进程并终止它们)
lsof +f -- /volume5
Kill -s 9 pid
```

——————

dsm的web后台技术做得很流畅很轻量不费资源，还有webassit在线安装升级是syno的二个特色（从这里我们可以分析得到其边分区边升级、安装的技术）。结合前文《Dsm as deepin mate(2)：在阿里云上真正实现单盘安装运行skynas》，甚至你还可以压缩skynas磁盘布局为5G内，这样镜像做出来也会更小,集成更多内置spk和一个叫move_syno_pkgs.sh的脚本，etc…..前文遗留问题：有一些spk如vide,photo只能在有单独hd as volume的情况下安装,把sdb1改成sda4



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337917/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



