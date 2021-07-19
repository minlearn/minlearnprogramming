去windows去PC，打造for程序员的碎片化programming pad硬件选型
=====

__本文关键字：混合，多合一OS，封闭应用，有限应用机器。程序员的硬件选型，程序员的7寸umpc programming pad，移动编程机器__

在前面我们谈到nas与chromebook的绝配，一个是webos，一个是web client os，群晖中的docker不但用于开发，运行，和devops，甚至可以用于”新式装机“，比如群晖中有lxqt,qnap中有多种guest os,他们大都基于docker技术或相似技术，如果说webos总是出现在host/guest os的组合中使server OSs能做到不断融合和generic，，那么作为客户端OS的chromeOS+chromebook，也有类似这样的演化方向么。

chromebook with fydeos PC选型:去windows,去PC
-----

这个是有的，fydeos即是这样一种host chromeos与guest andriod,linux,windows混合的定制版，fydeos也是用类docker技术实现的三种OS混合，它的基础是google的Crostini，当然，可以用，和用得舒服自然，这二者的差别可以是一个天，一个地，fydeos能运行andriod和linux的能力是很彼此相关的，因为他们都是linux系，可是运行windows程序需要借助docker wine化的codewears公司的crossover层，实际效果只能是勉强带起，很不令人满意,这里面需要太多工作需要填充业界还没极尽完善 ------ 但不论如何，第一次，当这些生产环境能第一次结合，且真正第一次能证明能协调工作时，这是一个伟大的进程。

>为什么这么说呢？从《除了linux，我们有第二种开源OS吗》一文开始，我们就在探求一种能实际用于生产环境的多OS实现选型，就像我们在选型多语言系统一样 ------ 我们的实际需求有可能是这样:我们刻意去windows，去PC，理想将一直只工作在某linux下，但经常或偶尔地，我们需要windows下面某个zip程序，办公需要wps生产力环境，至于游戏嘛。。就算了，我们只是先想得一个这样的办公生产工具，以上有限几个程序最好都暂时能工作在这个linux中，而又不想总是依赖一个windows pc设备-----未来，PC，windows和这些有限的windows app都在我们要淘汰的设备和app之列。

>然而，在windows下融合linux较容易（colinux,win wsl）反之这个过程却难多了。业界努力多年，但这二种APP一直不能以自然的方式共处，并得于实用。----- 一般地，做win+linux混合os方案或混合应用的，首先想到的就是从kernel层着手，像龙井系列，也有从subsystem着手的，像win10 wsl，colinux，也有从应用层着手的，像virutalbox,也有从api层着手模拟的像cygwin,wine，这些都带来了超多的工作量，都是因为OS从入口处就分别过大，而APP构筑于OS kernel的生态之下，仅想模拟app绕过kernel是不现实的，或者说是非常困难的，------ 设想下从kernel层着手，只能使用内核调用层的翻译技术来使二个系统生态相处，打破了windows kernel内核本来就具有生态封闭的特点(虽然有开源reactos，但它本身发展未演示进beta)，强行揉合和翻译内核调用会带来超多的工作量。如龙井就是这样。windows本来就kernel和app不易剥离，想要集成更是难上更难。除非真采用为fake一点的方案。

这样的退而求其次的方法就是wine,加上容器技术，它可以用相对不原生的方法从APP层开始，这个时候性能考虑已经不重要了。通过使用crossover，fydeos终于可以第一次勉强把这些放到一台机器中。

程序员的新硬件选型:移动端a programming pad
-----

在前面的文章中我们谈到nas与chromeos绝配时，视chromebook,pc,smartphone为nas的集散型结构的三端，并有利用七村小本加vim打造程序员的programmingpad想法，这一切的考虑都是为了向程序员-这个特殊的群体倾倒，毕竟，除了一个nas as mineportal，程序员依赖以上终端的日常实际上很典型:追求高效输入，碎片化文字创作或代码调试的人群，这cover了好多人使用终端的场景。

硬件选型上，当一个现代程序员的装备越来越碎片化，越来越自动化，那么其需配备的终端的演化方向又将如何呢，

access box:

在选型群晖或qnap时，我们总是要谈到ip盒子，frp这样的东西，因为一个mineportal必须首先是个mineaccess serverbox，access server的作用在于，它可以为应用准备一个access point和私网路由接入逻辑，使mineportal变真正的内网盒子。内网云。ip盒子的微道维系的口号是私人云接入商。而oray的蒲公英更接近这个概念。蒲公英就是一个带oray官方vpn线路和组网逻辑的路由器，排除这个他就是普通路由器，围绕mineportal的大部分应用都是内网应用，大都基于局域网ip ，如群晖的摄像头，如控制开机，如监控，或许我们还需要远程访问，但并不需要一个公网对外服务。这个VPN就可以。蒲公英X3附送的免费组网，一台路由器下可挂3个成员可以是路由器也可以是普通客户端，Oray的产品刚好弥补了一个类阿里云ecs云主机除了服务器本身之外所需要的远控，域名，外网服务，但实际上有了蒲公英，我们并不需要花生壳，向日葵，局域网控制也可用纯软件实现。。

除此之外，程序员还需要一个特殊的代码终端：

programming pad:

普通的PC太大，现在的手机全是触屏，实际上有时输入并不自然。程序员在地铁上这些碎片化场合无论用PC还是mobile，平板，都是极为不便的，以前有人试过有pocketchip，gemini pda，gpdwin,一号本，onebook我也试过，但是甚至gemini pad,gpdwin也是不好的方案，除了7寸键盘本身有点大之外，gemini的键盘设计得很好，然而实际上跟gpdwin一样它设置成桌面式放置并不是长时间手持防累的，这是smartphone和PDA的领域。而后二者要么没有实体键盘，要么尺寸不够6寸----对，我觉得能快速输入用的手机最大只能是6寸的而并非7寸的。

我最终选定的是黑莓的keyone,keyone2,priv实体键盘安卓系列。

实际上这样一个programming pad对硬件有要求，对软件也有假设和要求。

devops下的programmingpad ide
-----

如果选定在一台6寸物理键盘的手机上，那么打造programming pad的最终的想法。除了与我在前面讲到的一系列其它软件上的选型相关，它应该还应有以下几个考虑：

programmingpad终端的OS不要求是windows，不要求是chromebook，可以是单andriod。它只要装上普通的APP端与远端的服务端设施配合工作即可。这个服务端必须是一个devops，它负责反聩程序员高速输入和调试源码的远端反应，谈到devops，我们在《docker as engitor》讲过，在《群晖docker上安装gitlab as mydockerbox,mysnippterhosting》时我们谈到安装了gitlab的群晖dockerbox也是devbox,因为它支持devops。其实本地也有devops,像cpp的sandstorm用的那套ekam，https://github.com/capnproto/ekam，甚至在jupyter中也有，如基于jupyterhub多人版的jupyter实现的mybinder.org中，利用了docker打造了一个ide和类似gitlab webide的环境。

这一切都是因为devops的基础，都是engitor需要讲到，需要以此为基础构建的东西，所以该有的，除了他们各自内部的不同，其它他们该有的，都会有。

然而就是上面谈到的那个APP的要求了，它显而易见应是个liveide，然而与普通的liveide相比，它应该也是特制版的，比如它要受屏幕限制，因为工作面积小，碎片化时间短，它还要更“智能”，比如像个对话机器人式反馈结果，当然通常情况下，它只是一个配备了vim的更好用的代码编辑器，或一个更形象可观的可视化的IDE，更或者，他们的组合 ---  比如像ellie一样的就好，见我以前文章。

设想一下，如果是jupyter devops下的ide，它应该是mybinder.org一样，有源码归类的树形目录，有远程docker连接调试，如果是gitlab下的，就是那个web ide。还可以像ellie一样分成三屏，在手机上，或6寸umpc上，滑动卷屏，随时查看源码，html显示，程序运行demo和debug。。。

-----------

全linux的组合可以作为生产工具（一个nas,几个终端）。对于封装的个人云应用，我们交付的服务端，以及是终端。什么是封装的私人应用呢？那就是有一个mineaccessbox的，以mineportalbox为运行环境，可以programmingpad终端上运行其APP且调试的组合。

关注我，关注"shaolonglee公号"



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338630/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



