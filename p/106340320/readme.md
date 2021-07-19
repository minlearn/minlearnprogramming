为你的硬件自动化统一构建root和firmware
=====

__本文关键字:Buildroot，coreboot as Buildrom__

前面《基于虚拟机的devops套件及把dbcolinux导出为虚拟机和docker格式》系列文章中，参照LFS,我们曾手动自建过交叉编译环境和linux，完成了包括gcc as 开源toolchain,linux as os kernel，busybox as开源os rootfs,各种libs的所有手工化构建步骤，不过仅限于在PC上，和x86之间进行，都没有考虑进我们现在除PC外的其它使用linux的硬件，—— 这就是嵌入式平台及特化PC，它们甚至是我们目前硬件平台中的绝大部分，如路由器和手机(nas,laptop也算)，—— 虽然说linux对嵌入式都有支持，构建的流程大同小异。它们跟PC平台相比，对构建的需求还是有点不太一样：

首先，但嵌入式硬件多样，有着各种各样的非x86 CPU和cross compile需求，采用的一些版本也是PC上的特化版本，如libc。libc是程序与内核交流的媒介与用户编程接口，，嵌入式上往往使用ulibc.

而且，嵌入式中，firmware跟os,甚至app的界限非常不明显，往往高度耦合，前面我们在《Boot界的”开源os“ : coreboot，及再谈云OS和本地OS统一装机的融合》中说到，firmware编程是OS前面的那块软件编程，嵌入式平台的firmware与os往往高度偶合，往往firmware直接就是boot loader，使用uboot这种方案。程序要求也不一样，嵌入式往往就一到几个守护程序持续运行，简单的自启脚本即可，除了像带需要带安装APP支持的智能路由器需要像openwrt这种拥有类PC包管理的之外。

而且他们使用构建结果，安装系统的方式，PC跟嵌入式也不一样，它们需要rootfs尺寸要裁剪非常小，它往往是一个静态的firmware或rootfs镜像，直接刷到嵌入式硬件上的一块静态的固定存储量的ROM上用于承载OS。

因此针对嵌入式的构建，这些支持往往分别分散在各个厂商的手中。

—— 所以，如果有各种嵌入平台，还要考虑进coreboot as 开源firmware，，为什么不把for pc,for embededs，for rootfs,for firmware编译和安装统一考虑进这个综合构建呢？

—— 只是，因为这里面的要处理的工作和复杂度实在太高太大量（这已经是制作一个linux发行版，甚至硬件的工程量了），难于用一个框架来述说。你看LFS就需要一本书。这套构建工作最好是自动化的。要用一套脚本来处理的话就最好了。

—— 幸好，build firmware和build root其实是一个很相近的过程，都是从交叉编译工具的构建和驱动开始， (其实，区分这二者也很简单。一个是硬盘上的OS，一个是没进入OS之前的那个硬件初始化（firmware）中的OS)。这二者统一，自然而然。

市面上终于也出现有这类工具，Cross-ng和Build root，OpenEmbedded,openwrt，就是这样的工具/环镜。如cross-ng只处理交叉构建，buildroot就是整合toolchain（它也可以使用external toolchain built from cross-ng）,kernel,bb,rootfs,libs的工具,只限于直到生成打包的文件系统。不往发行版本一步，coreboot,firmware的OS，buildroot有点像coreboot as buildrom，。openembeded可以直接用来构建发行版。openwrt也跟openembeded一样生成发行版不过它大都针对路由器。

这里面最大的好处是，1，当这类工具统一在同一套脚本上，它可以实现all in one,self contained，所有的部件保持同一风味，比如linuxkernel,bb是同flavor的，它们使用kconfig，且将编译过程中所有涉及到的大件都menuconfig化，如果libc也可以，coreboot也可以，不是更好么。在build root内部就是这样做的，它增加了glibc-menuconfig,linux-menuconfig这样的东西。使得脚本更统一，更in flavor。2，both for embed and pc,buildroot本来就是for嵌入式的，像build root同时支持了ublic和glibc库和各种mainboards,cpu。这样就给了我们融合多套硬件制作同一套软件镜像上的统一构建用的codebase,，而coreboot也是for 嵌入式的，我们看到其soc下有很多欠入式,广大嵌入式的构建工作更应应用coreboot的统一设计而不是分散厂商的。— 这些可以用同一套codebase生成融合硬件使用的镜像不好么？。3，可以统一使用，其写好的firmware可以在虚拟机中，如qemu中测试（qemu因此也是firmware和build root眼中的一种抽象主板）。Qemu可以单独喂给一个firmware,也可以同时结合喂给一个firmware+OS的组合。4,可以统一让融合硬件使用同一个codebase，谈到硬件融合，我们前面有很多例子，虚拟appliance+一个有融合作用的HOST as shell也可以解决硬件融合的很多事情，比如PD的融合模式，而现在，像一些激进的硬进如purism,librem(一个可以发展装有linux的手机。类ubuntu touch一体手机电脑融合OS的硬件产品)都ship了Coreboot

带着这些观点，我们来研究build root和coreboot的脚本

其中的所有原理，都是我们在《基于虚拟机的devops套件及把dbcolinux导出为虚拟机和docker格式》曾手动自建交叉环境和编译之后kernel,bb，libs,所涉及到的那些，Coreboot和Buildroot的编译都能在一台unix派生系的host上完成，要求host上一个toolchain加少量必要的库就差不多了。因为它们都面向构建基础套件。

Build root
-----

Buildroot可以在本机上直接构建，官网提到GCC要4.4以上，HOST要用常见unix派生系os。。当然最新发布的buildroot已经用上了vagrant虚拟机，这样就省了好多工作(如果要在host手动安装编译需要的东西，参见那个vagrantfile中要安装的包即可，)。

我们仅讲解自动的。在vagrant中，我们发现最新的buildroot都使用到gcc7来编译了。buildroot(2019.8)大约有10000多个package(实现一个发行版直接可以拿来作为仓库),每个package下面都有.mk和.config.in，package源码都是一次编译即时下载到src/dl的。你也可以make source把它引用的packages源码下好，里面有board/qumu,生成的output中，output/中良好排序着交叉三元组：build, target, host中的临时文件，还有我们要得到的最终的东西，images，可以对比《基于虚拟机的devops套件及把dbcolinux导出为虚拟机和docker格式》来理解和使用。

这里需要搞清build root涉及到的devops工具的关系和区别：

vagrant和packer都是hashicorp的Devops七件套之一。packer是提供一个初始镜像，控制虚拟机如VB，PD（我们讲过PD和VMWARE方面的例子），自动化输入。inplace生成新镜像。vagrant是命令行版本的虚拟机管理器。它可以管理VB，PD等（provoders），如果有可用镜像box，可在vagrant配置文件中写明，然后用vagrant up起来给任何一个provider使用，up后可在虚拟机内干任何事情，关机后不导出镜像

它们的关系才是它们的重点，1，用packer也可以制作vagrant使用的镜像。2，Vagrant技术原理它跟packer差不多，里面也有provisioner等等。也是控制虚拟机，也有自动输入和配置文件，只是表现形式产出结果不一样。3，它们共享很多其它机制，如都可以在host/guest间用nat，Nat方式也可以在本地搭建一个也有http sharing。只是配置文件内的写法大同小异。
下面来谈在vagrant guest os中使用host的VPN。比如我的VPN在HOST的127.0.0.1:1087端口,虚拟机里面的是不可能获得本地vpn的，所以我们需要在vagrant files中。加一句。export http_proxy=http://10.0.2.2:1087;export https_proxy=http://10.0.2.2:1087;因为用的是nat 网络。

>>加在这个位置:
>>config.vm.provision 'shell', privileged: true, inline:"
>>export http_proxy=http://10.0.2.2:1087;export https_proxy=http://10.0.2.2:1087
>>sed -i 's|deb http://us.archive.ubuntu.com/ubuntu/|deb mirror://mirrors.ubuntu.com/mirrors.txt|g' >>/etc/apt/sources.list
>>...
>>config.vm.provision 'shell', privileged: false



如果不在vagrant虚拟机中开启VPN支持。可能会卡在Downloading and extracting buildroot 2019.08不动。这其实也是从前文《packer中学到的》，，可在打开的虚拟机窗口中测试box os用户名密码都是vagrant，ifconfig看到，在本机是访问不到nat内部的不能ssh vagrant@10.0.2.15。

Build Coreboot
-----

Coreboot最新版是coreboot-4.10,使用linux kernel式的kconfig，和bb式的可视裁剪配置。所以它跟build root是flavor的，也对qemu有board支持。。跟buildroot的职责一样，完全对引导期，进入OS前的firmware OS的构建。

Coreboot并没有使用vagrant，你得在一台HOST上配置再编译。也是配置toolchain(因为稍后Qemu x86，所以用any toolchain)，mainboard那些，不同的有payload，对于mainboard，如果是测试，可以完全选择使用Qemu 模拟x86.

之后make一路通过。

我们的改变
-----

如何集成到我们的cohdevbox srctree中。及你们的工程中。

上面提到vagrant和packer同基础，工作方式和配置都大同小异，在我们现在的实践文章中，都是用的packer，那么为什么不能把vagrant当packer用呢。Vagrant还可以避免出现packer那种一旦失败，全部从0重来的尴尬。即，在Vagrant退出按ctrlc,再重新执行up，它会重用上次grant provision的结果(因为它是虚拟机嘛)。这跟packer不一样，packer的构建不会重用上次provision的结果，会重新构建。虽然它不产生镜像（实际上那个镜像就是你的provider在你硬盘上的镜像，关机复制得到和packer一样的结果）。

我们会在srctree中提供一份编好的tinycorelinux硬盘镜像作为vagrant用的box。然后整个源码会deprecated packer。只用vagrant.然后在其中通过配置文件手动编译avatt，结合修改，代替现在的cohdevbox srctree。

最后，我们可能全程用qemu作虚拟机，因为qemu支持命令行提供firmware，它可以统一测试firmware和rootfs，完全可以用qemu来替换VirtualBox，qemu是命令行虚拟机工具。与其他的虚拟化程序如 VirtualBox 和 VMware 不同, qemu可以模拟CPU(全虚拟)，甚至喂给firmware。但vb,pd都不可以。还有一点，QEMU不提供管理虚拟机的GUI（运行虚拟机时出现的窗口除外），也不提供创建具有已保存设置的持久虚拟机的方法。除非您已创建自定义脚本以启动虚拟机，否则必须在每次启动时在命令行上指定运行虚拟机的所有参数。

我们选用的host平台是osx,不能用到kvm/qemu,因为kvm是linux的。在osx上我们不打算用到任何加速方案（半虚拟）。因为是测试。直接使用qemu默认的console窗口而不使用任何GUI管理器，vagrant可以通过libvirt用上qemu。

注意到coreboot编译GCC toolchain等输出信息很精简。而我们在《packer》文中不能控制选择过程warning等信息。所以决定研究后采用。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340320/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



