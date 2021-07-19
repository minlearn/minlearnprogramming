比WEB更自然，jupyter用于通用软件开发的创新意义：使任何传统程序秒变WEB
=====

__本文关键字：online language,在线语言系统,jupyter,ipython jupyter,在线编译器，在线解释语言,engitor__

在《engitor:基于jupyter,一个一体化的语言,IDE及通用分布式架构环境》一文中我们提到，jupyter不止是一个分布式IDE它还是个分布式架构，更准确地说，它使任何使用engitor开发的程序变成WEB架构下的程序，而这个“WEB”，是jupyter webiz之后的web,然而它比web更自然，更强大。

jupyter作为一个极具创新的普通的产品不可忽视性，在于它是一个能改变现有软件开发，发布，甚至教育，用户体验端的东西，使之秒变WEB，做成WEB架构下的该程序版本，正如jupyter主页上提到的interacting computing，利用这二字官方强调的重点还主要在这：可用于编程=可用于开发布署=可用于定义一个appmodel，也就是利用jupyter可以达成一个计算。这个计算就是WEB化。

因为jupyter system实际上是作为一个一体化的多语言开发测试部件live code,debug+demolet show engine和ide,meta container环境。而存在的。有点拗口？

第一，它增强了分级容器的范畴。使程序接上WEB式的组件开发环境，以及UGC支持。
-----

普通的容器就是PAAS，RUNTIME AAS，像gae这样的东西，lamp这样的东西，在ipynb面前只是基础建设。前者没有细化到具体应用级，是从云服务到语言服务到语言框架服务的分级细化过程，提供服务的厂商只能算是ISP。而后者可以做到应用生态内部。为具体应用定义一个由.ipynb 组成的demolet定义的容器环境。

jupyter将应用接上了一个任意式的demolet容器，如果说传统分布式方案是从云服务到语言服务到语言框架服务的分级细化过程，那么jupyter就是直接具体应用分布式。

jupyter是langsys as distributed srv,langsrv,这跟容器/applicationsrv —- 语言库api as services 不同，它更适用当前者完成了之后，要将容器运用到具体应用层之后的情形，

PS：语言即库，库即API，API即组件。始终要记得可复用件在编程史上的演化（特别是其与平台，语言系统结合，脱离语言系统后变成分布式可开发复用件之后那些形态），可复用件必定存在一个容器或hosting backend.接下来谈到的分布式API也是如此。

第二，它增强的是WEB等分布式API，从程序的开发发布方式变成类WEB的分布式架构。
-----

分布式架构要纳入被开发，首先要API。程序的最上层无非开发发布，最原始的开发件叫API，最原始的部署件叫库，传统WEB和桌面分布下，都有自己的方案。从本地组件到到分布式组件，到接口到服务性API，到脚本（甚至开发件和部署件在脚本环境下最终做到了统一，这就是组件,demo即api，能在语言环境下动态运行services方式被识别为api的，都是组件），都要跨网络和socket这层。WEB是直接整合HTTP。更通用的WEB是退一步将HTTP换成了websocket，可是于任何方案将API做成分布式，都免不了要解决网络这一层。

jupyter的架构最上层CS二端用消息通讯，消息是文本化的协议表示。更接近语言处理支持需要和持久化定义需要。比如json,xml-rpc,soap就是用对象化的方式来持久表示协议的方案。

jupyter基于消息，可用于BS，CS，它免HTTP是一种比HTTP WEB还要通用的通用分布式CS协议。是另一种websocket之类的等价品。它使纯粹一体化的CS式WEB变得可能，即jupyter WEB（jupyter console,qtconsole as common jupyter app based appbroswer）。甚至不用到websocket和jupyter notebook时免传统浏览器。

第三，它增强了WEB等综合通用分布式APP的范畴，使任意程序变成page based applets。
-----

这里的APP用来指终端程序（VS库框架等），用jupyter来构建WEB，代码和运行可维持only in a page unit…发行单元以一体化的ipynb page(含界面+逻辑一体化不再是分散的html+服务端处理脚本)为单元。是富文本界面（js+.ipynb needed)和渲染过程的传统WEB界面的等价品。而且，天然CS，no bs式的web可分离部署是个巨大优点（ 有时候省事做站要求我们将逻辑后端和界面大量静态资源的前端分开部署）。

综合一，二，三，api的发行直接源于语言系统并如一所讲提供了由任何.ipynb demolet组成的容器环境，能采用WEB开发架构且脱离浏览器的分布式程序，这已经是个“通用分布式”的WEB环境了，于此，jupyter做到并增强了WEB。能将任务jupyter支持下的语言系统的任何程序变成WEB架构。


-------------------------

一切的一切，是对于分布式架构开发部署来说，jupyter的断层和抽象切入面是：从语言层就集成cs和WEB支持就开始将自己化身为langsys based srv that micmicing a websrv，，所以免bs.免分布式API..软件界伟大抽象的二大特点，一是选择正确的粒度选择，另一个就是jupyter聪明的断层，从精巧的切入面去正确的断层抽象。当然这可能是jupyter设计者没想到的而我只是个事后诸葛而已。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339910/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



