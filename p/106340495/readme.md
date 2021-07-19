Plan9:一个从0开始考虑分布式，分布appmodel的os设计
=====

__本文关键字：plan9,Inferno,limbo,Plan 9 from User Space:plan9port__

在《除了UNIX，我们真的有可选的第二开源操作系统吗？》中，我们讲到那些传统的os之争是集中于游戏好不好支持，桌面好不好体验，发行够不够流行，总体好不好用这些方面。而从x86 cpu从0开始的抽象全栈，他们都是一样的 -------  换言之，某种意义上他们都是一样的OS。

这种共同点在于哪里呢？对于最终的APP和APP开发来说，它们都是基于单PC+单PC下网络程序设计的。于是在这种架构下有了我们现在处处见到的web,云，—— 我们发现，当今的集群和分布式是放在云这个普化架构来做的：集群就是好多好多的PC通过网络计算起来，附带一些PC监控节点 —— 它们还是PC，在每一台PC内部运行的APP，都是从socket开始（更抽象一点，也许还有DCOM，消息件）的“云”程序:这种程序其实还是网络程序，这种总架构下的APPDEV，以OS来看，其实本质都是单机环境下网络交互的程序。

而这些都不是究极的分布式和分布式开发设计。

在《一种开发发布合一，语言问题合一的shell programming式应用开发设想》中，我们讲到了对于任何programming的设想，其实都是一个四栈从0开始叠加的设计。每一个appmodel，都是从hardware从0抽象来的，OS是大件。——— 把这种源头尽早控制在OS的源头，则每一个OS和单机则都天然具有云属性。包括开发。就得到了创新的appmodel设计 — p9，它是一个统一问题，语言，平台的总设计:

> Plan 9不是一个很知名的作品，但是它的前身Unix是世人皆知的。而Plan 9是Unix的几位作者在AT&T职业生涯的一件巅峰之作，是被设计来超越Unix的。 
实际上，Plan 9在1992年第一次发布时，就同时实现了Google Docs、Dropbox、Github、Remote Desktop等目前很火爆的互联网产品的功能。 
Plan 9能做到这些，是因为它把所有内容都注册到一个称为9P的文件系统里。 
举个例子，一个Acme编辑器进程会对应9P中的一个目录acme——我们可以用9p ls acme命令看到这个目录；这个编辑器中的每个窗口对应一个子目录，而窗口标题，编辑内容分别是这个子目录里的文件——我们可以通过修改文件内容（比如通过调用一个shell script）来改变标题和编辑内容。 
因为9P是个分布式的文件系统（类似后来的Google GFS和Hadoop HDFS），所以不管用户身在何处（公司、家里、旅馆、咖啡馆）都能看到同一个文件系统。甚至可以在家里的电脑上修改办公室电脑上运行的一个ACME的某个窗口里的内容。或者回家之后，让家里的电脑上运行的ACME访问办公室电脑上的ACME对应的目录，就看到了和办公室电脑上同样的界面——比远程桌面加上Dropbox更加远程桌面和Dropbox。
> Plan9没有推广起来，一个原因是它的思想太过领先——在用户还没有意识到存在这样的问题的时候，就把问题解决了。

9p，every problem/app is file io,这也是我们在《bcxszy series》中一直在寻求的分布式方案。

plan9的曲线回归
-----

开发是源于平台到语言到问题的总工程，由设计贯穿，设计包括对OS的设计，OS下编程本身的设计，OS下编程语言的设计，编程方法的设计。—— 所以，先哲学后理论再实现的思路没有错。

9p下的语言。也异于常类。它使用专有的语言limbo作为app langsys，使用c作为toolchain。

> Inferno操作系统是Plan9的姐妹操作系统。它的思想和Plan9基本相同，都是基于文件的。但是它只有内核是C编写，其他的应用程序都是Limbo编写的。所以它和Plan9不同的地方就是在这个系统上运行的程序都是Limbo程序而不是C或C衍生程序了。后来Rob Pike又开发出的Go语言有一些地方的思想就是借鉴于Limbo语言。

Go是limbo语言在linux的再生者,Go 语言的实现带有9p的深重痕迹，即使在x86上，也使用Plan 9的汇编器,为了实现所谓的语言自举，硬是绕开glibc去自己用汇编封装linux系统调用。可见plan9一开始就想彻头彻尾的自立门户，对传统OS和GNU没有任何依赖。

虽然历史上都选择了C family as toolchain和unix as os，没有选择9p和limbo，go，然而这不是9p的错。是工业和市场的错。

plan9 under linux
-----

虽然历史上都选择了C family和unix，没有选择9p，但9p可以是一种附加而不是替代。bell labs的9p是主，其支流也有一些。在linux下使用9p的方案，有Plan 9 from User Space:plan9port

或许我们以后在新ebcolinux rootfs 设计中编译plan9port，我们用plan9 under linux，terracing for go



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340495/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



