docker as engitor及云构建devops选型
=====

__本文关键字:docker as engitor,云构建devops__

在发布《engitor as demo show engine,applet container》时，我们谈到engitor是一种延续langsys，独立于开发，但对接开发增强开发，及负责整个发布和部署,甚至运行（除了xaas层次的那部分运行）的综合过程统称，解决开发完成后，增强开发环境，问题域demo组装及发布至上线的一系列后续工作。类似应用服务器，但不止这些。比如vs appserver,an engitor可以是强化语言系统之后可视化的开发增强支持(engitor=app engine as editor)甚至提供baas，paas这样的运行增强支持，而传统appserver仅负责特定的部分。

>我们总把整个软件工程分为xaas,langsys,engitorx(appdomain),apps四个层次。越到最后规模越小，程序逻辑越片断化具体领域化，最终，它使源码形式的语言系统写出的源码片断程序，可以以可视化的形式开发，碎片化发布/累积的方式运行，使得练习和演示可以向最终应用逐步无痛地迭代。。我们一直在为这些领域选型。如上所述，appdomain的engitor承上启下是规模较大的一块，其选型也就越复杂。

在那里，我们主要选取了jupyter和openresty来说明engitor：

1)在用jupyter充当这个engitor时它同时是enginx，它的特点是.ipynb，技术上这实际上是一种web脚本和各种语言后端的服务环境"web engitor"(但是它支持cs app化engitor)，.pynb可以欠在webviewer中也可以欠在其它支持jupyter cs协议（类似cgi）的clientview中，由于engitor是支持多语言的。所以，它实际上是将多种语言源码片断(a note)统一发布成web应用的服务端脚本的形式并将执行结果返回。这其实就是动态web脚本的理念。但是它第一次实现了将不同的语言统一化为服务端脚本，且提供了一个在线IDE(以开发一段note测试一段note的行为)。而实际上足够复杂的ipynb是可以开发app的，也不限于用在线IDE的教育工具的形式去展现,其前后端一个是语言一个是应用就像普通WEB一样。

2)而openresty的enginx是纯运行方面的engitor选型，我们在《发布enginx中》，提到组件服务器环境，它使服务性程序的协议部分，变成组件的交互。将lamp,lnnp,lnmp结构扩展到可配置的通用组件化服务器程序的结构(实际上是利用lua为nginx写脚本)，而kbengine这样的结构就演示了如何用openresty来充当通用程序的enginx-game app

legacy engitor和devops云构建
-----

以上选型都有几个共同的特点，1,在这种engitor是一个组装运行环境，这种语言环境“在线收集合成了”用户碎片化方式提交的源码逻辑，是个云构建化的开发环境类程序。2，且形成的engitor app要在这个engitor辅助下运行，因为它要面向源码片断输出这种源码下的应用。这此都符合我们对engitor选型的一惯要求和标准。

那么是否能构建一个engitor，它依然能够面向对一端是语言src逻辑输出另一端是应用输出而不局限仅用于要求输入端必须是源码，输出端必须是APP？(一言以蔽之通用化构建任意程序)，且不要求运行在以上具体engitor下？那么这还叫engitor吗？还有意义吗？

毕竟，我们想得到一个万用的engitor，将传统上从(linux的生态开始处,CUI处，那个时候仅有os kernel和toolchain)，将任何复杂应用的开发涉及到的多种语言源程序/二进制的编译过程，多种语言vm的打包过程自动化起来，将这些在传统上是构建脚本的编排技术，和OS的包管理技术考虑进来，甚至使构建本身云化和构建服务外部化云化，喂给远程构建-云构建，。形成自动化，云端脚本化编译的结果，并以此为运行目标，仅负责书写最终APP上的事。

这实际上就是输入端接受任意构建，输出端产生任意程序的单一要求而已。这样的engitor实际上以os为enginx运行，以能运行上其上的所有可能语言系统为engitor中的langsys。而engitor也不必是个jupyter+web执行环境式的“云构建”和中间件打包。比如，它可以是任何程序（非源码形式的某语言源码片断,二进制也可，非IDE类产出过程也可）构成的“云构建”和中间件打包。它可以没有任何关乎engitor意义上的输入输出。但是依然可以适用于engitor特例。

那么如何整合这些，这实际就是devops做的事。传统我们在PC上用各种开发用的虚拟机vagrant，那么我们现在有docker和devops

docker as engitor和云IDE。
-----

在那文中我们讲到jupyter也有jupyter hub。实际上它相当于docker版本的github+dockerhub组成的devops。

docker as 通用构建技术和容器的情况下，实际上docker与docker-compose是二个独立的过程，docker只负责run，而github相当于ide中收集源码的工程环境，那么我们还可以得到什么呢？
比如结合前面的ellie，我们可以在结合docker和gitlab cl for elmlang的情况下，把这个ellie ide放进去。做一个云IDE。

自由大开脑洞去吧。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340001/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



