一种设想：为linux建立一个微内核,融合OS内核与语言runtime设想
=====

__本文关键字：os之争。微内核,language based os,language on bearmetal not on os，华为鸿蒙，语言即OS，类脚本语言，把原生应用变语言模块。__

我们知道，OS兼容跟语言兼容一样，是一项几乎不太可能完成的事情，因为OS的使命就是作为闭环竞争的商业产品出现，成就造出它他们的公司。从文件系统的多样化和存在的互访鸿沟就知道，见《一个设想：基于colinux，去厚重虚拟化，共盘直接文件系统安装运行的windows,linux》。osx有apfs,windows有ntfs，linux有btrfs/ext，难得出一个exfat，又仅是作为数据盘存储格式不能作为系统区存在而且也有坑（不过也听说过在exfat上成功装系统的）。os之争与语言之争一样，技术上不是不可能融合，而是厂商各自为政。不想那么去做。

但幸运的是，软件是抽象的堆栈，业界方面这些融合一直在进行，只是比较缓慢。融合可以从源头(对于os是kernel)完成，还可以在别的层次完成，类似《编程语言选型之技法融合，与领域融合的那些套路》谈到的那些语言融合。

OS/OS内核/OS的融合经过了很多发展阶和变形形式，如日常我们使用的PD,VB虚拟机管理器里的os，wsl( as os subsystem)，虚拟机管理器on baremetal(qubeos,etc..)，虚拟firmware内嵌虚拟机管理器，os级的虚拟化openvz，简单的chroot沙盒os，iaas大数据级别的kvm，最后，就是这篇文章要谈到的：从0开始，和从os kernel源头进行融合OS和语言系统的os。即“微内核+unikernel”。

> OS的融合其实和语言/语言runtime的融合其实可以是一个相关的过程，如《基于colinux的metaos for realhw，langsys和一体user mode xaas》如《一个隔开hal与langsys，用户态专用OS的设想》中关于让os的子系统直接服务于语言runtime的思想。我们在《云APP，virtual appliance：unikernel与微运行时的绝配,统一本地/分布式语言与开发设想》中也讲到语言的后端和容器和OS的关系(通用容器和语言虚拟机/语言沙盒/baas/serverless，这些东西的存在，其实是”可移殖可隔离runtime“的功能重合。正是面向不同方向但同一目的的某些意义上的“重造轮子”。)


安全的内核级语言：unikernel化
-----

os与语言的关系从来不是先有蛋还是先有鸡，而是先有蛋（语言），再有OS（鸡），自举语言和自举OS特殊情况另当别论，历史上和我们现实中出现过的常见OS，其kernel都是C开发的（我仅指osx,linux,windows kernel），，c runtime作为toolchain系统实现层面存在。c库存在用户空间和程序空间的部分是有特权不同的，它们之间要跨syscall互访。应用开发层次的语言系统处在用户空间(由此,官方py版本这种语言是不可能作内核编程的，抽象层次太高运行时也巨大)。我们把内核中的c称为“bearmetal或toolchain”语言，因为它们最开始的作用是给硬件平台提供软件管理层的作用。

由于语言放在OS之前设计（没办法，先得用起来，那个时候还没有出现既能保证开发效率运行效率又能保证安全的语言如rust），架构于os kernel之上。于是kernel的二大机制（内存管理和任务进程）会继承C语言的固有缺点，比如运行在这种OS上的程序会发生内存泄漏，这对于系统实现和APPDEV级都是延续的影响。我们现在的OS，如果它们自我宣称它们是“安全的OS”，其实是某些高阶层次的完全，比如OS应用方面的安全。并不是“内存管理和任务进程”方面的绝对安全。运行在这种OS上的APP还是会发生内存泄漏，因为“程序申请内存”这个功能，无论处在编程的哪个层次，都是从OS继承的能力（任何编程都是某种意义上的系统编程）。GC只是自动释放并不能改变OS内核这种可能的缺陷（这只是一种可能，OS内核的质量会保证这种情况很少出现）。

那么，为什么不采用一种从一开始就足够安全的bearmetal语言呢，如rust？

当然可以，语言放在OS之前设计可以直接省掉OS发明的很多问题，如cpu的保护模式甚至都可以略去，特权模式无关紧要，一种内核天然是bearmetal和usermode通用的（驱动和hal层放在用户空间）。你甚至还可以将rust的stdlib整个实现在bearmetal层,使得内核编程跟系统开发编程/系统应用编程共享同一门语言。

这些特点加起来，可以直接用语言作为虚拟机管理器（这种语言系统和OS最绝佳的关系，正是另一种os kernel-libos的特点：它可作为lib的os被嵌入语言使用，用户态无须经过虚拟机管理器。直接为支持“一app一os”而提出的os）。因为你可以把语言系统作为OS的源。直接compile and run一个kernel出来，使得语言架构于OS之上，做成devable os。在《云APP，virtual appliance：unikernel与微运行时的绝配,统一本地/分布式语言与开发设想》中我们提到unikernel是docker等lanauge runtime的代替品。这里是“more safer unikernel”。

当然，为促成microkernel+unikernel，这还需要一些额外工作。比如，将这种内核裁剪下，去掉“不那么重要的部分”。


采用了安全的语言，我们还要给内核“微内核化”
-----

首先，这种内核要小。仅保留必要的“内存管理和任务管理”。如上，我们知道，内核中最关键的是“内存管理和进程管理”,因为这二个部件，足够可以运行某种“APP”(内存和并发，刚好它们也是语言runtime的重大问题之一，所以这里“语言runtime即OS，这种特征又一次出现了”），除此之外都是addon,非essential的。hal,drives，文件系统,gui都是第二级别的。我们可以把这种OS的“驱动”保障效率的前提下放在user space（如上，实际上，这种OS kernel下，没有内核空间和用户空间的区分了）。传统上，我们将hal和driver一部分放在(firmware或boot层)，一部分放在os kernel层。

这就是微内核。OS层的其它组件可以通过ipc通讯放到其它具体os kernel下，它本身可以很小（作为meta os kernel与其它内核合作，管理这些内核并融入这些内核,这也使这些内核可以实现真正的跨OS）。比如给linux kernel,这种monolitch一体化宏内核作上层，复用它里面的东西。

综合上的microkernel和unikernel，如果融入现在的linux，比"用虚拟firmware内嵌虚拟机管理器打造applevel虚拟化的容器和开发方案"这样的办法要强多了。


-------

(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108455957/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>






