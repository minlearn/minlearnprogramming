群晖+DOCKER，一个更好的DEVOPS+WEBOS云平台及综合云OS选型
=====

__本文关键字：dualrunning os bootloader设想，dockerized os subsystem appmodel,云devops学编程__

经过前面对于群晖的讨论，我们知道它是一个从bootloader到os都很有特色的系统，我们重点讨论了黑群晖的bootloader能安装到不同平台的方式，我们还讨论了如何更好更省事地使用常见群晖套件实现单文件夹同步+同步套件省事同步，及配合访问点服务器使用(frp,公网IP盒子)的方式。，甚至讨论了使用webstation作code snippter空间及利用docker实现类似live code snippter hosting空间的功能，后来我们知道这是devops 

综合上群晖总归是一个好用的web化的云OS，这要归结于它可以安装到远端，平台管理和APP都是基于WEB的，也要归结于它支持VMM和docker可以分虚机同时运行多个VM，最重要的还是docker ------ docker是一个同时支持应用和平台虚拟化的东西，docker可模拟subos lxqt效果,也可实现devops.下面我们细细道来。

关于云OS的bootloader pe，从diskbios到cloudbios
-----

pe和bootos越做越复杂的情况有很多，如群晖的webassit，它实际上是一个纯净的pe和liveos,我们称其为dsmpe，包含了大量大容量存储和网卡的驱动的OS，本身并不作为直接可用的系统存在一个安装到实机的驱动解包和适配过程，负责系统功能的是pat，webassit负责的是建立可用的磁盘结构安装和引导pat并后续更新，另外，一台机器开二个同时运行的OS是很现实的需求（one client,one server,非dualboot而是dual running），实际上在xaas系列中我们不断看到过这类系统，如host/guest os(colinux cooperative but not dual running),虚拟机，docker OS，subsystem OS,crossover 模拟器,cloudwall(VS前面的方案，唯有它建立起cloudbox,cloudos,clouddevops,cloudappmodel全包的方案)那些，但我们并没有接确到一种能在系统启动时支持双OS会同时启动并运行的PE或工具。

diskbios即是这方面的努力，在diskbios设想中我们提到在一个类似WINPE的环境中实现虚拟机管理的功能，且在发布dbcolinux时我们在一个linuxpe中实现了集成vps管理器的功能，结合OpenVZ Web Panel管理这样的东西，我们可以实现dsmpe类似的东西，多/system(x)这种方案既保留了虚拟机方式也保留了docker方式的虚拟粒度还很自然化。------ 但那文并没有讲到如何使这些VPS运行起来，如何引导进入的方式，------ 提供PE和如何提供双OS，其实这可以是一个相关的问题。

>当然如果直接采用PE中套虚拟机管理器+集成VPS WEB plane的想法会更简单，但除了WEB plane,还有其它更优雅的方式吗，传统类grub boot的方式会更简单有效么？比如，它或许可是一个强化的grub loader，比如为dbcolinux增加dualos bootloader running功能，可以直接单次boot二个系统，一个OS是linux+vnc thin	client，然后选择性boot into guest os through grub.....这些留在后面讨论。

关于云OS本身app和硬件。从minportalbox到minlearnbox,从WEB appmodel到私有gui appmodel
-----

经过前面一系列的xaas讨论，我们明白我们要追求的generic os其实是一种涵盖支持realhw（见《利用colinux打造云环境》）,云主机(见《阿里云winpevirtio装ISO》)，无屏小主机(见接下来文尾《mac mini 2014上装黑群晖》)，虚拟机等硬件装机环境,提供支持os subsystem(win10 wsl,fydeos anriod container,wine appcontainer,linux container) app，webapp,remote x11 gui app,local gui app（dsm lxqt）的云OS，它有这么几个特点，1，要支持多硬件平台，要能以web方式(准确来说，远程都可以,分布式指代远程，也指代一种可负载容错的多节点结构)支持安装分发程序与OS系统本身，2，要支持虚拟多OS APP和多种远程APP，----- 而这，其实就是云OS的一般特征。

我们一直提到和实现的diskbios tinycolinux，就是这个最终目的和generic os的概括，所以现在，这个diskbios不妨称为cloudbios os。

群晖作为一个很好用的WEB化的cloudos。支持web appmodel(page ui appmodel),它还有硬件方案，符合cloudbox->cloudos->cloudappmodel全包的方案，但却没有clouddevops支持，需要挂靠docker，论更符合传统方式的云OS或更集成化的云OS，还有fydeos和cloudwall这种，fydeos虽然是客户端的但是也可以用在服务端，它支持docker os as subsystem/guestos,和多种subsystem appmodel

群晖利用docker很轻松能实现这种docker os as subsystem, subsystem appmodel,linux视图形为APP gui model，加入了协议和网络，形成了remote app model，即x11架构，这对于本地游戏没有优化，然而对建立远程程序天然形成优势。我们在前面说过，任何一种逻辑栈配上一种GUI栈，就是APPMODEL，vs vnc和远程桌面，x11可以直接从docker subsystem中透出，如利用xmingw，或dockerized x11 gui appmodel这种remote app结合teamviewr app窗口模式这种或硬件化的oraykvm这种，如果能做到这个，我们就可以将没有port到那个平台的客户端以remote app viewer的形式投射到那里。类远程桌面。等5G一提速，云streamable游戏和remote gui app就兴起来了，web兴许不用了它的时代就终结，因为WEB始终是一种过渡方案，它体验不如原生，它的优点是管理难实用，像云游戏做成webgame其体验就十分不好。

>还有，fydeos这种docker as subsystem将docker维持在os subsystem级，还有一个好处，因为每一个docker app总要带系统镜像然后才是分级的APP联合文件系统，docker app总是有点粒度太大，as subsystem复用就强多了，类似anriod container app,linux container app,crossover windows app(docker wine appmodel)，就可以充分抵消一个APP一个OS的docker aufs带来的性能和存储损耗。

>甚至还可以有，私有APP model，unform server/client app model,可以使一个程序的界面都托管在云上，就不用专门的客户端开发了，甚至瘦终端可以是仅带x11和vpn的手机。

而cloudwall作为云操作体统的特点在前面我们讲到是多端原生同步化，还有其devops特性。这利用群晖加DOCKER也能达成。见下。

关于云OS的devops,从yunappmodel到ci backend yunappmodel
-----

云OS最鲜明的一个特别是其对DEVOPS的支持，linux+docker可轻易实现，docker in docker和docker可透露volume1服务可以很容易使之支持CI变身devops。其实本地也有devops,像cpp的sandstorm用的那套ekam，https://github.com/capnproto/ekam可以是devops,gitlab runner based也可以是。关键是有一个可以编译源码且自动化的程序存在，无关乎你将它做进docker还是什么东西，而docker不但能CI编译也能运行构建后的APP。docker很容容易被做成ci builder。

在群晖上利用DOCKER也能实现DEVOPS。


----------------

这应该是xaas系列最后一篇关于群晖的文章了。整书关于整个现代编程开发的选型，就是围绕，云OS，云devops开发，云APP，展开的,整个demos选型也是这样，所以新demo中先提出一个os,再jupyterengitor再APP这种，但其实cloudwall这种才是最简易和全包的方案,为了学习起见我们做的是从基础做起的方式，或许我们以后会权宜先在群晖上把dbcolinux docker化，在群晖上把deprecated demos迁移到docker,再考虑后续为它建立jupyterbackend devops支持，甚至自有硬件支持BOX化的方式,所以bcxszy不如叫cloudlearnprogramming。mineportalbox不如叫cloudlearnprogrammingbox




-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340478/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




