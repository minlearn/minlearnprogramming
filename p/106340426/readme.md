一个matepc,mateos,mateapp的goblinux融合体系设计
=====

__本文关键字：将桌面环境，toolchain设计为subsystem,rootfs as Xaas,rootfs层次的虚拟化,非Virtual OS Infrastructure,第二PC，模块化机箱，第二PC，存储，计算分开机箱，nas另置主机，mirror os,mateos,自建icloud,本地远程通用的云os,云app__

在《bcxszy:applvl programming》整个选型中，虽然我们经常强调xaas,langsys,appstack,app四栈一体的开发，但xaas跟其它部分在本书中一直是分开的并割裂成二部分，我们一直将眼光聚焦在applvl开发，却视xaas平台为devops工具,装机运维和虚拟化容器层面的东西，这些只是高于开发并不影响开发的东西，它本身并没有任何开发元素的整合：比如一种语言，一种运行时，一种库，一种appmodel，一种问题,etc..，因此它本质上跟applvl的开发并没有得到整合，本书目录制定上，第一部分的xaas部分《OS/TOOLCHAIN LANGSYS/XAAS》主要解决接下来为《APP LANGSYS/MIDDLEWARE STACKS/DEVOPS TOOLS》定制xaas服务的过程，xaas只是一直为applvl开发作铺垫，在《bcxszy》的序言中我们的选型也一直是语言系统和开发开始的，没有平台选型相关的部分。

这是因为传统XAAS系统编程领域与applvl编程领域是分开的，它可以不是直接的app hosting影响开发 —— 事实上，大部分虚拟机语言和开发体系都是这样处理的:将app托管在另外的hosting，脱离和超越native平台的编程和软件开发才是主流。大部分专用os也都是这样处理的如lnmp这样的东西它强调的是一种串序的四栈结构，它将linux作为xaas,php作为langys,nm appstack作为appstack，把webapp作为appmodel。

而xaas可以是一种新型的平台，因为慢慢地，在整个第一部分，我们发现，我们也可以将一些applvl开发的部分及早地上升到os层次。如GO作为分布式语言，它更进了一步，它将lang vm整合进入了app。如plan9和usermode plan9，它将分布式协议实现在kernel或rootfs，进而影响这种OS xaas下的app开发。——— 也就是说：在开发相关的四栈整合处理中，它可以以整合后三者的乱序方式进行，甚至它能居于langsys和appstack之上或寄宿其中。整合《bcxszy》的第一，二部分为真正的四栈一体的一部分。并制作出一种新的专用OS - goblinx。

Xaas的选型：高于开发的部分，和融合开发的部分
-----

来归纳一下整书我们对xaas的选型路径：

在《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制》系列中，我们讲到了一种host与guest分离，支持实机虚拟化，并良好组织在/system下的linux设计 — the pe as mainly装机系统，那时我们集中于装机的想法，我们想得到一个类似esxi 的东西，相信兼容os一直是人们的一个梦想，而我们一直都没有到达过，这种兼容OS的思想在《兼容多OS or 融合多OS？打造基于osxpe的融合OS管理器》一文中再次被提到。

在《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式》系列中，我们加入了devops思想，并去掉了为实机/裸金属/云主机作装机虚拟化hypior的首要需求，弃ovz选用这次的lxc主要是为了devops。lxc可以作为app容器，也可以作为系统容器，我们还是将其实现为一个pe。

这一代的dbcolinux是一个融合os(比如用osx base当parallesdesk的os，windows和osx作为其subos,这样osx base就是pd的专用os了)的不断增强，接下来的二个增强就强烈涉及到了开发：

在《打造一个Applevel虚拟化，内置plan9的rootfs:goblin（1）》中，我们提到了整合go,plan9到这个linux的思想：并提到了这样做的底层依据：整合busybox或plan9这样的live demo服务到每一个app作rootfs虚拟化，将虚拟集成到APP级，可以促成无架构APP，最重要的，将虚拟化做到app级，可以将devops也做到app级,比如，工具级的IDE就能生成app包做到包管级。利用一个工具Provisioner了，这二者可以达成最简单最实用的编程典范。plan9这种协议，是天生的本地远程同步/通讯协议，可以免协议开发(因为它是demo级的，甚至不是一个lib)在本地和远程都能运行的APP,且无修改地工作在一个OS下，就如同git p2p协议一样。由于它也是一种同步协议，甚至可以为每一个 app建立如pouchdb一样的内容同步机制。这样更接近天然云化了。

甚至hyperkit,Lektor:用开发本地tcpip程序的思路开发webapp,Shell式编程：会用就能编程，把问题域整合进系统域。这种思路，我们都提到了。

最终在《一种含云主机集群，云OS和云APP的架构全融合设计》中，我们提到分布式架构的本质问题是OS和APP的选型和实现问题，这就是说，在整创出新xaas,langsys,appstack,app四栈处理中，OS是一个可以将APP和APPDEV上升先行的路径，比如在OS级自带分布式支持的系统肯定出来的

本文即是上文《一种含云主机集群，云OS和云APP的架构全融合设计》关于goblinux的特化版。

Goblin基于上述融合设计的强化
-----

在上文《一种含云主机集群，云OS和云APP的架构全融合设计》中，我们从这种OS适用的硬件，集群开发，所用的OS到所开发的APP都讲到了,那么现在，我们继承从tinycolinux->dbcolinux->goblinux系列的成果，一点一点，将前2者附加到整合了后者的版本上去：goblinux的实现实用版本。

在硬件上，goblinux，就不追求多PC集群了，为实用起见，它可以是二台PC，大体上，这二台PC是计算和存储分开的机箱设计，你可以想象它为一台类似群晖mini itx机箱nas上面又加了一块主板，市面上这种mini itx模块化机箱蛮多的，但双主板和不多见，在这种设计中，主存储的主板可以用nano itx或更小一点的pico都可以，主计算和日常使用的那块主板可以接近一台普通mini itx。如果可以，你也可以三主机，一台路由器主机，但实际上，路由器更适合与前二者分体，这二台pc，一台装gui生产/开发环境。一台装file server/mirror cloud环境。都是goblinux子系统。

在OS上，也可以是一个管理器下的二个融合OS，主pe的goblinux就是像群晖那样的rom boot = grub+kernel+srs driver libs+bootpe，它很小。这个主PE goblinux内含三个系统，一个guy desktop,一个file server， 一个connect server, 对应上面的三主机。前二个OS是标配，要做成融合结构，代表分布式的本地/远程端，一个GUI，一个mirror tolocal远程osstub,它里面没有app，不需要维护，是个黑盒,没有界面，只有数据。但是也不给维护接口。所有的操作在local mirrored gui操作系统，Mirror os可用于装在上述提到的本地另外一台nas pc上，也可以装在纯软件融合os管理器下的file server子系统上。file server也可放广域网追求异地文件备份效果只是交互性能大减。可脱网使用，重新联网时，将同步远程。注意这并不是VOI,VDI那套。

开发支持上，如上所述，这是一种天然的分布式APP和云化APP开发内置的软硬结构。它有以下几个特点：

如上所述，用go，且利用9p实现无须协议交互的本地远程p2p，写git之类的东西，因为plan9有os级的实现也有linux的userspace实现品，甚至有9p lib。demo态，可以in the kernel, or embedded in a subos,rootfs,or app，运行态与开发态的区别是，demo级rootfs的9p是运行级的不需被编程，而开发级的是预编程。9p usrspace 9p是demo level级的。为现阶段简单起见，plan9p只集成在usrspace级，且用demolvl的。

—————

本文过后，我们会一步一步整合实现接近上述理想化的goblinux，无论如何，我们这是在将平台选型慢慢地实践弄成一个现实品的过程，和强化goblinux的最终过程。

Xaas内容过多，为清晰化起见，我们依然选择不整合第一二部分目录。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340426/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





