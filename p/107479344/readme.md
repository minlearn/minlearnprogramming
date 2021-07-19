一种设想：在网盘里coding,debuging，运行linux rootfs作全面devops及一种基于分离服务为api的融合appstack新分布式开发设想
=====

__本文关键字：Jupyter visual debug，基于网盘backend的ide和snippter空间,debug driven programming，make chrome like visual debug for every lanaguage，make every language a dsl,c系语言学习最好的时代__

2020版的macbook要迁移到arm了。这说明，云时代去终端的thin化努力在继续，未来我们的PC和手机会统一(请注意我指的并非是笔记本平板一体机，变形本这些玩意层面上的意思，而是架构和生态的变迁)，而云端始终配合终端变化，走的却是性能不断加强的fat化路子，不光硬件如此，《利用大容量网盘onedrive配合公有云做你的nas及做站》我们已经知道，云开发时代，本地的存储和带宽已经没有了任何优势，有一些资源是本地化不可能得到的，它们只存在于云端。群晖这样的nas跟onemanager+onedrive这样的组合比已经没有了太多优势，会越来越衰败。越来越多的本地的东西迁移到云上，成本也会越来越低，这是软硬一条龙上的变革，最终应用也会越来越倾向利用云计算，这所谓的一条龙，包括os,开发系统，也包括组成应用的各stacks本身，当然也就包括代码托管和开发方式的云化甚至软件的定义（跨多语言组件交互已成必要，中间件和service api原来分布式开发元素上面也有新变化），见下：

jupyter for od，让文学编程与网盘后端结合，,just another devops alternatives to cloudwall and github ci/di
-----

首先是代码托管和云IDE方面：在《群晖打造snippter空间》等文章中，我们从云收藏APP的角度讨论了用网盘后端来保存snippter这种特殊内容的方法，就像收藏视频，收藏网页clipper一样，收藏code snippter其实也可以像跟收藏文本一样简单，但相应地，vs 视频之于视步转码云服务md文件之于markdown rendering，对于codesnippter也可以加入复杂的云上再处理（srvside or browser/client side格式化渲染显示,ssr），甚至，更进一步，将它与复杂的netdisk storage backended devops和工程文件组织环境，ci环境，云ide构建环境（云Ide和开发的极大化，就是devops)，。建立起关系，

在《群晖gitlab+docker打造devops》《jupyter设想：enginx,engitor,passone》《cloudwall》当中，我们都介绍过devops:一种将开发上云的手段，这三者都有某种程度的devops支持/都使用某种storage/都建立了某种appstack/都支持一种多种语言，比如，群晖上gitlab是用本地存储作storage,jupyter也是，不过它是一种文学编程工具(librette programming允许我们使用更丰富的文档与注释输助代码)，而且它的devops功能和云IDE很强（最近，jupyter又强化了它的visual debuger功能，使用xeus-python甚至可以可视化高级数据结构和内存对象树,比我们介绍过的《elmlang》这些云可视debug ide还要深层和完善,我们知道elmlang 等visual debug的作用主要是make chrome f12 like visual debug for every language，make every language a visual trigger/action dsl，就像chrome仅能作客端js调试的道理一样，它能使任何编程变得像chrome visual debug和接近web开发调试方式），cloudwall则用couchdb，couchdb不仅是对象数据库还是http应用服务器，由它组成cloudwall这种很内敛却selfly containing gui/http/storage的appstack，它虽然仅支持js(但却可以视js同时为数据和代码，天然可以在线coding和debuging同时mateapp方式客端即服端方式运作 --- 可以说cloudwall是我们见过的第一个完备的mateapp和mateos环境)，jupyter可以多语言，它的功能核心是各种protocols,appstack为类似普通lnmp的web page app(.ipynb),而gitlab，github这种，只有devops中中部分ci/cd功能(github actions,etc..gitpage generation)，离线客户端也是重要的部分，却没有鲜明的在线IDE支持这些方面，线上部分主要为在线hosting代码版本，appstack不限。

但如果能将这种jupyter ide和living coding,debuging做到网盘里，以网盘为存储后端，代替github等coding hosting空间，情况也计会是另一种美好，比如ipynb存在网盘里作为托管codesnipter和工程组织文件，可以直接在网盘里运行语言和部署应用，结合jupyter的visual debug，我们可以在chrome里调用装配了服务端jupyter支持的情况下，我们可以利用服务端rendering，哦不，服务端debuging。这样，网盘后端的jupyter实际承担了，后端language baas，存储，coding,debuging等在线开发全支持。---- 所以，它十分类似cloudwall和github的整体或部分devops相关作用。比如，它还可以用jupyter-book这样的应用达成gitpages,届时，我们甚至可以直接在前端做站,在前端写内容，在前端直接开发调试后端(我们可以将其做成，为每一个app装配一个livedebug，Jupyter backed debugger inside app,比如正常情况下显示程序本身，press esc 会消隐切换到一个visual debug环境)。markdown也直接写样式统一html，我们知道md可以代替html用，这样就有了三个统一，开发调试统一，前后端统一，内容样式又统一。。

利用云booter，在网盘里运行os,作全面devops
-----

实际上，我们知道，除了语言后端和devops工具，更强大全面的devops需要docker或虚拟机来参与构建，虚拟机和docker始终显得太重，在《一键pebuilder安装deepin20中》我们提到一句后话：为了mate os下的开发，我们还准备了可视开发和网盘内devops支持，这就是整个文档集《minlearnprogrammingv2》part1提出的一种云booter：它更轻量，工作原理类似系统级的虚拟机，在boot和firmware级集成虚拟机支持，自带Linux kexec protocol，可以在启动booter时启动多个资源允许内的rootfs as os container content,使得app和os,subos一个性质都是rootfs。

ps:如果说在网盘里运行jupyter是对minlearnprogrammingv1 时代engitor概念体的强化，那么新的pebooter则是对v1时代diskbios的概念体强化（目前我们仅做到网盘云装机）。之前我们在文章中研究了 colinux,openvz,lxd等虚拟和多开方案并企图covering all硬件平台包括实机，但我们现在正式转到云boot上来并局限在仅云主机，天然多os和多开类colinux的rootfs级虚拟化。更适合云主机。

最后，当云开发，云devops,云代码托管都被做在了云上，再加上这里的云os，这所有加起来的好处是，我们可能并不需要一台真正的pc作开发和应用，不光内容同步可以整包打走。，甚至可以真正做到去PC戒电脑，这个clientless仅指去x86 pc化(比如我们可以选择更省能的arm平板加一台x86云上装有mateos和上面jupyter,云booter的服务器作mate pc。实现去pc化)。

甚至，当这一切发生变化的时候，云化开发方式和appstack也要经过云化：

我们知道，较之cloudwall，其单一性也是其自身的优势也同样是其明显局限，jupyter支持的多语言加网盘后端模拟的cloudwall更符合通用的情形，但是要达到用jupyter精确模拟cloudwall式mateapp的效果，需要更多额外的工作。

其中之一是需要统一的api定义和调用方式。

matecloud virtual appliance:一种统一web appdev和native appdev,基于api分离及服务组件化的融合分布式appstack
-----

在《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》中我们提到“一种可能行得通的appmodel设想：P2p 客服同体，只需sync api结果即可的app”，在《利用开发tcp思路来开发web》，我们提到利用wpcore as service，在《利用大容量网盘onedrive配合公有云做你的nas及做站》中我们见识到腾讯云函数serveless,这其实就是以前组件技术和远程方法调用，及webservices，serverless只不过倾向更多指免运维的极大化。我们还发现reactive page/pwa这种js库，缓存页面在客户端，反应很快。类桌面效果。因为它是使用web api的，前后端通过设置api分离。所以客户端html和静态资源可以缓存。

PS:甚至于我们上面提到的这种c++ xeus debug protocols，它实际上就是api（应用处）和语言接口（语言处）的一种类似体，我们知道，web天然就是服务化分布式的，web编程的实质在于service化，和service调用，面向产生和使用各种service api，这是一种与本地api开发和分发完全不一样的形式，是不同内存模式的多语言处理异构系统交互的方式，涉及到多语言和调用组件定义。然而我们现在的web是将html与后端逻辑形成一种组成app的stack，这样，实际上导致了界面和业务逻辑不可分，也导致了云服务应用中，客服cs/bs不可分，失去了streaming html as page gui的好处 ----  真正正确的做法，应该是仅把html分离作为gui content only不集成到appstack as gui，采用js/reactive的做法，后端与前端完全通过service api分离，后端定义出业务逻辑api用一个rpc表达出各种外部服务接口，供前端（它只应是一个js/html reactive客户端的程序）使用，web api与web服务，使前后端真正分离,将界面渲染全面放到客户端reactive pages，没有任何服务端内嵌html渲染html的逻辑。整个应用用传统web的mvc来解决，比如onemanager的类比物：cloudreve就用了这种技术，如果onemanager来采用，展示速度又会提升很多。

这种分离客服reactive page纯客户端渲染方式，像极了native app，而本地开发其实也可以用这种方法。为前端调用定义出api。后端透出api，又可将本地开发和远程开发统一一个模型：比如实际上deepin的qt+go backend的deepin ui appstack也是这样的思路。除了界面，存储也可以以不掺入任何界面渲染的方式进行，仅是另外服务区导入进来的api。这就是各端分离。云函数纯api化，单独开发调用，通过rpc交互的思路，又迎合现在跨多语言开发的方式。

如果结合上面的mateos cloud boot技术，可以将每个这种app欠嵌一个vm，做成virtual appliance。只有web真正跨三端如果所有界面都用js/reactive呈现可以达到原生渲染的效率，这样就模拟了go，且不再局于Go和 go app 是最好的virtual applicane选型的说法.

本地app和远程app融合的最高境界是microservice app，即不光数据解决了sync，程序逻辑本身也一体化。这其实要跟mateos相协作。见打造一个全融合的云《一种追求高度融合，包容软硬方案的云主机集群，云OS和云APP的架构全设计》。


---------

本文注定晦涩难懂，这是一种需要操作系统改革协作的工程，所以为此我们提出了mateos和直到mateapps的全设想，一个建立virtual applicance支持，一个建立appstack支持，相互绝配。这种分离也为编程教育划分了新的实践方向，我们会另写一篇《实践即工程与反工程》作序言放在《软件即抽象》下。因为实践所需要的工程规模选型需要同时涵盖本地开发和web开发域，搞懂这里的技术本质和经过几个大小这种工程，就可以由纯学习上升到参加工作所需能力的实践量了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107479344/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



