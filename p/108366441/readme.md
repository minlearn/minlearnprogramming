云APP，virtual appliance：unikernel与微运行时的绝配,统一本地/分布式语言与开发设想
====

__本文关键字：云时代没有软件，只有服务，虚拟app，虚拟OS，虚拟APP开发,metarootfs as service,container as service,virtual appliance，可devops编程os，Redox OS，融合app__

我们知道，OS的选型，其实关乎着开发，因为它位于开发四栈中的平台栈部分，（系统开发语言对”系统平台“进行开发）平台与语言产生联系的方式首先是支持该OS开发中的toolchain language中的runtime（kernel space或user space中的libc），它以OS为宿主，将系统服务做成API件，将系统实现系统开发接上语言和开发。

与之产生比对的，就是该OS上的各种应用开发层次上的开发语言的后端，比如脚本语言，它可以是各种port到该OS平台上的虚拟机平台作为runtime接入到os，这种软件"虚拟机os"+“虚拟机平台上的语言，包装来自原OS toolchain的API或OS可开发服务即可，照样可以作平台上的系统开发。（比如c,cpp虽然可以桌面应用开发，但是它有开发效率不快学习曲线问题和内存泄漏问题而桌面开发不要求指令密集且不能跨平台，那就可以完全可以另起灶炉发明语言和一整套开发系统换成py这些）。甚至最后，虚拟机语言和脚本更多面向WEB，因为这种“API服务可以来自不同OS不同进程。的虚拟平台，不是OS也可以“。，云时代没有软件，只有服务。在《编程语言选型之技法融合，与领域融合的那些套路》和《一种matecloudos的设想及一种单机复杂度的云mateapp及云开发设想》中，我们都谈到，传统OS和语言是面向nativedev的提供语言系统和开发API的。，对于web和分布式来说，因为API和语言都不需要来自这类平台。更适用这些虚拟机平台和服务开发。----- 平台，语言后端，这二者的界限模糊了，语言可对虚拟，脱节平台进行开发。

后来又出现了容器，“语言后端即服务”。（baas,容器化，对语言来讲容器其实还是OS或虚拟机，它类似一种wsl，只不过它可视为分出来的子OS，这种子OS作用是为语言提供分离服务，而不是其它目的，比如pyenv,和serverless后端）。当然它们不是语言后端仅可视为后端包装器 ---- 只是这次，os平台，后端，容器，这三者的界限又变得模糊了。 

以上这些都是平台与语言可完全脱节存在的例子。实际上，只要是脱离平台，该应用语言能对任何系统进行开发，甚至架空平台。

但是，它们本质上，它们都没有做到完全脱离。语言后端作为语言在这种具体平台上的实现支持语言在这种平台上存在，就是关系的纽带，不管何种形式存在（是对OS作后端还是OS上的虚拟机作后端），都有一个后端，都有一个OS存在：无论这种开发体和托管体，是single OS和APP之间，还是云服务时代，容器这种单元，只要它装配了语言后端，就可以解释为RUNTIME与OS的关系，这样这种后端与APP之间存在完备的运行支持关系才使之成为可能。即，它们是多平台而不是真正的跨平台，语言也并不是跨web/native开发，至少存在一种toolchain lang和一种appdev lang。（这里要明白一点：web不是平台是业务，lamp-l=amp才是runtime targetee,browser也是）

那么有没有完全脱离平台的语言呢，或者极大可能这样做的语言呢，这样做的好处又是什么呢？能给最终的开发（尤其分布式，WEB）带来帮助或颠覆性好处吗？

语言与平台脱离：极小化语言，微语言
-----

这需要从语言运行时本身进行。

这种语言和开发生态就是go和rust，这种极小运行时的语言。这就是我在《Golang，一门独立门户却又好好专注于解决过程式和纯粹app的语言》谈到的。go为每一个它生成的APP都内置了一个“go runtime as os”，而这种go runtime是不依赖任何平台甚至最初级的libc。

这种极小化生态，可以让语言更容易跨平台，甚至集成到浏览器（这是一种除了硬件嵌入开发的WEB嵌入式开发）。比如rust micro runtime之于wasm开发。而带GC的语言，往往不能用于嵌入式。

它带来的更大的好处之一，还可以使得virtual appliance成真。

virtual os与virtual appliance:  内置OS与language runtime的app
-----

在云计算机和虚拟化领域，有virutal iaas，就会有virtual os，和virtual appliance。见《一个隔开hal与langsys，用户态专用OS的设想》，《一个设想基于colinux,the user mode osxaas for both realhw and langsys》。

上面谈到容器，作为language backend包装器存在，它也包装了OS。比如docker，实际上每个docker实例都共享着一套OS模板在最开头，其aufs文件系统的开头，也引用着一个OS rootfs。而现在，随着unikernel的出现。这种专门为OS kernel进行虚拟化（而不是在OS内部共享内核针对rootfs虚拟化的docker）的方案出现了，它使带OS的容器可以真正变得极小。它针对解决的问题是针对容器每一个模板过大的问题，就如同rust之于runtime一样,从源头解耦。

再来谈virtual app,最初的virtual appliance是本地虚拟机跨OS出现的一种APP管理机制，如vmlite的那种，pd的融合APP那种。而rust的微内核，使得用这种语言这种开发出来的APP，是无须调用或依赖任何OS导出的API或服务作为宿主的：不是可不可移殖，是根本无须移殖（OS与APP可脱离且最小化，这就给APP虚拟化提供了基础）。

如果说应用开发领域，OS可以虚拟化，模糊它作为app backend的意义，那么app为什么就不呢？关键是这二者一旦结合，将带来真正的virutal appliance和开发。

1）首先，Unikernel才能促成application virtual，微内核将容器做到os管理的级别跟openvz和docker方案不一样。微内核可以集成到APP级别。使得virtual appliance真的跨OS有存在意义。

2）其次，使得虚拟OS可以被集成在APP内，见《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》，实现真正的virtual appliance，meta rootfs中集成虚拟机管理器。

3）1，2相加，针对那个最终的问题，可以带来更好编程的OS和分布式开发，使分布式真正集成到OS，在《一种matecloudos的设想及一种单机复杂度的云mateapp及云开发设想》，我们谈到现在分布式app都是跨OS的，但也存在一种原生分布式，即plan9,x11这种类似的“os绑定式分布式APP”，

该如何理解这种新appstack呢？传统的app是完成了gui,db,net之后在这上面，面向本地，堆应用，作为应用件，传统webapp是提出一系列关于db.gui,net的新规范（其中net部分固定为http）实现真正跨端，而新appstack是os限定的。跟传统os一样(受限于os透出的api/分布api)。新app是gui,db,net这三者，每一个都api化。且面向分布api化。作为开发件复合堆net关于分布网络。这里的思想类似于直接利用原生的os存储功能提供API和网盘服务。开发上，os即客服一体机器。语言库即os组件，见《一种matecloudos的设想及一种单机复杂度的云mateapp及云开发设想》第二节。

1，2，3可以使得nativedev适用于新的分布式开发，也使得传统web和web语言更容易拥有virtual appliance能力:webapp是初级的分布式和融合开发和融合栈的一种方式，因为它还没有涉及到virtual appliance和微内核容器化。任何应用编程，都是栈选型和归纳行为。归纳为有限几个栈。业务领域自有轮子。却没有真正的virtual appliance，因为web自身就是一种byond os的虚化平台，因此与web最搭的就是这种virtual appliance的appmodel。b端甚至可以不是浏览器直接是普通GUI。甚至结合rust microruntime和interpreter to js(这实现也是elmlang的特色)，可以用rust统一前后端开发。

-----------

rust的生态在慢慢涵盖我上面提到的那些rust coreboot，redox os微内核，User-space drivers 用户空间驱动，wasm,都在慢慢成真。目前这些只有molliza在做。感觉技术也是烧钱割韭菜。不过这套方案也确定是够slim省事。


-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108366441/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>

