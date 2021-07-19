DISKBIOS：一个统一的实机装机和云主机装机的虚拟机管理器方案设想
=====

__本文关键字：用hypervisor装机，用虚拟机管理程序代替winpe，带VNC的装机环境,单机虚拟化方案__

在前面xaas系列文章中我们涉及到多种不同的虚拟机/虚拟化技术和装机技术,比如在《发布virtiope》时，我们谈到云主机装机，那是在kvm的guest os里装机。比如在《发布colinux代替os subsystem》中我们谈到host/guest os技术,《发布tinycolinux代替docker》中我们比较了tinycolinux与boot2docker中的iso linux，甚至在《免sandstorm的davros》时我们谈到sandstorm其实也使用了一部分xaas虚拟技术。

其实虚拟机/虚拟化技术/虚拟机管理有多种，最常见的就是virtualbox,vmware这种工具级和应用级的虚拟方案（多用于开发机环境），它属于纯粹的面向单机内的虚拟方案。甚至有WIN10 WSL之类的东西其实就是二个OSSUBSYSTEM同时并列被实现在WINDOWS体内,当然colinux也属这类，它能在一个host os中装多个guest os,只是多用于服务器环境或NAS类环境 ---- 当然，如果严格说起来，它们可以组虚拟机集群只是不被提倡这样做。

集群下的虚拟机技术有kvm,vps这种，KVM是一种hypervisor，即真正的大型iaas用的硬件支持虚拟下的虚拟机,它介于物理机和os之间。有全虚拟化和半虚拟化如kvm,hyperV,xen等等，搭配openstack这种软件可以做iaas。用于运营和云计算环境组建分布式集群。一般非常巨大。vs kvm, openvz这种虚拟OS较轻量，它走的是常规的OS级的虚拟方案----面向在单机OS内部虚拟出多个OS，这符合我们对虚拟机的设想，且它的这种能力也使之非常灵活，可用于多种架构也可同时用于分布式和单机环境下的虚拟。

虚拟化技术要么从使一台机器或一个分布式架构增加虚拟化隔离的能力分裂出子OS或从OS出发，要么从有限的OS子组件虚拟分裂其计算能力。因此也有像docker这样的方案，它有来自内核调用的支持，它基本倾向于用户层虚拟化。它可以做到纯粹的文件系统虚拟化，像一些沙盒环境和绿色软件做到的那样，这是虚拟机中轻量的一种：虚拟容器。

在以上由管理器虚拟出OS的虚拟机技术中，被管理的os有的叫guestos，有的叫virtualos，有的叫容器OS，（你可以完全不用纠结这类说法，只要明白它们所处的明显不同的阶层就好了），而管理器往往以元OS的身份存在和被实现，它发挥的是一种管理硬件资源的能力-----相当于bios管理程序，只不过它面向装各种虚拟OS，且它本身往往采用的是与虚拟OS同样的OS技术:liveos，所以它又像WINPE这样的东西，这给了我们一个新思路：它可用于装机，如果这种管理程序用在给单机实现装机的话，可以说是除了BIOS之外的OS的OS，我们将它称为DISKBIOS。

比如OPENVZ，由于它足够轻量且可以被工具化。那么当它用于普通个人装机，不妨可以一试，这样它就兼具运营目的和实用目的了:比如，个人不但可以装小鸡出售，甚至可以仅用它实现一台机器同时运行多OS，多种不同的OS，且实现远程WEB VNC管理自己家里的那台机器。

来讨论一下可行性吧：

提出一个live os as winpe
-----

谈到支持虚拟化的内核，一般就是patch过的kernel，而guest kernel可以复用同样的内核技术，且各种具体虚拟OS层的OS模板可复用同样的linux打包发行技术，OS上套OS是虚拟化技术中最省事的。也有OSv这种重新发明了内核的(据说它每个用户一个VM)。

传统上OS分开系统空间和应用空间，先OS再各种应用，现在xaas时代，OS kernel只不过是我们组建复杂虚拟OS集的最小单元，至于各种应用。。当然处在虚拟OS中，作为元的hypervisor OS中只运行虚拟机管理程序：负责虚拟OS的创建，暂停，停止，资源分配，甚至更多事情，比如远程可视监视VNC，ETC。。

anyway，我们要谈到的实机装机也完全可以是通用最简单的OS套OS的思路：提出一个元liveos，在其中装一个虚拟机管理器用于分配和引导虚拟OS。hypervisor os和各种虚拟OS都采用tinycolinux，linux内核保证能编译支持多种最新MAS设备的能力使之能极好地代替WINPE作这种live装机环境，开机时可以选择让这些虚拟OS同时运行（服务器环境时），或开机装仅运行一个（实机装机时）。。

采用tinycolinux是为了保证元live os的轻量，毕竟装机环境不能太大，要注意host liveos 和guest virtual os是一直在线运行的，这个live os是一直在线管理的，只有guest os是可以选择性关闭，重启的。这是与winpe等装机环境离线运行装好的OS不同的地方。不过这也是它的优点：元管理OS常驻可以实现在线装机。

在liveos中配合虚拟机管理程序用于实机在线装机/维护
-----

虚拟机管理器则用openvz来做向linux patch虚拟化支持。再加上vzctl命令行工具或WEB管理器openvz panel等工具（如上所说机器一直开着元管理常驻就可以远程管理且装机/像ghost一样恢复etc..），各种IP池可用端口转发加虚拟网卡驱动来实现。

甚至我们可以增强它，使之还包含paas的东西，，比如像sandstorm一样虚拟出app,app-grains,etc..将资源管理更细粒化，真正极度服务于运营目的。

当然，如果是面向实机，实际上一个虚拟OS独立所有可用资源的情况下，不必这样做，直接装好了即可。

-------------

下一文可能就是用在《tinycolinux编译openvz内核》，然后制种一个live tinycolinux，，，再制种一个tinycolinux guest os的文章了。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340244/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




