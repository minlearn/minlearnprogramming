聪明的Mac osx本地云：同一生态的云硬件，云装机，云应用，云开发的完美集
=====

__本文关键字：native cloud,uniform cloud os and client os，利用PC打造云平台。caching as cloud, sync as cloud__

在前面我们讲到多种云平台和云OS的选型，像群晖，dsm之类都是为yunos专门建立起一种硬件平台和软件系统的典型，它有点像极路由，其appmodel和各种扩展都是基于web选型，它们的区别也许只是一个：一个面向大容量存储服务，一个面向连接服务。它们是设想与生产环境和工作环境一起协调工作的，但这三者明显有功能和职责相交的地方，但如果，存在有一种硬件软件选型，既可以负责起日常工作需要的那些任务，还可以同时作中心存储等任务而不需要另外添置新的硬件，不是更好么？这既节省了一台硬件，又节省了一个OS的开销。比如，就把PC同时当NAS当工作环境当路由服务，不好么？

在前面在《把群晖+DOCKER当WEB云OS》一文中，我们讨论了云OS从硬件到一系列其它方面的选型，包括开发。那么新的集成方案可以是同类的东西么（yet another alternatives）,这需要涉及到一系列问题:

>首先，生产环境的客户端和服务端能否做在一起，nas和软路由强调自身作为服务性环境存在，如果将其移到客服同体，比如在pc上开一虚拟机或docker或其它什么方式虚拟黑群晖，这会造成资源抢占效率低下，这台pc作为客户端也需要配备大功率的显卡，显得不伦不类，但这也不是什么大问题----对于个人使用是可以，我们在前面也涉及到多种host/guest共体的选型，host其实也可以作为管理OS。这其实我们第一种方案就是这样的《利用colinux和mailinabox在本地打造mineportal2》
其次，客户app和服务性app是否真的可以共存一PC中，比如使用容器技术分隔资源配额，互不侵犯。这样还可以统一传统的包服务和容器服务，像沙盒技术可以同时用在客户端和服务端APP打包技术中，只不过服务性app或容器需要透出服务，而且通常采用的web UI模型，其实普通的GUI就够了，因为也有remote appmodel这种程序，如果非要涉及到web，可以仅把web当ui,不要把它当一种全栈的appmodel，比如利用静态网站生成器的原理，treate web as only ui ------ 其实app提供一种ui就能在其上打造出一个appmodel而不论你在其中采用的UI方案是什么。
第三，关于同步，在前面我们看到群晖是基于客户端与服务端sync的，是工具性质的，更前面我们看到cloudwall这种是基于pouchdb这种数据库级的自动化sync的。----- 其实，一个appmodel，只要它是远程分布的，只要它提供客服sync方案的，这基本都是云。而云，这个概念广得很，其实云不一定要跟OS有关，跟采用何种内容同步有关，不一定跟开发有关，它可以是各种加速，比如，云加速下载就是把热门资源用自己的服务器缓存起来，像百度云和迅雷一样，也可以是云计算，也可以是区块链。
第四，各种虚拟化和devops支持-devops其实就是云开发的代名词。这些我们慢慢道来，在《TERRA++》文中我们提到可编程可以成就devops和CI。---  其实可碎片化就能可持续集成，像文档化的xml，可编程化构建的容器语法都是，这些在上篇《terra++》中有述。

综上，它们在技术上他们也是可以统一的只是选型不一，但都可以是云。所以是可行的。

Osx即是这样做的，还做得特别聪明，为什么是osx呢不是win呢？win没有集成太多的云服务，且除了pc只有mac是最接近可用的生产力工具了，而后者刚好也做得很不错很具典型。

osx的native cloud模式:remote gui appmodel+caching as cloud
-----

Osx生态没有专门的server版，是直接在osx上建立的，要说有，也是一个叫mac osx server的普通app，但osx被当作云还有其更多方面。

首先，其icloud与finder是整合的，数据存储在苹果的服务器，document文件夹(osx叫它文稿)通过finder作同步客户端，这是第一步，第二步，配合osx server的描述文件管理器，icloud数据还可以caching在另外一台装有osx server的机器上作本地同步,vs sync as content distribution,streaming还没有发展起来，caching其实也是一种聪明的方法。

>使用osx yun，你不必自建那么多服务，这些服务商的寿命比你的寿命还长，还有，这种中转性的服务足够了。比如我github一下，也可以混用客户端与服务端。本地只需作个中转不需要自建服务。而且osx的各种icloud app，比群晖的那些更好用。比如备忘录，图片上传多端可见，

其次，osx是直接把任何服务性的app都当成云APP的，这就是我们在前面一直谈到的，vs web as cloudappmodel，其实remote guy appmodel也是云app的一种，内容分发方面，如上所述，现如今5g和streaming技术不太发达，这种appmodel还远远没有发展起来导致的。

第三，osx server负责管理硬件，管理APP缓存，数据的分发（内容分发），甚至其还管理装机数据的caching,即time machine(time machine这不就是群晖等的active backup吗)，而且，osx server的做法更聪明，timemachine这类app是客服共体的，就像finder作为同步客户端一样。而且mac它的生态更统一，涵盖整个PC还有IOS这种可移动端。

下面来讨论一下具体做法，并把其各种方案和与群晖类似的其它方面作一对比

类群晖,把双盘mini打造成mbp的matebox:装机服务器数据恢复器，访问点接入器，应用服务器，和devops开发器
-----

OSX的同步和appmodel讨论过了，然后是其它。

解决其装机维护的问题：

我们的例子中，用了一台双盘的Mac mini（一般装osx server一盘作time machine backup）,一台macbook和一台iphone来发明，在MINI在这主机上绑定破解的server,我还配合使用了oray三件套（蒲公英，花生壳，向日葵控控a2），因为mini是属于无屏环境，只能通过网络显示器，如ipkvm的向日葵控控a2这种来解决装机和维护问题(Mini的设计要是加个小led屏幕和有限键盘，可以启动免去外插键盘和hdmi显示屏就好了,其实群晖也可以这样，只是它有一个不坏的rom，里面有bootloader，不用像普通OS一样装机)，Osx server可以提供多硬件管理控制和细微的企业各级人士权限控制。十分强大。利用上述工具装完机后，可以利用a2远控远程装机，oraybox组vpn,花生壳穿透建站等，后期还可以time machine恢复，icloud恢复数据，etc..在移动端也可以这样做。这种生态集成程序是空前的。

解决其多开虚拟机的问题:

唯一与群晖系列不好比较的是osx 没有集成vmm这种东西，还有docker，不过这也可以通过下载appstore或第三方的同类服务来代替。我下载的是破解版的paraelle desk（集成docker/vagrant可以当devops使用），当然其它方案组合也是可以的，，客服共体的优势不但是GUI是共享的，而且APP生态也是共享的。

------

原来的bcxszy，在序言中不光是写了一体化语言，还写了一体化uniform web/native选型，中途我们改成了JS，现在我们回来了，现在的bcxszy2,加了一体化平台选型，开发转用terralang，在整个语言选型中，我们追求的都是一种平台与远程开发兼容一体的语言。，从下一篇开始，我们写《利用terra++模拟cpp》,用terra++，是因为只有自己发明的语言，你才能用好,就像我们一直在发明自己的dbtinycolinux一样。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338732/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





