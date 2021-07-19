一种matecloudos的设想及一种单机复杂度的云mateapp及云开发设想
=====

__本文关键字：可编程的os/os kernel/os rootfs,os as service,os as service,mateos。cloudsubos，，客服同体，api/runtime共体，将os api化，headless os core for cloud api，融合云app__

matecloudos,matecloudapp:真正的分布式
-----

在前面我们谈到《enginx,engitor》系列，还谈到《Plan9:一个从0开始考虑分布式，分布appmodel的os设计》，这些文章共同点都是对已有分布式app（一类跨OS/OS进程的APP）的思考和未来创新设想，可以拿来从各个方向类比：前者是传统os+lamp,lnmp方案，以OS上安装的分布式软件作为OS服务子组件（多见于服务器OS环境）透出它们的服务，在这上面打造出一个web appstack(比如apache之于page gui,mysql之于持久db...)，在这里，可利用的服务，和API存在于这些组件抽象给语言的后端（非OS固有部分），而后者plan9的本质不同之处则在于提出了一个devable os，将内核中的对象和os本身作为可编程对象，透出这些API服务，直接在已有OS组件上作分布式不提出LAMP之类的多余结构，在这上面打造分布式APP。

后者显然更原生，也更好用，（如上，传统的os只是api runtime/c dlls，api服务也是离线的另起一层，且并不面向分布式，并不是与开发一起被设计。是先把os用起来的原则建起来的，os与api分离。以前，为了达成跨语言跨OS的API，是靠给API打stub完成的。这种方式粗鄙无用已被淘汰不单是rpc技术改良就可以完成的）。而新时代分布式程序和分布式开发的固有特点是：分布式开发要求开发接口与程序本身共体不分离，也就是脚本语言和web开发中,“srcfile即程序组件”那套，------ 当然，现在的web分布式是从lamp上搭建的，现在的OS kernel也都基本不是plan9这类。但我们可以在现存传统的os结果下进行改造。比如，我们可以另外铺设API。将OS作为一种headless api被调用环境。类似传统方式铺lamp服务那样的方式进行：只不过这次向上提升到针对os的组件进行。也即，我们利用plan9思路和方式，从传统OS本身开始做而不下放到lamp，OS服务要作为基础被分布式服务化，发展基于OS分布化之下的分布APP。而且不必涉及到统一使用者的OS跨OS。，如果成功。本地开发和分布式分开完全可以统一。------ 引申开来，即：os/os kernel as serivce，api和api runtime天然一体。组件即demo即api。本地组件即远程服务(自动API化，无须再次面向不同语言作显式API化)。

新mateable分布式用于最小实践教育路径和降低实践门槛考虑
-----

这样做具体有什么意义呢。

matecloudos和matecloudapp，由于OS都是自带文件系统，GUI，和持久的一类综合体，这些服务不必再搭建，作为客户机和作为服务器环境的作用可以统一彼此互为mate，这些理念对于APP开发的统一作用和难度降低作用不言而喻：

首先，（1）第一条，如上所述，本地开发和分布式开发将共享同样的技术基础，变成单机复杂度，我们现在的分布式分开几乎都是统一后端和webapp这二种理念：由于webapp是现在唯一的真正跨端的模型。。我们现在大部分移动端和一些native端应用都是通过分离前后端为统一后端+不同的客户端APP技术完成的，将所有分布式开发视为前后端分离，后端统一API化，都是web形式的调用，比如某个网站后端，或一个天然的cloud function，前端则利用统一的ajax,reactive,pwa技术构建。即所谓的MVC模型。这些都在《一种设想：在网盘里coding,debuging，运行linux rootfs作全面devops及一种基于分离服务为api的融合appstack新分布式开发设想》说过，如果os本身被api和服务化了，那么完全可以用native app逻辑。不必另造一个webapp出来。来达成分布式APP，简化后端。

（2）甚至，我们完全可以在类plan9的理念下统一，使分布式APP和本地native APP开发理念同源，再度降低APPDEV的难度，我们的开发一般都是某种app层次的dev，需要呈现为一个APP形式，比如web在是webapp，移动端是mobileapp，桌面就是nativeapp，软件和编程就二种逻辑和抽象：一个业务逻辑，一个关于APP本身的appstack逻辑。appdev的本质是三种stack（一般是gui,持久，网络）的形式在所处不同形式下的演化。比如文档数据库。就是视图模式下的“文档”，区别于存在于磁盘中的文档。appstack占了一个app逻辑的大部分，可以极大得到简化。

一个例子是类似p2p性质的gitcore客户端程序即是服务端程序，可以以同步的方式代替协议交互。也可以同步的方式来处理数据交互。比如，直接把原生GUI透出为分布式API服务，客户端也可以是远程桌面remote app这样的东西，如果客户APP也是服务端一样的OS，则GUI无须渲染（只需像那些远程桌面协议一样传送位移量）提出一种新的类web的remoteappmodel。天然分布式。

（3）还有，如果是plan9这种kernel，由于整个os内核都是构建于基于对象上。所以可以以编程的方式来调用和共享，以OS抽象和语言统一的方式（这个作用不得了，见《一种shell programming编程设想》，统一问题域和抽象域，甚至语言方案域）同步二个OS间的对象。这样在OS的构建过程中，就穿插预埋了对未来这种OS上的APP编程的设计，甚至统一了APPDEV和这种OS上的系统APP开发。

（4）更多，在线IDE就随处可放。调试也不再基于堆栈跟踪这些反工程层面。。。。。。

----

我们的设想是基于deepin做一个这样deepincloudos，再在这上面发展一些好用的matecloudapp(集成云开发IDE和云LAMP，首要的就是在线IDE和mineportal apps:如file explorer内置同步，如果能在jupyter上写程序，那么这是一个天然debug backend app，已有这样的产品，不过这是工具层面的，我们就是要将其做到与OS级天然集成。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108172262/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


