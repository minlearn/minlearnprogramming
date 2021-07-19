兼容多OS or 融合多OS？打造实用的基于osxsystembase的融合OS管理器
=====

__本文关键字：兼容OS。__

相信兼容多os一直是人们的一个梦想，因为在一台机器上跑多个OS是很客观的需求，不光开发，有时一般办公生产都会涉及到在单机上开多个OS的需求。这种东西不光要能用，而且要求要“好用”。我们在前面多次谈到这些。如《reactos》,《colinux,去虚拟化一种文件系统共享的多OS设想》，《dbcolinux利用虚拟机管理器装机》，etc。。

在兼容多系统的发展道路上，有colinux这样的东西，也有wsl这样的东西，有龙井linux这样的东西，还有fydeos这样的东西。也有exsi这样的东西，还有虚拟机，docker，群晖vmm这样的东西，更有虚拟机中的osx parallesdesk这样的特殊品种：”融合os“，当然还有很多。。。

兼容多OS的分类
-----

一般地，附在这些实体，虚拟架构上的多OS，有时有二个，互成host/guest，这是最典型的情况,有时有多个。但基本都有一个管理性的OS，或虚拟机管理程序，或hypervisor,为方便讨论，我们将其区别称为（管理性OS和用户性OS）因为如果视管理虚拟机本身所在的OS也往往是一种独立OS，是“OS的OS”，用户OS主要是子系统的话，那么其中运行的子OS是平等的。用户主要使用的，就是运行的各种子OS。------ 如果管理性OS和使用者OS都是主体，就是前一种，如果使用者OS是主体（仅对用户可见），就是后一种。 ——— 这样讨论就方便多了，所以首先，“兼容多OS”基本可以按这二类归类：

1，从平台和架构上来，有的是面向实机，裸金属装机，如exsi，独占机器，有的是面向计算意义的。除exsi之外的都算。他们只占据该计算架构中的某块资源。如云主机架构中的各种OS。有的是硬件虚拟化，（有的是hypervisor，受硬件支持 ,有的是OS级的虚拟化。只要OS提供了分裂子OS的功能，就可以在一台机器上跑多个OS)有的是工具层面的，如各种虚拟机程序vmware,virtualbox,etc..。

2，从相互归属性上来讲，colinux,虚拟机oss,fydeos+3 oss,docker subos都有鲜明的host/guest特性，因为我们平时不但要管理这些子OS，也在主OS中工作，同时接确这二者，这种主要是双OS，而exsi，vmm,云openstack是一类，我们基本，或很少，不能、或无须接确到上层。我们把后者称为平等多OS。

来深入继续归类：

从性质和技术来分，有的是经典的内核增强技术。如龙井,wsl,colinux，都是从严肃的内核改造/再造开始，而vmm,docker,fydeos这样的东西，虽然有内核定制，但都是轻量定制了的内核加多样化了的虚拟管理程序，和不同的rootfs为主。而虚拟机则纯是一种软件层级的再造方案+（可选的硬件直通能力）。

从生态上来分，hypervisor类的多OS往往有多种不同的OS。而OS级的虚拟化，往往都是一种OS的变种，互通性容易些。

最最重要的分类问题来了：

在日常工作中。那些我们平时需要频繁用到多OS的情况中，哪些分类是起决定作用的。即“好不好用”这个最终问题，才是决定用户选择的方案分类问题，这个分类问题就是性能和最终体验问题：

但事实是，我们目前很难找到一种保持原生OS体验，性能不打大折扣的方案：从性能上来讲，虚拟机是最经不过考究的，即使加了硬件直通性能也大打折扣，wsl，colinux这种基本CPU能力都能直通host的次之。从体验上来讲，也许只有parallesdesk最好用（也许融合，不打破才是最省事最好用的目前方案，虽然其它的方案也在进行中。。但目前唯有PD最实用），wsl次之，其它也都是半成品/实验品，可是PD它只存在于osx，而且收费。

那么，可不可以将PD做成类exsi的东西呢？

设想：打造基于osx base system的实用windows/osx融合OS管理器
-----

这种方案就像在winpe上集成virtualbox或vmware一样，然而选择osx是因为PD在osx上的体验最好。我们可以定制osx base system，将PD装在osxpe上，然后开机启动。在其中安装并列的osx和windows。这二者的互通和融合通过主OSXpe中集成的finder和工具栏docker来进行，必须保证osx base system的管理者OS身份。因为它足够小。

或许你可以按《OS X Recovery Partition: Customizing With Different Apps》进行得到这样的一个东西。如果还能将它硬件化到nano itx小主机就爽了，做成黑苹果更通用。

————

从《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制》系列到《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式》，再到《打造一个Applevel虚拟化，内置plan9的rootfs:goblin》，我们贯彻的选型依据之一，就是：哪个兼容多OS方案是最好用的。最后由于实现的需要。我们不可能采用PD这种，最终屈服于从linux加OS虚拟化级去定制，而且我们需要更偏向开发相关，而不纯粹是装机的那些问题，所以就有了goblin。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340294/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



