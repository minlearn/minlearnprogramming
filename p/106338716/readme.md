ubuntu touch: deepin pc os和deepin mobile os的天然融合
=====

__本文关键字：ubuntu touch as deepin mate os,second pc os__

在《一个matepc,mateos,mateapp的goblinux融合体系设计》我们一直寻求第二PC的硬件选型，它可以是一个小主机配个电脑通过typec相连供电，一台一体机配个小主机钉显示器后面，一台双系统机箱内的双主机，或者裸机架架起的主机群，还可以是你能想到的任何组合方式。二台主机可以同局域网（通过路由器），数据线直接交互，甚至异地（通过互联网）。。这种双主机需求是很常见和急迫的。

这些主机间用某个主机上的OS管理器管理，呈一样的外观，就好像他们在同一台主机同一个OS下的表现一样，这就是融合os，在《兼容多OS or 融合多OS？打造基于osxpe的融合OS管理器》《一种含云主机集群，云OS和云APP的架构全融合设计》中我们都谈到这种技术的基础和理念，由来，类parallesdesk方案：它尽量抹去了不同操作系统间的沟壑，而不用真的试图去填补这些OS间的异同。

谈到融合，有更多的例子，比如锤子tnt，三星dex将PC和mobile模式合而为一的显示方案，变形本，这些只是硬件上的例子，是处理现在既成事实的条件下，在多样化，不同质的产品方案间求得统一方案的权宜之计。还比如上面提到的mate os ------ 它本质也是一种融合os管理器技术。只不过我们要更进一步。

我们将从OS层面去融合，如果融合可以从选型开始加少量的融合工作本身，依然可以不用折腾太多。那么，何妨从软件的底层去融合呢？比如用同尽可能同一份OS同时用于pc,matepc,作mate os。这样，可以将相同的OS间共享同样的机制，subsystem，比如同样的os可以将同步做在os级别，matepc可以直接与mainpc互为可同步的mate，增加一个新的节点，只是增加一个同质的os，同步照样可用，运维也方便。。比如将mateableos作二份发布，一侧fs托管在别处。则另一侧必为其管理性系统，比如提取一个阿里云access key就可以在本地mirror它。这样就做到了在OS->filesystem层面的同步。

1,把deepin和skynas作为一对mateos?
-----

最近我用上了deepin linux(说实话，很早以前，大约2015年第一次尝试它也是各种不顺手，也不是因为小bug，而是根本不习惯bsd派生系用在桌面的风格和习惯，ubt之前也用过一直没能习惯，故放弃，后来折腾了半年的osx之后，有了过渡，所以这次2019年9月再次折腾v15的第11版，虽然时间过去这么久deepin已由ubt based变成了debian based,也由qml切换到了qt+go后端,虽然这次少量bug依旧存在，但最终通过试用它几天后我总算还是成功继承了自己使用在桌面使用osx的感觉)，加上发现它里面的应用已经足于应付我日常工作和开发了，而且也实现了它的承诺：美观轻量的linux桌面环境，所以最终决定就把它作为自己的装机OS,mainpc os了。

deepin还缺少icloud，timemachine这样的互联网，局域网备份装机支持，这也是我要为deepin找一个deepin mate的原因。我选择的是阿里云ecs+skynas群晖：虽然配备了大容量存储和本地式黑群非常好用，但配有公网IP和异地备份的远程云更合理化。现在ADSL也是越来越快了，如果不是用来存储小丽姐，其实最大100G的云服务器是够用的，而且这个成本一年也是个人用户能够承担的。

基于上面的同os的matepc设计，阿里云ecs上应装deepin，webdeepin，the headless deepin mate os for deepin，这样的第一步，是把deepin的kernel提取出来，作成一个syno的webasisst之类的东西 ，支持rootfs的安装和升级。至于mateos间的文件系统及文件系统同步设计（可直接使用brtfs的snap？oss远程文件系统？），又或者可用couchdb实现的数据库分布式文件系统。二个系统在开机后就自动同步了，不用在mainpc上像群晖一样打开一个守护程序。又或者它是一个git repo的东西，手动同步的，支持客服同步APP同逻辑（只不过remote,local分布不同）。

无论如何，为deepin增加云存储功能。且保证好用稳定的同步,互为mateable，这些，一定要做到OS层。

当然，未来我们的mateos，是Os级整个的同步，包括api,kernel，不只支持装机和用户数据cloud sync，因为它要是能够支持bcxszy的matestubos and bpi programming设想的，这是后话。

2,如果matepc还是一台装用mainpc os的手机
-----

可是它要是能用于三端mateable，手机和云端和本mainpc，这就是一个更为复杂的选型和融合了。

最近我还发现了ubuntu touch这个项目，其实不过这个项目在2018年就被官方deprecated给了另一个团队了，然而，它最大的特点是可以利用常见的一些手机作为matepc，甚至把它们当成开源手机硬件平台使用。这不是chroot技术，也不是linux on deploy技术，而是实实在在的将ubuntu全新安装在这些设备中。

ubuntu touch与deepin有着极为相似的生态，甚至可以将前者发展为deepin mobile.

在这台第三PC上，要安装mate os for pc,可以做成deepin mobile，----- 一般来说，pc和mobile这两个是个人最常用的mateable的标配，路由器或云主机都不是，路由器在外面就离线了没有公网，云主机同步过来的文件并不是立等可取，只有mobile随身带，当它能用于100G个人理想数据存储量。------ 这样所有的APP可PC可MOBILE，可ECS，mateable entity之间可以相互之间融合app了。

话说，ubuntu touch的目的之一就是降低多端APP融合的难度。这样多端一统的OS设计，可以同步用户数据，解决装机/全盘备份问题，甚至可以统一web，手机app，消灭web/webapp本身。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338716/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



