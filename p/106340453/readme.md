用开发本地tcpip程序的思路开发webapp
=====

__本文关键字：the headless cms,b/s web to c/s web, headless webapp backend.__

不可否认的是，webapp已经是与desktop, mobile app并列的主流appmodel之一，但是，web却是一种典型的Appstack as os，webapp是在native server apps上打出的一个b/s洞，再在这个洞里发展出来的一整个世界(跟移动端APP一样比如安卓)，，比如，它底下的appstack，分别属于native/desktop的范畴,负责gui的是nginx,db mysql,etc(代表webappstack的是服务端app，客户端只是“client side html,js resources”，它脱离了服务端，就没有自身存在的意义。不是一个独立可用的app)..

Vs nativeapp和nativedev，它从来没有自己的os，或任何标准的宿主定义。较之native app，它不算是一种有专门运行它的OS供它托管运行的“app”，你要说webapp的host是lamp，很明显，lnmp中的l并不属于web,是applicationserver?beanserver?,也不是，它是语言的组件服务器，换言之，只有nginx,mysql这样的东西属于web —— 只是说，技术上通过ln+np的组合，能搞出一种web程序。这跟移动开发类似，它们都是linux和一种虚拟机语言双重托管运行下的app，——— 本来嘛，web开发和移动开发是beyond native层面的，也只须这样。

web的设计与缺陷
-----

在开发上,动态程序的web app是monolith的前后端整合的，叫page app，程序员在后端完成所有的程序开发，Webapp的框架逻辑无非是routing,template,orm，route，mvc这些框架逻辑。代表一种appmodel的，无非就是它的stack框架逻辑。因为它考虑进了浏览器是服务端和客户端一体app。web程序之间不用交互和复用，没有api机制，也没有web件，web as service(当然，这些后来也有。。。)，只有语言源码级的复用。

应用上，和后端运维上，也都是整合在web的。用户在一个web界面上完成所有的事，比如cms后端管理。用户和内容也是集成在远程的。没有线下只有线上。
这也就是说，程序靠后端，内容靠用户，全民线上分布式，没有线下分布式，它整个就是monolitch的（而且采用http，js这些具体选型没有二选，使得webapp是fixed的）。

以上这引起都不是问题。也不是web的问题。就像Web刚开始一出来其实就是分发静态文档只是后来有人用它来强行运行webapp而已(而且分布式应用开发本来在工业上的实现就很破碎，历史上并不存在一个真正的分布式OS，也不存在一个分布式appserver，见《plan9一个真正的分布式OS》) ——— 历史上，长链接，webgl,streaming content，这些，一直都是从各个维度去克服web monolitch page appmodel，使之多元的努力，

只是，只能有线上分布，不能有真正的线下分布，web这个缺点是显而易见的。有完全适合将web置于线上的现实需要，也就存在与现实的web应用现实相左的需求，比如，存不存在一种线上线下合作分布式的webapp呢？那些在本地可以处理的就让它在不必在远程，比如后端管理，使之跳出browser? 就像git的分布式那样，——— 在前面，我们也不断讲到此类思路，比如用静态网站思路来开发webapp，用tcpip来开发b/s。

新webapp
-----

这样的方案是存在的，网上有wordpress headless cms这样的项目，这样努力的结果就是重新将web置于规范级，将webapp重设计，它仅需要是一个http协议，也不必是一个b/s app，web只需是一个gui，而不应成为full appstack的全部。这样就分离了线上线下。线上线下分别开发，这二者通过api链接。

重新分离的好处多多，最明显的就是，开发上：

1）Web的服务端可以真正作为headless backend，变身as service服务。有api机制和复用。客服分离开发，用c/s方式和类nativedev方式开发，客服不再拘泥彼此的技术规范和语言技术选型。

2）简化了服务端开发和选型，显示逻辑分离，服务端web框架再不用mvc这样的东西及其它同时考虑处理客户端routing等的逻辑,Lnmp中也不再需要php了。可以在服务端用任何一种语言来实现。也可以有gitstack这种多选择的选型结构。或仅是其它采采用了http的其它非lnmp的xxxstack,所以，webapp的后端可以是任何东西。

3）将客户端开发独立成线下，不再将webapp视为一个monolith的appmodel，类c/s web，可以用任何语言实现将html视为编辑器中的asserts，不仅是浏览器了。，可以是其它独立的客户端app,

4）变“瘦服务端开发”“强客户端开发”，比如Wordpress,插件可以在客端做，不用开源了。或者反之，那种复杂的线上交互网站，也是可以的(可是，那还有其它方法来解决不是？比如将动态部分也弄成一个headless “chat”,headless “comment”,headless “xxx”??)。

应用上：

1）cms可以发展为headless cms在本地管理，拥有真正独立的客户端app，可以fully Turn into functional in-browser application. 

2)如果是展示型网站如cms，远端web程序也只需是个存储空间。比如阿里云OSS这种。这样，一个静态空间就可以解决cms托管的问题，而且更专业。

还有更多。。

其实，上面几点就“treating b/s webapp as c/s webapp in nativedev manner”一个意思。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340453/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



