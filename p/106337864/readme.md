Dsm as deepin mate：将skynas打造成deepin的装机运维mateos​
=====

__本文关键字：ecs在线安装自定义镜像，通用无限制云主机dd安装windows，Debian live os as packer，用群晖统一打造Time Machine和iCloud server__

本文是《ubuntu touch as deepin mate os,second pc os》中用dsm作为deepen mate的继续。

当mac产品之间变得越来越融合(osx10.15,ipad os,etc..)，我们抛弃它的成本就越来越高，对于我，我最不能舍弃的是它的icloud与finder无缝的方式。还有它能timemachine整盘备份/网络恢复,osx recovery在线重装os的方式。

所以我一直在寻找脱离osx生态和替换以上产品的方式，当然，要以不牺牲体验为前提。比如，它可以是一个类unix系统，如日益变好的deepin和一些第三方的cloud sync产品。见《ubuntu touch as deepin mate os,second pc os》。

可deepin刚好缺少这几大功能。它的recovery os是传统windows PE式的不支持网络安装。它又没有全盘镜像。deepin dde也没有集成cloud,要寻找另外的方案。

那么有没有在网络环境中通用的安装iso的手段呢？网络安装系统的一些实践和通用技术是什么呢。netboot?pxe?tftp?ghost network?

那么用户数据重装恢复呢。这基本可想象是现今常见的cloud服务+客户端同步，对于linux的桌面，有没有无需客户端的cloud mounter呢且能与explorer尽可能融合的方案存在呢？甚至，它能与系统重装统一吗？

好了不深究它了。关于系统重装,运维，这绝不是一个小课题，以前我们集中于单点和实机，现在我们有云和分布式节点频繁重装和异地灾备这样的需求,还有更多未来需求 —— 我们将这一切上升到一个OS，这一切我们称为mirror os,mate os。

我们一步一步来。依然是采用skynas和deepin作例子，我们首先要将skynas打造成deepin的装机运维mateos。

网络重装系统，与用户数据恢复，整盘快照增量备份恢复
-----

在前面《共享在阿里云ecs上安装自定义iso的方法》我们讲到在ecs这种异于实机的装机环境中换ISO，那时我们利用virtiope在linux ecs中操作硬盘释放系统安装。大部分都是手动过程。利用的是windows pe这种图形GUI的中间维护系统。

现在我们有linuxpe和自动化的方案，moeclub在《从零开始:在 Linux VPS 上覆盖安装WINDOWS通用教程》中用Debian live os描述了手动利用DD备份和恢复镜像的方式，如以下：

```
dd if=/dev/sda | gzip > vdaimg.gz
wget -qO- http://xxx/vdaimg.gz |gunzip -dc | dd of=/dev/vda
```

除此之外，moeclub还并提出一个整合性的脚本installnet.sh，并附有修正ip,远程桌面等的作用。它可以dd方式恢复windows镜像(你可以想象osx pe的在线安装系统)，也可以以安装方式在当前云主机安装常见的linux（你可以想象为syno pe的webassit局域网安装pat），使用的是debian livecd，它发挥二个作用，1，livecd可以将驱动注射到维护系统，Debian netboot mini.iso https://wiki.debian.org/DebianInstaller/NetbootFirmware https://www.debian.org/devel/debian-installer/这相当于在线生成virtiope的自动版，2，它类packer,可以为即将安装的系统按模板生成目标实体。这二个完成之后，它会全盘DD或安装。

这个Debian liveos+moeclub的脚本即提出了一种通用网络安装/恢复的方法。当然它还没有自动/手动备份和恢复用户数据的方式。系统的更新和系统配置数据，，应用数据，用户数据往整盘的增加，即镜像的多版本，都要求timemachine支持增量，

在osx中，timemachine 也是有局限的，它只备份osx所在区。至于用户数据恢复，其实timemachine在完成一次恢复之后，icloud数据是默认被排除不恢复的。在osx中，系统和用户数据分开备份。是因为apple认为timemachine数据在本地时间胶囊之类的东西上局域网云，icloud永远在远程云上，用混合备份的方式。

而现在我们统一了系统和数据都在远程。趁现在它没有像timemachine那种持续增量DD和恢复的方式，我们可以改进一下，把timemachine发展为cloud share sync的方式。用户数据和系统数据彻底分开。只要求备份某个有数据变动的分区。

比如，从用户名开始的以后部分全部是backup的。彻底分离的usr/system设计,system为readonly，这其实是安卓类手机系统的一惯做法。可以是一个pe.这也是类群晖系统更新的关键所在。updateable的部分不触动webassit部分。Mac osx 10.15之后也是将系统作为只读镜像,tinycorelinux也有cloud模式，所以以后，osx timemachine估计不会备份系统除非单盘方式。

好了。以上是问题的高级部分。回过头来：

下面我们来讲解使用它在阿里云安装skynas的方法。再来讲集成dsm的cloudsync服务到本地的方式。moeclub的InstallNET.sh脚本针对windows镜像的，才有改IP的效果。如果是skynas这种，需要手动去改。或事先在镜像中做好。或通过定制脚本来实现。不过skynas是可以自动配置IP的。

在阿里云上网络安装skynas镜像和搭建类timemachine的服务器
-----

在《使用群晖作mineportalbox（3）：在阿里云上单盘安装群晖skynas》和《利用整块化自启镜像实现黑群在单盘位实机与云主机上的安装启动》中我们讨论了手动抠出skynas镜像并单盘安装的方式。那么现在我们将使用上述InstallNET.sh来自动恢复它。这三篇文章中都有类似linuxPE，脚本这样的组合手段和相似思路。

所以我们这里直接DD出skynas镜像并不需要按上二文处理，你可以按需买一个双20g盘阿里云（这里仅作为测试，数据盘可以买大一些，系统盘和数据盘都不要使用ssd），vda用skynas615-15254初始化在初始化过程中一旦出现已运行，在控制台立马快速强行停止云主机，然后作快照（使用阿里云自己的导入镜像没用它是没有分区表的），恢复这个快照到数据盘vdb，然后重置vda为ubuntu，在其中dd并gzip vdb，将得到的文件scp上传到可以下载的地方，然后利用installnet.sh将其恢复到任何ECS的vda上./installnet.sh -dd 'http://xxx.com/img.gz'，如果有VNC可以看到Starting the partitioner 会停留很久,就是在执行上面二句，一定要等它完成。(其实你照样在这里可以得到系统盘+数据盘单盘安装skynas方式，提示一下：把数据库作为临时系统盘启动，让系统盘中的镜像临时认数据盘为vda，然后卸载数据盘。)。

以上或许还支持将skynas安装到其它服务商的ecs上。

好了。skynas支持TC和cloud station。然后找到一个叫cloudmounter的软件，cloudmounter支持将dsm webdav 方式的cloud托管服务集成到osx的finder，还支持linux, windows，并将缓存的文件放入一个可选的本地位置，当然，它跟提供了finder sync api的osx finder的可定制性还是不能比。不过它跟原生的iCloud in finder最接近了。

———————

下次我们将把群晖打造成一个uniform 运维，使用，开发者统一的os设计。配合浏览器录屏把群晖打造成收藏os，能收集所有类型文档的，聚合os,,像以前的tumblr一样，所有的网上视频等，不必经过下载。收藏视频的方法是录制，像note crapper一样，并将让tinycorelinux支持netboot和网络安装。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337864/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



