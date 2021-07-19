mineportal新硬件选型，威联通or群晖？
=====

__本文关键字：威联通vs群晖,公网IP盒子,群晖personal photostation__

在《使用群晖作mineportalbox（1）》中，我们谈到组织单一同步文件夹的省事方法，即，在那里，我们基于将所有用户使用NAS过程中产生的资料按逻辑置入一个统一可同步cloudstation文件夹的思想（这样主要是为了统一cloudstation上传下载+cloudsync云备份，且支持各种除cloudstation之外的xxxstation直接存取里面的数据，而避免一定要用到群晖默认的那套xxxstation->顶层xxx文件夹结构和多共享文件夹分别备份的需求），发现群晖是不能将photo,video（media），www,codesnippters(softs),_current syncing docs置入一个单一共享cloudstation进行备份的-比如将其置入home下或者一个单独可cloudstation同步共享文件夹，其中最大的问题就是photostation和webstation。前者通过ds photo自动备份上来的照片_current pics像册文件夹和photostation调用的图片必须要在顶层photos下，后者webstation不直接利用home中的文件夹建立虚拟机。

本文是那文的增强，依然只谈三个问题，硬件选型，公网转发，和省事单一可同步文件夹组织。

威联通
-----

事情是我得到了一台qnap，发现较群晖其硬件和系统层面各有特色，虚拟方面，x86架构都支持虚拟docker和虚拟机管理，有了虚拟机的情况下，mineportal作为身边的云主机这个概念才算是完善的-因为在资源范围内可以按需开不同的机器，qnap威联通性价比都说比群晖要高，但实际上qnap太吃内存了，同样存在开虚拟机需求的情况下，qts 16G内存不见得能开三个虚拟机，而dsm已经可以建立起好几个虚拟机了，好吧，个人感受当不得真。

qnap虚拟机的亮点在于其可当PC可当NAS可当虚拟机建立开发机可当服务器可当安卓盒子可当机顶盒播放器的特点，但其实只有docker出来的和linuxstation出来的pc是运行在裸机层次的，运行在虚拟机内的要通过qvpc技术透出到hdmi，实际上也是vnc网络信号，其体验由于虚拟机是软件指令下的机器，性能在本地vnc也是柔柔的操作体验-这是性能问题不是网络问题。

而，实际上，这种双系统主机是非常有意义的，比如nas系统+安卓系统，可以远程连接安卓播放nas中的东西，或在安卓中下载一个百度云客户端进行云同步。连结远程windows，可以调试本地局域网下的linux开发机，在多虚拟机环境下，你可以虚拟出一个软路由系统（假设它有一个公网IP，稍后会讲到），将其它虚拟系统中的服务通过软路由网关服务透出来。

其它方面,qnap的videostation支持rmvb streaming,但是它的andriod photo支持怪异的备份方式，它的notestation可以导出pdf，然而默认标题留空的话不会自动取内容第一句填上，这对用它作记事的用户是十分不友好的。当然还有更多可以比较的方面。

公网转发和IP盒子
-----

另外一个使mineportal靠近身边的云主机这个概念的就是公网IP盒子了。frp加云服务器的方式毕竟太不稳定，
盒子的原理就是一个高速转发器，网上可买到。可直接也可旁接。

在直接法下，盒子可以通过直接法连需要获得公网IP的设备（直接法下，盒子LAN口接待获公IP设备）。也可以侧接法，旁接法下，LAN口闲置。 这二种接法下，盒子的WAN口始终要接路由器某LAN口拉出来的线，因为无论如何盒子必须要联公网。

盒子设置页面是盒子访问路由器获得公网信息（比如IP）的手段。盒子只能通过一条网线和电脑直连访问到盒子设置界面（盒子LAN口），里面有二个页面:盒子设置和公网IP设置，有几点要注意：

>1)公网IP设置页面中的那个内网地址是要转发到的内网地址，可以是148.100（直接法，由盒子本身作网关），也可以（旁接法）下，由路由器作网页， 此时这里可以填路由器下可能分配的地址192.168.1.xx。此时，盒子只是一个路由器局域网内转发器（所以不用任何转发规则和端口定义， 就可以透出NAS对应端口的服务。因为远程IP并没有限制你对端口的使用）
>2)即使上面那个IP设置不断变更，通常也不需要担心地址错乱而reset盒子，因为其网关和访问地址永远是192.168.148.1
>3）如果一旦reset,盒子重置后需要更新，要断电20秒。否则会提示一直初始化中。 
>4) 将盒子界面的内网IP设成路由器 后。转发规则会失效。 所以这种情况下，不适合一种方式，就是达到ISP给路由器分配出公网IP的效果，然后你企图在路由器上作DDNS，转发。

在直接法下，盒子本身也是一个网关，，侧接法下，属反向代理，程序上，外网不能通过程序手段获取得到你的设备IP。 

最方便的做法还是侧接，因为盒子跟路由器放一起，上网设备只需连接路由器（可以无线），这种方式下也相当于给路由器配了个IP（注意这只是相当跟ISP给你路由器分配IP不同，下面会解释到） 所以，侧接法这名字和接法这个名字比直接法更符合IP盒子名字和直接隐喻。 是推荐的方法还可省去一条网线。

省事单一可同步文件夹：dsm支持personal photostation
-----

威联通的qts支持链接，可以将一个顶层共享文件夹的内部目录透出来成为一个新顶层共享文件夹，这样就可以将所有资料（无论是按用户逻辑归类的还是NAS应用产生的默认文件夹）归类到一个可同步文件夹下。

后来由于种种原因我还是弃用了qts转用了x86的群晖，因为虚拟机是我必须的，hdmi透出桌面不是我必须的。我必须的是dsm xxxstation给用户培养的使用习惯，如notestation用于记事不用写默认标题，ds photo支持自然方式同步备份手机照片。还有cloud sync支持百度云而qts仅支持部分。

其中最重要的，还是我发现dsm支持personal photostation，photostation设置中开启home photostation，且右上角用户头像中开启当前用户home的photo目录，在其中建文件夹当像册，建立到dsm桌面的个人photostation快捷方式，ds photo中选择那个向下箭头可以以personal photostation方式登录。这一切就跟使用根下的photo归类一样。你可以将archived photos归档和_current syncing photos同时做到home/photo下。

所以我现在新的统一备份和归类方法是：

>即home/cloudstation:_current,archived.docs
>home/notes:_current,archived
>home/photo:_current,archived （photostation个人）
>home/video:xxx （videostation没有个人）
>home/www:xxx (webstation个人)

以上archived都是经_current整理后打包的档。各个_current你可以理解为_2018lifedocs,_2018huaweimobilepics,_2018homepc等等， cloud现在单独跟其它station并列并不作最顶层，同样支持整个文件夹一起cloudsync,hyper backup备份，使用cloudstation下只需要上传备份某_currents即可。

-----------

关于利用类群晖或qnap的硬件作mineportal，atom等intel baytrail等架构功率和速率上完全可以类比移动平台arm。一些插座式主机，Marvell插座式电脑还可作手机伴侣，群晖也有vesa到显示器背面的产品。这些都是通常这二种NAS的变体。但mineportal一般用于服务环境，建议四核x86环境毕竟开虚拟机性能上至少四核内存越大越好。

至于客户端，你可以用vnc，云终端，或chromebook同样是低功率架构（群晖+chromebook是天然的搞创作，办公生产的绝配，这二种都原生web based，用vnc+一套键盘鼠标可以达到透出虚拟机桌面的效果，产生《利用chromebook打造无线同屏器效果》的话题）。当然这样的搭配就不要玩用于游戏了。玩游戏可以用专门的安卓手机,psp,win gbc等。甚至，由于客户端可以无穷定制和专门化，也可以有《利用七寸umpc打造programming pad》这样的话题。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338675/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



