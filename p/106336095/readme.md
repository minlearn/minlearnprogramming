0pe单文件夹，grub菜单全外置版
=====

1, 0pe与镜像定制启动的绝配
-----

本地SRS驱动需要在镜像启动前后注入到系统的情形往往都很常见，0pe就是一个强有力的工具。因为它几乎是专门针对这个问题提出的一个整合方案。包括集成驱动和用winvblk驱动镜像部分。及注入驱动部分。

本站这方面的例子有《高可用virtio reactos svr版本发布-集成生产环境和开发环境》，和《将virtio集成slipstream到windows-isowinpe-原生方法和利用0pe》

0pe有统一PE版本，比较固化，和0penb版本，适合定制。

2，0pe纯定制版
-----

P大的xpe/2k3pe核心的0pe即使在现在WIN nt6系列的pe核心时代，，也是有极大实用和学习意义的

其实对0pe早有耳闻，也用过，只是本人最近无事，闲来wuyou，发现0pe除了他本尊还有一变体0penb，所以兴趣上来搞了个《0penb纯粹DIY版》，，学名叫《0pe单文件夹，grub菜单全外置版》

>0penb单文件夹，grub菜单全外置版

>基于0PE_NBv1.4.3(2012-06-19)27MB_UDM.7z得到

>0，将一切移到boot文件夹
1，改变grldr内容为加载\BOOT\0PE\GRUB\menu.lst（menu.lst由原menu.0pe改名而来）
2，将所有根目录下的0pe,srs移到boot
3，将所有GRUB相关的菜单外置，做成不藏入形式，并归到一个文件夹，方便了定制位置。
4，保留dos.gz到0pe core,xp文件夹改为xpe，并整个放入imgs
5，还有一些细节的改变。。。。。。

技术细节不多，难度也不大，只是自己需要这个一么绝对DIY的0PE，这样一来，也方便了需要的人。。故共享

—————————————————————–

下载地址：

http://www.shaolonglee.com/owncloud/index.php/s/dD8dm8c9FPcAUje



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336095/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




