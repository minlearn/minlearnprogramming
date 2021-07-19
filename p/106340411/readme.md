一个隔开hal与langsys，用户态专用OS的设想
=====

__本文关键字：efi based os，native hosting oriented OS与  APP hosting oriented OS,将OS编程与硬件编程独立，将用户OS变为真正的APP空间。新dbcolinux和goblinux设计。__

在《一个设想基于colinux,the user mode osxaas for both realhw and langsys》中，我们开始提到了一种用特定host/guest OS组合同时作装机和作app hosting for langsys的思路，主OS用来装机，guest os作用户空间的通用程序托管runtime/移殖层子系统/app容器/特定语言系统面向的开发运行langsys backend baas，—— 这种思路实际上是最初级的用双系统方案来解决分离传统OS和APP容器OS的思路。因为传统的docker式独霸业界的方式我始终无法完全接受，我想找到一个能用于本地实地远程装机运维，能用于开发和集成devops的虚拟化层面，这种层面要能用于legacy native方式，要以科学的集成方式在它应处在的层次，而不是现在docker和裸金属这样，高阶，仅用于云。

鉴于此，我一直在找这方面的案例和整合方法，一些努力在我们的后来的文章中被频繁提到，如在《兼容多OS or 融合多OS？打造实用的基于osxsystembase的融合OS管理器》《一种追求高度融合，包容软硬方案的云主机集群，云OS和云APP的架构全设计》《去windows去PC，打造for程序员的碎片化programming pad硬件选型》中我们提到了其用于不同硬件平台融合和移殖子系统的用途部分，在《群晖+DOCKER，一个更好的DEVOPS+WEBOS云平台及综合云OS选型》《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》中我们谈到了其用于appcontainer,devops和云架构，appmodel的部分。在《打造一个Applevel虚拟化，内置plan9的rootfs:goblin（1）》中，我们将其用到了具体APP（将plan9 rootfs集进app，treats os logic and app logic together，把file作为APP的逻辑）层。—— 以上其实都是结合使用二种one host/one guest或多种one host/multiple guests的架构，使之职责分离成硬件OS（传统OS）和APPOS的职责的具体过程，算是开头提到的那个初级思路的延伸。

最后，我在《DISKBIOS：统一实机云主机装机的虚拟机管理器方案设想》,《DISKBIOS：一个统一的混合OS容器和应用容器实现的方案设想（2）》希望将它整合成一个PE层。于是用了diskbios这个用词：它是从bios开始做起的。是通用高级BIOS。最后，我们在《一种追求高度融合，包容软硬方案的云主机集群，云OS和云APP的架构全设计》《一个matepc,mateos,mateapp的goblinux融合体系设计》，《ubuntu touch: deepin pc os和deepin mobile os的天然融合》中提到了它用于第二PC的例子，—— 这都是选型研案，在整个《XAAS:the final all in one os》实践部分，我们在一步一步实现这样的一个系统：dbcolinux+goblinux。

新DBcolinux方向：dbcolinux based on efi
-----

而如今，这种整合思路有了新的方向。

因为在《一个统一的bootloader efi设想：免PE，同时引导多个系统》中我们谈到这项工作可以做成EFI。那篇文章中我们还谈到EFI实际上可以发展为pe的替代，比如它可以发展内存管理相当于一个OS代替PE OS，更复杂化还可以集成hypervisor，这样做的好处是可以在一台PC上并行boot主OS和host os。同时作主PC和第二PC也可以自由分离。且可以用于云。同时不再需要另外的recovery。

其实在这个架构中，已经没有了HOST OS，所有的OS都是平等的。而且它还可以有另外一个天然的附带作用。—— 如果hypervisor足够简小。它可以以足够轻量化的方式本身成为一个可用于OS也可用于APP的架构。这样结合我们上面谈到的《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》《打造一个Applevel虚拟化，内置plan9的rootfs:goblin（1）》。我们完全可以集进诸如hyperkit所用的虚拟机xhyve。这样，移殖层，融合，容器，applangsys backend baas,都可以以更简单和自然的方式达到了。

我们当然也可以组建传统的host/guest,我们也可以发展一个主OS。在这个主OS里组建复杂的OS生态系统。如vpnkit,datakit那样。做复杂的OS或容器集群。

新作用:goblin based on efi
-----

还有一个更重要的特色，由于所有的OS都是用户态OS了。它分离了传统OS的职责。hal，mm等层面可以完全从用户态OS中分离出去，保留在那个efi os中。用户态的OS都是单一职责的app hosting os，这样的OS就是直接面向APP的了，可以叫APP OS。这样就降低了APP开发者的学习难度。也降低了通用编程学习新手的入阶难度— 因为它们从此不用再学习任何系统知识，处理任何系统编程。在APP os层面可以专心与一门语言绑定，或专心融入一种问题，或抽象方法，如plan9 all is file的理念，变成纯粹面向domain的专用OS。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340411/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




