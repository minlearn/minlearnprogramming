一种设想：利用tinycorelinux+chrome模拟chromeos并集成vscodeonline
=====

__本文关键字：Chromium as linux desktop,x11 – 在没有GUI的情况下在服务器上启动GUI浏览器,kiosk mode,Porteus Kiosk,x11 kiosk mode，chrome --enable-consumer-kiosk,webkitgtk kiosk，发明自己的chromeos，直接用vscodeonline当textcui,vscode as text os cui__

在《cloudwall：一种真正的mixed nativeapp与webapp的统一appstack》中我们讲到，web是一种从native和nativedev打洞出来的appmodel（它的协议是应用级的http，html是云UI,是一个可以没有一个宿主render的云GUI。其后端可以是lnmp中的nmp都是非平台依赖的APP级的引擎,这是一种"GUI/网络/存储/业务"的四栈云化的结构，在这些层次上它已经脱离了本地开发），webapp后端也可是k8s这种，在《利用openfaas faasd在你的云主机上部署function serverless面板》中，我们介绍过serverless,其实，云和云开发，本来就是serverless和terminal-less的。因为云的属性，强调的就是没有PC，没有一个专门平台，承认docker它是服务性APP中去掉server backend的自然平坦app结构:server"less" virtual cloud appliance，在《一门独立门户却又好好专注于解决过程式和纯粹app的语言，一种类C的新规范》我们讲到golang的极小运行时，和分布式开发中,"平台/语言/设计/人"四栈变三栈的正常现象，没有nativedev和平台依赖，云APP可以是三栈结构。即，云APP有三栈就OK。这些文章提出了对webapp本质的疑惑又一一进行了分析。----- 所以，总体上，web这种app早已经是云APP合理的存在。人们应该接受了这样的离散平台，和新型APP和开发。

> 再说一次，为什么说云开发无OS呢，因为os，已部基础和服务化了。app不直接寄宿在os或虚拟机上。开发变成了服务调用。或者说，这些开发来源不属于你自己的机器，因此你不能控制该服务所在OS（无须传统部署）。

在《cloudwall：一种真正的mixed nativeapp与webapp的统一appstack》中我们还提到chromeos作为webos(终端)的合理性，如果说一种app决定一种os，既然web是一种云UI，那么云OS，如果它基于PC上的OS实现而来，chromeos这样的实践就变得合理了(《minlearnpgramming》整书toc vol1已经重构为云os融合/云app融合二部分)。对于webos ,在《群晖+DOCKER，一个更好的DEVOPS+WEBOS云平台及综合云OS选型》我们还谈到docker based webapp/webos(而其实我们还谈到docker based webos被一种unikernel os代替)，无论如何，现在我们尝试在tinycorelinux上以kiosk mode安装chrome，基于以前也写过的一篇《在tinycorelinux上装chrome》文章，使kiosk mode的chrome成为系统启动时进入的唯一全屏独占应用，打造类似chromeos的实现，未来，我们直接用vscodeonline来代为该chromeos唯一的页面。使这样的linux发行成为vscodeos。

kiosk mode实现chromeos
-----

这个模式的本质原理只需要在startx的末尾启动一个full screen的webkitgtk demo或基于chrome的demo,不需要装桌面管理器(因为它不启动桌面环境)。其实在tinycorelinux界有一个实现了，如webDesktop-v.0.2.iso，https://github.com/joaquimorg/早就不维护了。它基于webkitgtk，自己编译了一个叫desktop的全屏demo browser在/usr/local/bin，然后安装了x11，最后在startx中写入启动：desktop $PHOTOFRAME &。

而chromium（开源版chrome）是自带kioskmode的，因此省去了将它做成全屏独占的模式的工作，可以直接使用chromium –kiosk –incognito http://localhost的命令形式开启全屏模式并导向本地主页（为了安全起见也要禁用xorg设置中的tcp，如果你已有桌面管理器，要设置禁止禁用屏幕保护和自动重启）。你当然也可以使用cfe这样的lib自己编译定制demo，我们采用的测试版本是tc11.x 64bit，仓库中已有chromium-browser.tcz和x11支持。可以直接测试。可直接用ezremastered测试，这样方便。

下面介绍我的一个初步成功尝试：

```
利用ezremaster给tinycorelinux core增加桌面生成新iso

ezremaster.cfg部分：

cd_location = /home/tc/CorePure64-11.1.iso
temp_dir = /tmp/ezremaster
对于窗口不能全屏，中文不能显的处理bootcode,注意它与接下来chrome的lang参数中的连符作辨别
cc = lang=zh_CN vga=791
app_outside_initrd_onboot = chromium-browser.tcz
app_extract_initrd = openssh.tcz
chrome只能用xorg 作x11server,flwm这些用的都是xfdev不是xorg
app_extract_initrd = Xorg-7.7.tcz
extract_tcz_script = ignore

手动的post调整部分：
cd extract
为tc passwd一个密码，然后sudo cp -f /etc/passwd etc/passwd /etc/shadow etc/shadow
sudo touch var/lib/sshd,sudo cp usr/local/etc/ssh/sshd_config.ori usr/local/etc/ssh/sshd_config
顺便opt/bootlocal.sh，加入/usr/local/etc/init.d/openssh start
sudo touch etc/sysconfig/Xserver,写入Xorg，保存

接下来是重要关键部分：
tce-load -w getlocale graphics-4.5.3-tinycore64
仅安装xorg-7.7并startx，会failed in waitforx,waitforx在xlib.tcz中,尝试网上说的：1，注释/etc/sktl/.xsession中的waitforx行，2，isolinux bootcode在corepure64 后append喂vga=791或max_loop=256 iso=UUID=$rootuuid$isofile or after system starts,sudo fromISOfile /mnt/sdb1,startx,make TC boot and be used directly from an ISO file)，都不是解决问题的关键。/usr/local/bin/Xorg发现错误/var/log/xorg.0.log,no screens find，装lspci发现这是一张virtio gpu显卡。于是tce-load -iw graphics-5.4.3-tinycore64.tcz ，failed in startx消失，壁纸一闪成功进入黑屏桌面（因为没有桌面管理器）。
getlocale.sh选择四个zh_CN，生成mylocale.tcz(看来，tinycorelinux除了主框架部分Core-scripts，tce也能成为藏脚本的地方)再安装一个字体wget http://mirrors.163.com/tinycorelinux/4.x/x86/tcz/fireflysung.tczg到/tmp/，字体x86,64，跨版本能通用
这2个tcz和其依赖不能像openssh,xorg-7.7一样成功集成到initrd，原因不明，只好手动释放，字体只能手动：
sudo unsquashfs -f -d /tmp/tce/optional/glibc_gconv.tcz /tmp/ezremaster/extract
sudo unsquashfs -f -d /tmp/tce/optional/mylocale.tcz /tmp/ezremaster/extract
sudo unsquashfs -f -d /tmp/tce/optional/i2c-4.5.3-tinycore64.tcz /tmp/ezremaster/extract
sudo unsquashfs -f -d /tmp/tce/optional/graphics-4.5.3-tinycore64.tcz /tmp/ezremaster/extract
sudo unsquashfs -f -d /tmp/fireflysung.tcz /tmp/ezremaster/extract

最后，/etc/skel/.X.d，ls -alh,新建一个任意名字脚本（linux中命令行启动与x启动后的运行命令是二个不同的放置过程，命令行下显中文和图形界面中文能不能起作用是另一回事，这个注意），放置启动/usr/local/bin/chromium-browser --start-maximized --kiosk --lang=zh-CN https://www.baidu.com/的逻辑，保存退出。这样会在startx时，在home/tc/.x.d生成实际脚本，，ctl+alt+f1可退出桌面(黑屏)进入命令行界面,ctlc退出chromium再次ctlaltf1返回正常命令行
整包导出后为160多m
```

集成vscodeonline实现cloud dever os和writer's os
------

实现了kiosk mode还不够，我们的webos，要本地和跨网络都可用。打造一个沉浸式不离开IDE的全能环境和一个vscodeonline based cloud dever os，。我们直接用vscodeonline当这种os的textcui,vscode as text os cui直接把vscodeonline作为shell。,因为vscodeonline本身有一个命令行区web cmd，可以当复合shell使用。而虽然vscodeonline比较笨重，但是它的主要界面实际上是一个复合了的编辑器，可当真正的写作环境如写md文章:vscode没有工程组织，包，用磁盘文件代替工程文件类go用环境变量+文件夹当工程组织，这种方式很适合组织文档。。网络上，当代替备忘录这样的频繁打开/关闭的碎片化写作环境当然不行，但vscodeonline有一个remote ssh，断了也能很好连上，连上能持续连接很久，这放在一台开发终端上可用性还是很高的。

对于webos的shell选型有ssh cui/tui，也有桌面环境有rdp/vnc，这二种是最基本的，本地则有局域网投屏，wifi display这样的方案，跨网络有网页化，也有专门的ctx，remoteapp，虚拟桌面这样的专门方案。vscode online本身就有在远端开端口进行网页服务可用，而设置一个本地终端版的chrome devos，填补了本地也可能需要一个vscodeonlineos的情况。比如可以在利用remotessh的情况下，维持一个远程这种cloud dever os和一个本地终端版的chrome devos，设置一个同步插件同步二者的home目录，这个home区就是网盘。这种实用性还是蛮高的。

----

未来我们要在这个os中集成panel.sh，在这个panel上去掉pai，为terralang增加c header files装上terralang，为devplane增加后绑定/变更域名。vscodeonline  as ide andplugin server+terralang server core+openfaas serverless core,使之成为devpanel.sh。

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/109491497/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



