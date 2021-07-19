一键pebuilder，实现云主机在线装dsm61715284
=====

虽然群晖上云没有什么很大意义，因为云主机附属的带宽和存储成本现在还没有做到类似onedrive不限速，几百元1T一年的程度。类似cos,oss这种有api的”类od网盘“成本也偏高实用性也不大，群晖这种OS也没有完全做到以网络硬盘为后端存储，它倒是支持一种分布式硬盘的iSCSI只不过这种分布式是大架构用的海量存储方案。在应用方面，群晖大量采用开源技术，也有一些自研的应用比如xxstation组合，但是cloudstation的sync算法经常会导致冲突，但是anyway，群晖的webstation是个很好的php虚拟主机管理面板，也可以作为网盘间内容转存器，因为它有cloudsync(不过百度盘最近又限制了cloudsync的配额，其一惯作风~)。所以还是来折腾一下：

技术背景
-----

1,aiminick在nasyun上发布http://www.nasyun.com/thread-31988-1-1.html的《DS3615xs直升DSM6.1.4U2【KVM平台虚拟机】加强版引导镜像》: 作者结合利用了jun大引导和老骥伏枥二合一引导（这二个是平等作用的东西，前者有外挂驱动完成了黑群的主要部分，后者主要是多合一，可以用jun大引导中的三个文件extra.lzma、 rd.gz 、 zImage替换老骥伏枥/boot/grub/DS3615xs 下同名文件，主要是extra.lzma，这个文件以沙盒补丁的形式对黑群文件系统进行patching），重新编译了syno内核和virtio modules替换了相关文件,在本地kvm上完成测试了最新版黑群614系列pat的安装，(除此之外他还更新了一篇3615xs  6.1.7的kvm安装，除更新了版本支持值得一提的是这个版本去掉了老骥伏枥引导中的grub1,tinycorelinuxpe等，仅保留了grub2部分)。

这证明kvm黑群（类似《在阿里云上单盘安装群晖skynas》和《dsm as deepin mate》系列针对的skynas。但纯粹的kvm黑群方案相对更原生。）也是可以做出来的（虽然接下来我们会谈到，使用aiminick的包在kvm上折腾还是有人碰到很多坑），但是aiminick并没有解决sda->vda的问题，使用的virtio ide方案。云主机所属的kvm使用virtio blk或scsi。这个问题也在第一篇黑群文章中就提到了。

2,作者tiandishi在nasyun上发布http://www.nasyun.com/thread-70722-1-1.html的《群晖上云服务器,完成度90%！！》，搞定了上述问题，并进一步修复了阵列问题，得出了最新版单盘安装黑群的方法并介绍在了另外一个贴子里。

tiandishi没有直接采用aiminick的包在kvm上折腾，他仅采用aiminick的选型和编译方案（615机型和linux-3.10.x.txz内核），倾向使用原始的老骥伏枥DS3615xs黑群晖 6.1-15152版中的文件制作启动，却使用了jun大ds3615xs 6.1.7引导来做，仅最小替换virtioblk来测试，他重新编译了能识别vda为sda的dsm6.1内核和驱动模块，他修改了 linux-3.10.x.txz 中源码的drivers/block/virtio_blk.c,修改 virtblk_name_format("vd", index, vblk->disk->disk_name, DISK_NAME_LEN); 这一行的vd为sd。先从virtio ide+虚拟机下（proxmox kvm下）开始，按正常IDE盘格式和折腾aiminick的3617的包在kvm上安装617pat并进该硬盘的系统，不断打包进新引导包替换测试，如果失败则回退到原处再次测试，最终发现新的引导可以产生与aiminick测试类似的结果：安装  3615xs, 6.17,15284  版本的系统至完结.虽然至少此时正确能识别硬盘为sda，但这也是新问题的开始，因为他导致了不能重启后正常使用（值得一提的是在tiandishi的包中，他保留了其中tinycorelinuxpe，因为它日后可以用来修改grub中的mac）：

1）现象：启动时发现无法识别群晖的/dev/md0 /dev/md1 软raid系统,导致认定系统未安装,重新进入安装界面.作者分析
2）原因：发现这是因为单盘系统盘是一个软raid1的软阵列，黑裙引导的时候需要识别这个阵列并加载阵列的系统。但当硬盘类型为virtio_blk的时候，内核启动检测阵列使用的 mdadm --auto-detect 这个命令不会映射virtio_blk到阵列专用的设备符/dev/md0,进一步导致内核执行/usr/syno/bin/synocheckpartition 的时候返回结果为Partition Version=0，报错Partition layout is not DiskStation style.;需要替换为 mdadm --assemble --scan  ,这样可以正常识别阵列分区, 使得/usr/syno/bin/synocheckpartition 返回正常结果  Partition Version=8. 临时解决。
3）方法：去掉virtio盘的虚拟化blk格式，按正常IDE盘格式和折腾aiminick的3617的包在kvm上安装617pat并进该硬盘的系统，仅为得到一次磁盘信息，正常安装下头二个区一个2G一个2.4区为软raid区，这相当于在系统内mdadm --detail --scan > mdadm.conf，获得的关于镜像盘的如下md/blkid相关信息，

```
ARRAY /dev/md0 metadata=0.90 UUID=57d437b5:798159fc:3017a5a8:c86610be   你需要手动修改uuid为你的盘的uuidARRAY /dev/md1 metadata=0.90 UUID=7580a324:5f709df7:dcd287d5:29e3aff2ARRAY /dev/md2 metadata=1.2 name=yunnas:2 UUID=b96e00a8:dab9a2b6:c99e9210:f50c9dbf (这个盘是数据盘请略过)
```

这个mdadm.conf将被放进extra.lzma/juno/etc/中作为解决新系统不能重启运行的补丁,手动喂给booter调用：修改linuxrc.syno的补丁文件jun.patch ，在内核挂载阵列设备 /dev/md0 前执行 mdadm --assemble --scan，该命令优先读取上面的配置文件，识别出硬盘阵列，然后后面的挂载系统能顺利执行。

```
mdadm -S /dev/md0 && mdadm --assemble --scan echo "Mounting ${RootDevice} ${Mnt}" 放这句上面
```

打包extra.lzma ，扔进引导，重新启用virtio盘的虚拟化blk格式。测试成功进入系统！还有二个后期小问题：1，启动后其它数据盘部分无法识别，需要添加一个临时数据盘组raid（这个可结合我《Dsm as deepin mate(2)：在阿里云上真正实现单盘安装运行skynas》处理tmp/space.xml的方法尝试解决），2，启动后添加的网卡拿不到IP，需要修改grub引导中mac的地址。但都非关键问题了

下面直接制作镜像，用tiandishi的路子,找对版本,先搞定识别sda安装完系统，再搞定阵列修复进入系统：

制作镜像
-----

准备工作：我们直接使用jun大ds3615xs 6.1.7引导(注意这混乱的版本命名，3615是硬件平台，617是内核，15284等是pat)和原始的老骥伏枥DS3615xs黑群晖 6.1-15152版中的文件（用jun大617替换里面的原615引导的相关文件，/boot/grub/ds6135xs就不改成ds6137xs了以避免需另改grub.cfg）,和tiandishi弄好的http://down.nasyun.com/forum/202005/17/064451tfwjwlvwawylwwgj.zip?_upd=kvmdrive.zip,q外面的那个virt_blk.ko仅仅适用于3.10.102内核编译的系统。前者是引导，后者是补丁，

制造下面的617kvmboot：


```
qemu-img create -f raw 617kvmboot 32M

fdisk 615kvmboot自动dos mbr，n自动一区，a自动激活
losetup -fP --show 617kvmboot
mkfs.vfat  /dev/loop0p1
mkdir bootmnt
mount /dev/loop0p1 bootmnt/
grub-install --boot-directory=bootmnt/boot /dev/loop0

...(复制boot所需文件进来，不要把各种tcpe,legacy grub复进来，，先kvmdrive.zip中重命名extra.lzma中的etc/madam.conf(deepin下可以直接编辑名字自动gui打包)，以防其它非意料因素出现，用extra.lzma直接替换617kvmboot中的extra.lzma即可，另外修改grub.cfg中virtio mac地址一定要525400开头否则一会识别不到。有调试需求，则可以在grub.cfg中将虚拟串口设置为开机不自动启用。)..

umount bootmnt
losetup -d /dev/loop0
```

替换后的镜像也都是mbr虚拟机/云主机适用的，依然是我们前面的在pd中装deepin，开kvm，再启动qemu制造主镜像，二步走，第一步用qemu-system-x86_64 制造镜像，第二步用qemu-system-x86_64 测试启动镜像，如果不分二步，直接上来virtioblk，会提示格式化失败

第一步:

```
qemu-img create -f raw dsm61715284 20G

qemu-system-x86_64 -enable-kvm \ -machine pc-i440fx-2.8 \ -cpu Penryn,kvm=off,vendor=GenuineIntel \ -m 2120 \ -device cirrus-vga,bus=pci.0,addr=0x2 \ -usb -device usb-kbd -device usb-mouse \
 -hdc ./617kvmboot \
 -hda ./dsm61715284 \  -device virtio-net-pci,bus=pci.0,addr=0x3,mac='52:54:00:c9:18:27',netdev=MacNET \ -netdev bridge,id=MacNET,br=virbr0,"helper=/usr/lib/qemu/qemu-bridge-helper" \ -boot order=dc,menu=on
```

这一步会出现网上很多人拿DS3617xs_DSM6.1_Broadwell.img在kvm下折腾都出过一些问题，新的booter也会有，如下：

引导进入dsm，安装时局域网扫不出ip，除了可能开头就提到的你grub.cfg中virtio mac地址没有改成525400开头否则识别不到,如果dsm已识别网卡，只是扫不出而已。也可以这么做：从宿主机看guest分到的ip(arp -a)，或者找个脚本扫virbr0所在的网段了（192.168.122.2到254）。我这里得出是192.168.112.32或78，以后几乎固定都是这个不用再扫，如果还是得不出ip，重启宿主deepin再来(我发现aiminick的网卡识别比较好和效果稳定)，deepin中用chrome打开，在webassit上显示对应的ip也不是上面指定的mac，被改变了，如果连不上重启虚拟机重来，连上了webassit，下载pat,准备手动安装的https://cndl.synology.cn/download/DSM/release/6.1.7/15284/DSM_DS3617xs_15284.pat。安装的时候提示格式化 1 2 硬盘，勾选安装就可以了，引导盘不会被格式化，如果pat版本没匹配，会出现各种安装到57%左右文件毁损，

如果没有问题，系统安装完成，进去命令行mdadm --detail --scan > mdadm.conf,检查输出(本来设计对镜像作blkid和tune2fs -U写入uuid，但那个2.4g的是个swap fs，无法这样完成，故还是修改mdadm.conf)，生成的文件重新整合到xtra.lzma，则可以继续第二步，启动测试。


第二步:

```
qemu-system-x86_64 -enable-kvm \ -machine pc-i440fx-2.8 \ -cpu Penryn,kvm=off,vendor=GenuineIntel \ -m 2120 \ -device cirrus-vga,bus=pci.0,addr=0x2 \ -usb -device usb-kbd -device usb-mouse \
 -hdc ./617kvmboot \
 -device virtio-blk-pci,bus=pci.0,addr=0x5,drive=MacHDD \ -drive id=MacHDD,if=none,cache=writeback,format=raw,file=./dsm61715284 \  -device virtio-net-pci,bus=pci.0,addr=0x3,mac='52:54:00:c9:18:27',netdev=MacNET \ -netdev bridge,id=MacNET,br=virbr0,"helper=/usr/lib/qemu/qemu-bridge-helper" \ -boot order=dc,menu=on
```

最后，第三个分区20G-4.5G开起来。且开第四个分区用于在主镜像中制造硬盘引导为日后脱离617kvmboot也能引导，即并losetup挂载将617kvmboot中的内容以及引导再次离线写入到生成的镜像的第四分区，tar cvpzf它，并将最终镜像上传到onedrive互联。

完工！

利用pebuilder在云主机上安装
-----

由于pebuilder.sh足够智能就不用保留tcpe对于mac的替换，况且这需要在具体目标系统中完成。比如放在pebuilder完成dd后要重启的位置，你可以直接将处理mac的逻辑放这里：(grub.cfg中可以写function，为什么在那里不做？一定要在进入系统后？)，以至于这里也可能可以跟处理mac一样放处理tune2fs的逻辑：

```
d-i partman/early_command string $PIPECMDSTR; \
sed -i 's/mac1=52540066761B/mac1=yourmacaddr/g' \$(list-devices disk |head -n1)4/boot/grub/grub.cfg \
sbin/reboot
```

----------

要看pebuilder.sh gif演示的。去我的github，下文，用开源包替换cloudstation和用videosstation云转码


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107559419/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



