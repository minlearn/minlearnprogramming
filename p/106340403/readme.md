一个设想:基于colinux,the user mode osxaas for both realhwlangsys
=====

__关键字：umwinlinux,从文件夹中启动的linux,user mode linux windows,iaas,baas,paas穿插开发运行环镜，是原生装机系统，还是语言系统后端虚拟机，实机/虚拟机/os内部 统一操作系统。真正的应用程序级统一的user mode OS，用户态操作系统。用户态操作系统内核。__

自古以来，像python,js,php这类动态脚本语言系统都严重依赖于后端虚拟机实现，毕竟，可移殖性是soft vm的重大作用之一，这使得基于其上的开发和发布可以做到伪“跨平台”（实际上是各大虚拟机在其上都实现了一遍），更有甚者，.net和java这些虚拟机更是提出了统一后端，使得常见的多语言系统有了共同的后端规范，基本上可以将包括上面这些语言在内的各大各自为政的语言整合到all in one和极致，比如ironpy,ironjs,ironphp based on clr。 —— 所有这些，不过是把不同OS本地上的各异性封装了一次，用软件再造一层抽象，有了统一的接口再在其中建自己的东西，这里的抽象与封装过程作为基本技术，在软件技术/艺术的各个层次频频可见。

但是遗憾的是，类似技术并没有上提到OS层，，OS作为规范硬件各异性并提供native dev and run的统一层面，与上面提到的python,js,php等langsys backend soft vm有异曲同工之妙，然而正如它们没有进一步发展为.net上的免binding ironpy,ironjs,ironphp一样，各种OS上面实现的APP规范和系统调用（各种os subsystems,etc ..）实际上是各自为政的，不可移殖的，unix posix与windows win32/64实际并不兼容，所以才会有各种CUI级别的cygwin,wine和各种用户层虚拟机方案（接下来会讲到）的出现，它们的工作正是为了统一这个层面。前者主要是为了运行程序，后者主要是为了开发/iaas化加速。

有几种特殊的OS：
 
现在的云各种虚拟OS的加速方案kvm,virtio,openvz等，还有各种虚拟机管理器virtualbox技术，还有vagrant开发虚拟机，特别是vagrant,随着开发的复杂化，建立paas,baas,iaas的穿插环境，在vagrant中建立起各种虚拟机环境，这种需求都开始变得很明显和频繁。
 
这些技术的出现，都可以称作是一种user mode os的层次的东西。而jvm,clr这样的规范和实现，一开始也都是工作在用户层的。有相同的架构层次和整合基础。

separated user mode os from kernel mode os
-----

虽然OS这个层面并不需要直接考虑任何langsys，但是不妨这样想，在OS的实现层，如果有一套类langsys的soft“虚拟机”，用来代替os的subsystem（传统意义上的os subsystem api兼容机制并不足以提供太多的东西。），那么可以实际上有二种 OS:

一种OS是那些kernel的东西就足够，并不需要包括cui层次，只包括driver实现层次，是仅仅管理内核层次的OS。

而另一种OS不直接附在硬件上而是作为一个vm存在，专门用来负责除硬件虚拟化之外的其它任何应用兼容和开发层任务，就像jvm,clr，安卓内部的java虚拟机一样。

而第二种OS实际上可以完全采用第一种OS的kernel实现技术，只不过它全程运行在user mode下。以第一 种OS为meta os，后面第二种OS的实例可以在资源限制范围内无限开。 ——- 这完全类似于文章开头就谈到的：在langsys层提出clr,jvm，用它来建立起isolated langsyses的统一后端，达成最大兼容和可移殖。

业界类似方案有colinux，也类似openvz这类方案，都有二套OS内核。二者兼备才能跨内核和跨用户层都能做到高度统一，比如，通过colinux等usermodeos,实机OS可仅作metaos，而user os可以作各种虚拟层.

为什么是colinux？
 
以上这些技术在colinux中全被包含,它符合浅封装原生OS和不带来太多损耗的原则。
colinux实际上是user mode linux的一种，不过它是建立在以windows/linux为host上的只是不能以windows为guest。并且，它支持从某个实机盘和实机驱动。
 
为什么不是虚拟机管理软件和reactos这样的方案？
 
虚拟机太重。
reactos太注重重复windows已经做过的工作，忽略了新时代user mode os的需求。
拿龙井Longene来说，龙井也放弃了驱动兼容（它在最新一份解说中，提到类linux的andriod系已变成主流，windows才是需要兼容linux的，所以不必兼容）。毕竟多OS共存才是合理现象。本来就不必从那个层次兼容。弄错了兼容层次。它现在专心做应用兼容了。
他们都用了wine,希望ros能改正过来，向龙井靠拢，将roadmap改成先发展和完善user mode的东西，再有余力去发展驱动兼容级的实现装机方向。
 

这样的二套OS可以装在实机上，当用在实机上，用户可以在任意架构的机器上同时（注意这个同时）安装运行多种操作系统并不需要安装额外的驱动，性能并不会有太大的损耗，在装机上可彻底去除UEFI这样的东西，机器出厂商仅需要集成第一层OS及驱动支持即可 — 向用户透露更好的直接装应用的层次。

user mode os不但可用于装机，还用于语言系统和开发运行
-----

再来说第二层OS用于作为langsys backend，如果第一层OS可以支持任何类型的第二层虚拟机，那么几乎历史上所有的APP开发/运行层兼容问题都不复存在了。比如，一台手机的实机OS集成了三个第二层OS虚拟机，那么它就可以同时运行winphone app,ios app,andriod app，还可以适用于云主机虚拟机，还可以用于任何CUI层的vagrant开虚拟机过程。

看来，提出一个先行于realhw os和考虑整合langsys后端的用户态OS，看来是潮流啊。。

xaas:大一统的user mode os
-----

当第二层OS可以以vagrant方式被管理和使用时，它实际上变成了xaas，因为它可以为langsys baas服务了。

在我以前的文章《发布engitor》《发布enginx》中，实际上enginx +engitor是当paas和langsys baas用的，尤其是engitor还有engitor as visual editor service的意思，可以看作eaae service吧。

如果这些都可以做进我的msyscuione->xaas文件夹与engitor,enginx放一起，（比如msyscuione中可直接安装不同第二层user mode OS，比如colinux,再在colinux中装msyscuione langsys），而不用到像vagrant之类太多虚拟技术和云技术。统一用于实机，云端装机/开发。那么这个user mode os就是不折不扣的跨iaas,paas,baas等的综合xaas user mode os了。

或许还要加上我《一个设想：基于colinux，去厚重虚拟化，共盘直接文件系统安装运行的windows,linux》就更完美了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340403/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



