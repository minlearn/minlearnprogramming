cloudwall：一种真正的mixed nativeapp与webapp的统一appstack
=====

__本文关键字：在数据库中安装程序。以数据库直接为后端托管程序，文档数据库管理器直接为云文件存储程序。无backend webapp，在web中开发webapp__

大约在很久以前，我开始放弃追求统一化分布式应用程序和本地程序为同一个appstack的努力，这二者之间似乎天然存在鸿沟，像是应用的使用方式决定的，这种人为的界限并不是用来跨越的，拿web来说，它作为一种分布式架构和分布式appstack架构，不能做到像本地GUI程序或硬件加速程序一样灵活，比如web强调将一切放在broswer端渲染导致需要采用html5,webgl,js+css html这样的东西来增强它，这样它才能稍微像本地程序，一个例子就是用WEB实现的WEBGAME - 这种效率跟本地硬件加速实现下的game完全不是一种路子，WEBGAME的体验跟传统PC游戏的体验也相似并不相通，因为始终无法在远程上实现硬件加速还能stream到本地。web在服务端采用http而不是原生tcpip，导致需要websocket才能做到像主动推送这种原生TCPIP轻松办到的事。当然还有很多不同。

这不是WEB的错，WEB最初就是那样被定义的：它本来就是一种高级的native tcpip程序构成的生态。它的界面是PAGEUI，而PAGEUI是一种应用层的渲染，在服务器端，WEB程序大都由LAMP，LNMP这样的东西作backend，这类程序本身，其实是普通的TCPIP程序，并不是某个WEBOS的基础组件，就像原生程序之于传统OS实现中的任务机制界面机制一样，这也就是说，所有的WEBAPP都是有backend的，就是那个lamp中的amp等东西。它们用服务器的方式组建了一个分布式appstack，定义了一种appmodel，因此历史上，像WEBAPP＋WEBOS这类东西并没有纯的，- WEBAPP是原生界面中采用有限技术打出来的一个点再在这个点构建出的一整个stack生态，因此，WEBOS也是OS上的高级OS而已 -- 本身并没有WEBOS存在。

在《在tinycolinux上安装chrome》上我们曾谈到相似的概念:chromeos脱离不了它其实就是原生界面（X11，GDI）加一个浏览器的技术本质，其实并不能与真正严肃的OS工程类比 。一个像群晖那样的APP管理界面就能称为webos。还有像owncloud,standstorm这种：sandstorm比oc多了xaas的部分。

web作为云计算负责定义APPSTACK的成份意义比较大，云计算下的程序无非就是WEB程序，因此云这种东西，除了虚拟化那一层，在APP生态上，它其实依然没有属于自己的东西。依然是高级原生分布式程序的BS化。

那么，这一切会不会有突破呢？有朝一天，WEB也有自己完全不依赖传统BS架构的东西呢？变得像一种真正独立的，由新的东西构成的应用生态呢？而cloudwall也许是另外一种“webapp”:cloudwall的确提出了很多新的耳目一新的东西，它虽然还是面向WEBAPP，不过它其中的一些部分可以作为与传统WEB迥然不同的部分来产生新的审视，比如它的nobackend设计，它的宣传语也一针见血：cloudwall，an Operating system for noBackend webapps.如它所言，它甚至提出了一种新的webapp和webappstack,webos雏形---改变了传统webapp中的大部分。

cloudwall中的couchdb:the only backend as webos部分
-----

首先，它使用了apache couchdb，这是一种直接与WEB接轨的文档化数据库，如果我们把我们接下来要谈的APPSTACK称为某WEBOS的appstack的话，那么couchdb定义了这种appstack的唯一的backend部分，这免去了需要lnmp作backend的需要：这是它独有的特点支撑了它与lnmp这些东西的某组件明显存有不同的所在：这种DB是文档型的，且它nobackend。

couchdb支持直接hosting app并运行，称为couchdb-hosted webapp，它加一个类似数据库管理器的东西天然就是一个类OC的云存储程序，支持各种cluchdb插件的开发，这就是webapp整个cloudwall就是这样一个couchdb管理器。

>CloudWall is a kind of offline-ready toy in-browser OS, for authoring, storing, and sharing docs and CouchDB-hosted webapps. CloudWall installs via replication and needs only CouchDB and modern HTML5-compliant browser to run. Also CloudWall can run on static hosting, as a set of files.

>All CloudWall components run in a browser tab, no server or even internet connection required after system started. Any local DB can eventually sync with external CouchDB instances over http(s). One CouchDB can have several users connected, thus providing shared workspace, docs and applications set.

在我的《appstack series》《app series》系列文章中，我一直在寻求云存储程序的选型，我们换过mongdb,postgres,这种程序选型其实说大了就是WEBOS，我们在这些文章中都提出过这样的企图和设想。

来看一个这类OS的设计：是否一个app必备一个stack?将它的栈放大到受WEBOS直接支持，那么这种云程序背后的OS技术就会明朗化：

实际上，当考虑到一个app要配一个appstack东西的时候，它依赖原生程序appstack定义了自己新的appstack的局限就永远都避免不了，因为这里的mongdb,postgres永远被当成了appstack的dbbackend部分，，，而webapp应该是没有明显无backend的：像nativeapp stack一样，它们应该被集成在某一webos内部被提前解决掉。

而couchdb就是整个用数据库管理系统来作OS直接管理和存储WEBAPP的东西（当然它也能天然像其它文档数据库一样直接管理静态文件作云存储），如果将couchdb像cloudwall一样作为整个webos，那么传统的webapp开发就被定义在这个webos中，cloudwall的四个appstack组件，它们被集成在称为cloudwall os的webos理念当中。

>GDI:Renders HTML and receives user interactions

>App runtime:Manages bindings between data and UI controls

>CloudWall:Prepares, runs and closes apps, manages app switch

>Storage:Stores apps and documents, optionally syncs with external CouchDB instances

而这种开发，已经使webapp开发变得像本地一样了（无须处理appstack的部分只须关注app内的事情），我们一直希望得到的效果：webapp像本地一样以文件存储为后端符合像本地应用的习惯，这个目的也达到了。

cloudwall中的inapp editor：语言和开发部分
-----

在《bcxszy series》在所有的努力中，我想得到这样一种程序和开发方式：不改变原生程序与webapp的大面，使WEB程序变得像本地程序一样简单，这样可以共用本地程序/webapp开发的概念，在模糊appstack方面，这就是cloudwall中的couchdb中谈到的，已经被解决。这里要谈到的是与语言开发有关的部分：

可以说，在《bcxszy series》在所有的努力中，我还想促成这样一种程序和开发方式：源码即文件，随处打包再走，直接per app an ide开发,这无论对实用和开发，编程自学都是极为便利的。

所幸WEBAPP src 文本化，支持轻量带走inplace editor是所有WEB程序它的天然优势，而且虽然一开始WEB程序与本地程序有很多不同，但像WEB标准化，HTML标准化，JS语言标准化这样的东西，它们其实在走一种联合化的努力方向，使WEB生态接近本地生态，比如，JS的努力方向也有一种是nativejs：reactos

其实couchdb与web结合紧的另一方面就是js，js是一种能够真正带来naitveapp与webapp合一的增强剂，这使得cloudwall支持极度便利化的inappeditor，这样cloudwall支持下插件的开发就是cloudwall webos下的webapp开发了，它支持用couchdb直接存储和保存编辑app开发过程中的文件。

这样有了以上这二点，personalcloud应是web os的论据就充份了。在开发体验上跟本地开发一样，甚至更简单。等cloudwall以上二大概念不再局限bs开发时，那么它就可以是新一代的WEBOS。

-----------------

这篇文章可以用来丰富《编程实践选型》web的极大化未完的部分，整个文章的思路可以用来作为《bcxszy》part 2实践部分。如果使用cloudwall的理念，以后《appstack》，《apps》可以整合为《apps》

关注我


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339586/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



