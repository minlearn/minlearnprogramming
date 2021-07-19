DISKBIOS：一个统一的混合OS容器和应用容器实现的方案设想（2）
=====

__本文关键字：将ovz用于应用级容器设想和dbcolinux fs用于os template设想,boot into chroot at system startup,将initrd做成自带livefs,ovz as chroot管理系统，livefs as metafs template to make linux an container os，为一个app配一个OS__

在《DISKBIOS设想1：一个统一的实机装机和云主机装机的虚拟机管理器方案设想》中我们讲到利用ovz维护单机和云服务器环境统一其装机方案的设想，主要设想就是我们简单地制造出一个装好了带ovz的tinycolinux as pe环境（与硬盘上另一套tinycolinux共存），这样对于单机，我们可以用这个ovz作为pelinux维护后者，通过web方式重装/恢复后者的操作系统，对于服务器，我们可以在资源允许范围内虚拟出多个这样的硬盘系统并用于运营，同样以web/单机的形式管理后者，注意这里pelinux和硬盘linux，虚拟linux都设想为用的同一份tinycolinux rootfs。

想象是美好的，但我们并没有触及到如何使用ovz来实现《设想1》里的东西，在《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制》1,2,3系列文章中我们讲到在tinycolinux上编译ovz和定制/system /usr分离式rootfs的过程在《发布dbcolinux上的cozylight》一文中我们把它称为dbcolinux，也并没有串联起ovz和tinycolinux rootfs。

那么这篇就是增强这些设想的详细内容且再进一步的过程了，来深入讨论一下，那么，将ovz tinycolinux kernel与dbcolinux rootfs结合起来，这样有什么好处呢，最终地，我们希望ovz在dbcolinux里究竟要达到什么样的效果呢？

对于问题1，因为pelinux是常驻的，包含了ovz的pelinux也是普通linux，我样可以以此为模板不断制造新的硬盘操作系统(the system0-systemx)，这样的好处是我们能自定rootfs到/systemx下并boot into chrooted systemx的方式进入它，由此我们做到了OS级的虚拟容器且更轻便。。对于问题2，云服务器的本质就是各种容器和容器化，包括OS级容器和APP级容器，因为OVZ本身就是OS级别的容器所以通常认为它不能用来替docker这样的东西，但想一想docker那种用了分层文件系统的容器它只是将文件隔离在了各层，我们是否可以利用ovz本身的方式将OS虚拟视为应用虚拟，打造一个应用级的容器呢（共享内核，共享rootfs，仅应用容器自身的内容被放在这个容器）？这里的技术可以是：从system0开始，每一个新增的systemx都共享了整个机器的内核和rootfs，仅将应用的数据或程序放在这里，这样资源占用形式其实跟docker分层差不多。这样，一个systemx可以是OS容器，也可以是APP容器，看你怎么看待了，反正ovz使之oneapp oneos的理念做到了极致。

>那么为什么一定需要这种应用级容器呢？为什么ovz和docker这样方式的ovz要共存呢？

>举个例子，曾存在一种讨论，PC上的多桌面是不是必要的，一帮人认为多窗口多任务有了，多桌面实际上只是在同一个桌面开多个窗口，在窗口间切换即可。但实际上有了多桌面/虚拟桌面来归类这些窗口，在窗口粒度层切换其实也是很方便的，尤其对于一些需要保持多个任务在线，且满屏应用的PC使用者来说，，，，这有点像多显示器。多显示器更进一步，它是虚拟桌面/多桌面不需要切换的一种，扫视即可切换。

>所以，正如多桌面多任务可以共存增益的道理一样，其实ovz这种OS级的容器和docker这种APP级的容器都是需要的（一个共存OS相当于上面讨论的情景中的多桌面，一个共存多容器相当于多窗口）都是需要的。诚如《带pelinux,带分离目录定制（3）》文尾讲的，我们需要使ovz成为应用级容器.

这里的技术细节会是哪些呢？正如上面不断提到的，所有的技术归宗于chroot+定制rootfs template来解释。我们可以把ovz想象成chroot管理器，必要条件是我们依然要在单机或ECS上装是linuxpe带ovz，并以linuxpe自带的rootfs为基础准备一套rootfs作为模板,通过mount源模板rootfs虚拟路径到systemx的实际路径一一对应建立所需具体rootfs的方式。pelinux livefs中的rootfs只是映射到了/hd/systemx，不破坏它作为管理系统和元系统的livemode。然后：我们设想kernel是bzimage,rootfs是initrd，它们二者由grub的linux指令和initrd启动,于是按照linux的启动技术：

我们可以使系统开始启动的时候就进入一个chroot(使用busybox的switch_root，pelinux所在/被清空，将/过继给了硬盘中第一个启动的rootfs — the system0),只是如果这样，那么往后的系统便无法按ovz的方式再开，当然通过其它途径再开是没有问题的不过这样就失去了ovz的意义仅是一个单机dual booting方案,这类似于单机下多OS dualbooting的设想。dualing boot multios but anytime one os running)，然而这不是我们需要的。

我们需要的是：不清空pelinux的所在的liveos，在它里面chroot分化容器：再使用chroot（exec /usr/sbin/chroot /systemx /sbin/init）或ovz的vz enter切换进入容器的方式。这样的话，我们需要为每个systemx下的busybox init进入的路径指定一个x，就如同为kernel提供一个chroot参数一样，这次因为我们是从已运行的meta pelinux中chroot，而我们定制的rootfs只从/system下启动，所以，我们要1，改从kernel到busybox级支持chroot，2，支持/system路径的参数化。

实现了这二者后，那么实际上diskbios可以叫anybios/containerbios了，单机下为bios for disk，容器或ecs下叫anybios/containerbios。

--------------

这样，在ecs上开多个OS，和利用chroot在旧安卓手机上安装linux这样的课题可以统一了。我发现老毛子在OS方面造诣很深，win10精简,reactos,还有这个linux-deploy，linux deploy也是那种同时运行二个linux的chroot，可以用usbwifi连网使PC与mobile同处一个局域网，且mobile作为移动nas代替我《一个设想：什么是真正的云，及利用树莓派和cloudwall打造你的真正云中心》提到的树莓派。

还有，我们在前面的文章中谈到couchdb使sql结果也得到持久，，实质为同步。既是一个dbserver，也是一个appserver，重点不是这个，我们还急待使cloudwall与elmlang联合，使cloudwall中inliner/ddoc的IDE环境中可以动态可视调视。。

----------------------

github - minlearn下载,如找不到本文可尝试百度搜索本公号名字！！



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340253/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



