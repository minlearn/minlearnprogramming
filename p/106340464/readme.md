hyperkit:一个full codeable,full dev support的devops及cloud appmodel
=====

__本文关键字：app level vm,hypervisor as component, hypervisor as api, langsys as api，cloud infrastructure as code，Devops is only tools but not appmodel, Lua as configuration language__

在bcxszy和bcxszy demos的历史选型上我们一直希望得到一个在native上面改良云/分布式程序/cloud的设想，接上学习，开发和应用的断层，实现学习上的最小投入和应用上的去重化 —— 并寄希望于它们是一块自成一体的(from xaas,langsys,appstack to discrete app)，在我们的集成思路和实践中，我们忠实地 1,发明了一个os:dbcolinux并用packer生成了它 — 它代表xaas中的云os，2, 提出了一种语言:terralang — 它代表devops langsys, 3,参考过openresty那种用发明传统服务器程序来写web程序的方式:enginx — 它代表appmodel —— 我们最终想得到一个创新了的集成三大件组成的fullstack devops in a box-我们叫他engitor,这一切都有别于我以前选择过，现在强烈鄙弃的现行web集成方向的东西:它们使用web appstack,使用js，与nativedev断层，这是一种在栈上造栈的行为，造成了学习成本加大和资源浪费，而且与未来的5g脱节。

>我们来理一下:

>以上terralang是C runtime and compile as a lib，整个C as a lib可以被lua codeable，这就接上了devops，这个lua可以作为c的shell语言，ide语言，甚至给c内部增强，extending c使用，openresty也是lua codeable的nginx。terralang可以直接调用和发现c dll as api（通过头文件，免binding），也可以直接编程一个binary和配置一个binary本身，以类似shell的方式调用他们（而不需要任何封装工作）。———  这一切都有了devops的可能，而且是lua的devops。

>这跟packer封装构建的dbcolinux式的devops却有不同:dbcolinux虽然也是可以packer配置出来的，也具备devops的可能，然而，它使用的语言是yaml。我们在前面谈到hashicorp的packer是利用非编程的方法，一种配置语言HCL is the HashiCorp configuration language，配置的devops，就像docker file，使用的yaml。这二种语言在现实生活中却都需要。

>因为一个运维者有时不是一个开发者，按照复杂度，一个语言必须至少分化出configuration language和script language，前者供运维人员或普通配置用户，部署人员使用。后者程序员使用scripting的情况下，更需要更为复杂一点的专业语言。而有些天才开发者，可能需要更为复杂的语言，但这就需要在保证语言核心尽量不变的情况下，给他们封装各种各样的sugar或增进语言机制（这会造成语言核心膨胀），但第一种人，明显是一条一条顺序语句的配置适合他们使用，甚至数据填充data drivern式的配置更适合它们。像html就是标识语言，lua nul就是标识语言。这就是为什么有了js还要有html的原因。

那么有没有作为库存在的xaas devops呢，而不是像packer那样的工具。使得1，它也可以 lua coding 的方式用于devops呢，2，而且同时提供配置式开发接入运维者。

hyperkit及其支撑产品刚好是用来解决上述二种问题的。

>hyperkit

>HyperKit is a toolkit for embedding hypervisor capabilities in your application. It includes a complete hypervisor, based on xhyve/bhyve, which is optimized for lightweight virtual machines and container deployment. It is designed to be interfaced with higher-level components such as the VPNKit and DataKit. HyperKit currently only supports macOS using the Hypervisor.framework. It is a core component of Docker Desktop For Mac.

>Hardware-facilitated virtual machines (VMs) and virtual processors (vCPUs) can be created and controlled by an entitled sandboxed user space process, the hypervisor client. The Hypervisor framework abstracts virtual machines as tasks and virtual processors as threads. —— 这跟colinux的思路有点相像。

下面来详解它的意义和用处：


1，有助于打造一个可编码的用于devops容器。
-----

也正是linux在各个层次/各种模式的可存在性：kernel ring0 space,usr space,cooperative mode, as subsys, as sandbox,vm space,hypervisor space,甚至于浏览器中(有用js发明的linux)，hyperkit —— 它居然可以存在于app中。

这就可以把全套devops放在app级，内欠式地去做。甚至以lua编程的方式去做，因为hyperkit是基于c api的。，而hyperkit就相当于可编程的xaas

这使得我们集成三大件的路径变得有了更多的取巧，因为hyperkit小而且紧凑，与linux kernel,lua runtime ,nginx一样以小为美，设计正交，我们何不将其放进一块soc中，形成为一个box应用。hyperkit和其打造的生态各种kit即是这样做的。

2，打通运维配置式编程from the same scripting langsys和binary api service发现
-----

如果说terralang能发现api免binding，那么hyperkit可以从成品中制造api不需要设置成开发接口免声明。

一般地，dll是开发者用的东西，其服务和api要通过某种语言，被编程才能透出来，binary可以被配置，其实一些组件机制可以用xml的方式直接配置，形成应用，像pme加持久的xml文件，但是它终究不够通用，且借用绑定专门的语言机制，因为我们用了hyperkit,相比之下使用hyperkit要通用合理得多，它可以借助一个叫datakit的产品。

datakit: Connect processes into powerful data pipelines with a simple git-like filesystem interface

而现在，进一步地，binary可以被进一步以更灵活的方式用于配置：借助hyperkit，可以对二进制进行输入输出的shell式再编程成为可能。

datakit只是提供了一种可能，使得多个kitbox可以通讯，在这个层次上重编程，至于所用语言，肯定就是terralang,这样就可用同一种语言完成shell编程和一般编程,形成一个运维与开发统一的开发思路设计:以c+lua为中心，所有的逻辑都可包装为shell调用或sugar语法

还是那个问题，如果要兼顾运维的话，还是需要提供配置式语言，这个terra同样可以办到了。lua有专门的类packer yaml的配置语言生成器。Universal configuration library parser for nginx.

有了配置语言，我们甚至可以裁剪式将terralang+各种DSL发布成各种语言的发布版，就像发布linux自定义了rootfs后的结果一样。

3，真正的appstack的事,Devops is only tools,appstack is the essential
-----

其实devops只是工具，它并不改变任何东西。它并没有带来appmodel的创新，那么，我们最后来思考appstack的事。四大件中，xaas，langsys，appstack影响appdev，那么，真正的云程序是什么程序呢？

我一直在寻找真正的native cloud能完美融合的统一appstack，从来就没有出现过真正的云程序，Web拖慢了真正的分布式程序达几十年之久，真正的云软件是saas，它是传统软件的服务化。web并没有按这个标准发展。web的api规范其实也有，wsdl等,但不是废弃就被证明不好用。我曾经以为虚拟化，devops，这些规范下的app就是云程序。但其实，分发层的所有东西都不足于区分web与saas app的区别。横跨本地app/webapp之间的鸿沟不可smooth，不能在web之上做saas app，这方面的例子，实在太多了：

比如，内容同步，见cloud wall。不但pause/start式的工具，其同步是表层，pouchdb的自动式也是。 Page ui也只是表层。整个web appstack也不能。是web设计中的链接可点击到达么，也不是。Devops也不是，那只是工具

如果没有接上开发，那么一切都不算数。先有云开发才有云APP,必须首先为云app建立起开发的最底层的那些api的定义，甚至硬件层的定义，然而将这一切接上一门语言，用于开发出可部署的东西：这就是最终软件的定义。

以前的RMI，broker，DCOM，这些，才是远程，分布式程序的正确思考方向，然而他们都被废弃了，WEB+脚本，代替了一切：因为移殖了VM，就可以在本地和远程保留一样的api service，无谓区分是代理或打桩过来的还是本地的。—— 可是这一切成全的WEB有太多缺陷了，最大的隐患就是它接不上未来的高速5G，5G可以streaming一切，web本来就是适配低速网络提出的js，现在它要把它还给高速时代的那些开发语言和方式了。

比如，在开发层，甚至OS的内核调用，本身就考虑了远程调用。甚至机器硬件级内置了远程调用，才可以。还比如像sync 内容一样sync api。建立一个api service server per app level模拟本地远程无差的api服务与结果。即建立一个sync api的维持结果正确，或称反作弊过程，类似fps网络游戏的消息同步模块。

4，一种可能行得通的appmodel设想：P2p 客服同体，只需sync api结果即可的app
-----

因为有了前面的1，2，3讨论，所以这里的讨论方便多了。因为这是xaas,langsys,appstack到final appdev的自然路径。

最典型的就是git，Git，一个客服同体,基于内容同步的云程序,fetch,pull original，都在同一个端同一个操作.在这种情况下，web只是一个cgi转换服务，或类似静态网页生成器的工具，根本不必为它发明一整个orm的大型domainstack中间件。

部署的时候，如果要用上多节点，可以将服务端的部分装在一个xaas os，另一个装在纯粹客户端。


-----------

未来会为整个文章描述的devops建立一个engitorbox，其下的开发是all lua codable的：terra/Lua/c，lua scripting, Lua shell，lua sugar.

这一篇应该是覆写序言部分关于xaas,langsys,appstack综合选型的而准备的《一个nativecloudbox:围绕openresty精简工具箱:nginxbox》，我们的新demo会取名egxbox，直接在openresty上强化成,一个围绕着openresty的真native/cloud appstacks与devop工具集，并以此为中心设计实作



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340464/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


