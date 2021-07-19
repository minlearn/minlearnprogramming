enginx:基于openresty,一个前后端统一,生态共享的webstack实现
=====

__本文关键字：openresty,enginx__


webstack的前世今生就是一个重复造轮的过程，它的目标是将本地程序栈弄成分布式b/s web，其实这在语言端可以做(比如语言模块的http unit，然后是一层层我们从桌面时代开发最基本的socketapp开始，http封装之后也许是一个aysn网络io库，最终到达语言库级的webframework直到专门的独立程序支持，也许这个时候人们发现那个网络io库可以独立出来作为一个server，再比如第三方容器在这种需求下很容易出现，流控安全等需要也会泛滥)，于是终于发展到用独立的服务器OS组件来实现这些强化，形成专门的产品来做，体现在开发上首先是webserver+CGI处理。web作为b/s在架构上假设有服务端程序存在，而cgi就是开发web程序的语言同webserver交互的扩展，动态语言将运行结果转成web page app的手段。像mod_swgi，mod_php就直接将phpcgi做到了语言。如webstack.语言则屈居之下。—— 这完全是语言，独立件，一方做大了包裹另一方的关系但二者始终是一体的。

以前webstack都是语言加各个产品小件的堆叠，像wamp,lemp,wapp,lnmp,mean…等，除去语言和OS服务这些大件，显然存在前后端，首先是page server，然后是存储后端。mysql,rawfilesystem,document stor（文档xml强调的是文档语义，能存文件和文档是其第二层含义）等。还有各种为了其它功能新加进来的增强件比如种新的本地/分布数据库memcached,本地件imagicx。。

PS：其实这些都是模拟桌面时代的appstack,人类其实在各层次复用同样的方法，解决方案和产品，形成各种类似appstack,webstack的其中明显二大件之前端部分模拟的是desktop app时代的page,后端模拟的是filesystem等等，毕竟是分布式中已经定义好b/s规范，连协议http都规范好了，有这二个产品，就足于构成一个webapp stack必须的界面和存储了，只要有app domain logic，就可以产生一个app了 —— 同道理的是mobile app那些，这里省略一W字……
以上备注栏里你看到了应用程序形成和选型历史的大架构，好了打住，继续：

存在几种流行的webserver(有的有容器比如tomcat支持，有的对语言支持好比如apache rewriter强大,etc..,有的功能单一，比如nginx静态页面友好在流量控制方面功能强大)，人们有时还混合使用它们，这使得webstack前后端分区呈现更复杂的前前后,前中后，前中后后等结构。

这里要来说的是大头的nginx+apache

我们不需要apache
-----

其实作为一个webserver,它的本质是流量引擎+webserver,当它与前端语言cgi沟通时，它就是webserver,当它负责与后端+前端一起上时，它就是流控引擎。很显然地，nginx最初的意义是分布式流量的“enginx”，在这种意义下，nginx能管好流控这是它最大的责任和优势，而apache显然做得有点过了：

apache并不仅是webserve其实它还提负容器的责任，这种在用户态重造一遍的事应该由语言层（比如msyscuione中的包管理或engitor as paas中的语言层来完成）。它跟nginx一样存在流控与各种语言对接的后端，然而它走的路子是，由语言来完成对数据库后端的对接，apache负责与语言沟通，这其实是webstack通用的设计和应用路径，只是这种情形下，webstack变得有点碎片。各种语言和存储组合起来就是一个webstack，，这样看似自由，实际只能二块二块地用，十分固定，于是你看到的不是wamp就是lemp，而meam这样的东西是不好用的。

总结：，我们没有一个一体的webstack生态圈产品，各个产品都是语言绑定的。具体前后端紧联系的。甚至系统绑定的。原因是，webstack中webserver的功能定义不清。

nginx=engin x: 强大，专一的流控单一，也是前后端开弓的中间单元
-----

而对nginx的合理应用可以完成做到用nginx+其它产品打造一个干净的前后端：

nginx它没有容器不像apache，因为这是语言层要干的事。

在这点上，apache于webstack必须有的地方没有，nginx却全有。我们也不需要nginx+apache的架构。我们完成可以去掉apache,语言直接作为容器，nginx直接作为语言容器的前端，以direct servering langsys的路子进行。

而且nginx的流控实在是太强大了。它可以做到一体化流控（负载，访问点控制，QOS，反代）与安全服务器。

而且，lamp,wamp一条龙的webstack设计者们肯定没想到，nginx它不但可以server前端，它实际上也可以server后端，比如nginx直接与mysql构成不需要通过php的phpmyadmin版本。

这有什么意义呢？

它使WEB件，从语言件彻底分开，实现了诸如，提倡nginx前端直接与后端mysql等通讯，仅要求语言提供php-cgi之类的东西而不要求它们提供php-mysqli之类的东西，构架上清希化出了一个“整合的独立的，webstack”，而不再是分散的wamp,wemp,mean等等。从此不同的语言导致的开发，发布的，架构上的区别都不存在。都是一样的从nginx为入口的体系，它掩盖后端那些子件的复杂性和开发维护必要。更重要的意义：打造了一个干净的多语言webstack

大一统的webstack
-----

openresty的架构有趣就在于此，它是个足够创意新奇，大一统的产品，它对前后端都进行绑定。

nginx之于db,storbackend(openresty一志整合)，就像jupyter整合语言件backends。我们从此仅需要一套 “xny” webstack — x平台的y语言，及中间的nginx，就可以做到统一开发和发布，最最深远的意义还在于：有望借助enginx，所有web程序共享同样的产品生态，统一发布，无沟运维。

下载（本地下载）：

以下提供下载的enginx，在openresty基础上做了适应msyscuione->各个langsys的适应。与engitor(paas)天然互补结合:一个提供语言与容器，一个提供安全和最终的paas服务。。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339845/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



