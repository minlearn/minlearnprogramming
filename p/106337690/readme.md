使用群晖作mineportalbox（1）：合理且不折腾地使用群晖硬件和套件
=====

__本文关键字：群晖,mineportalbox__

在前面《免租用云主机将mineportal2做成nas》《使用树莓派和cloudwall打造个人云》中，我们从软硬结合的角度，提出了二种代替云主机，使云设施不再受远程托管的方案，毕竟，数据在身边才是安心的，这样的一个可同步体和手边的云盒子它更符合我们一直想给它定的名字：mineportal 为什么呢？因为它是优先在内网使用的，只有在对公服务时才需要寻求内网穿透的这样的方案，所以更适合portal to public这个“portal”。----- 因为现在加了盒子的概念，所以我们进一步叫它mineportalbox。

业界早有这样的作品，比如群晖，比如智能路由器系统（可装APP的），在前面我们已讲到过群晖与各大paas的对比（默认我们指群晖的dsm），群晖是GPL开源的,其源码在sourceforge上可找到，应用也是可源的（而且还有synocommunity支持一体crosscompile构建spk的开源方案），本身并没有特殊的技术，遵循从我们bcxszy partii系列1ddcodedemobase中提到的从定制linux开始，准备容器，appstacks，apps这样的方案和路子。(它像oc塞到树莓派这样的方案，而oc没有对linux定制,sandstorm这样的东西最接近群晖dsm，但是也没有配备硬件,etc..区别仅此而已)

>synology是最好用的nas硬件(存储方面)和paas主机环境（软件方面），虽然贵但它是以好用出了名的，最流行的用法就是架网盘，较同类产品它更好用和美观。符合真正的产品级体验。
>它的套件也很人性和全面，比如它的note station,编辑界面能自动消隐标题这样可真正当note用，且同时可视为blog/snippter/note工具（textallinone）。它的multimedia station(photo,audio,video)可以进行流媒体播放。webstation中的wordpress也有直接从共享中插入图片到贴子的能力，就不用再hack出一套《wordpress以owncloud为图床》这样的方案了。

可以把群晖理解为：手机伴侣，本地型云服务器（VS 远程型VPS），装机恢复器，同步器。webos，能运行php codesnippter的学习空间，甚至上升到宅男神器这样的东西。

而这一切都是以足够不折腾地的方式下去进行。不折腾原则就是让黑盒的该黑盒，不涉及到对群晖二次开发或改造，满足使用群晖现有功能和它对我们塑造的同步数据的习惯，比如最终，一个人可以不再需要PC或电脑。仅chromebook+手机+装了足够丰富应用的群晖来提升前二者的离线能力这样的三件套就可解决工作生活绝大部分问题即可 -- 这是我们的追求之一。

这就需要讲到如何不折腾地使用群晖。


1）我们能用好的方面：使用好群晖的套件和同步
-----

我一向对云有二种要求：群晖要像cloudwall一样，可存储数据，可运行代码。这二部分可以统一同步和备份。所以我将同步文件夹分为media,softs，_current三个文件夹，media就是备份notestation,photostation,ffsyncserver里面的打包或导出的备份数据,chatlog数据，而softs就是webstation里面的代码，_current是收集每个终端来的临时同步数据，其下设各种子目录，比如frommobile，2018work，这样分别同步时指定_current/终端文件夹，就可以了，比如在家的时候仅同步_current/2018home，在公司时同步_current/2018work,从云端收集的东西放到_current/fromcloud。

这里注意不要使用drive而使用cloudstation，因为只有使用cloudstation才能获得cloudsharesync支持。而且以上用于统一备份的共享文件夹不要放到home/cloudstation中因为sharesync不适用于home，我倾向于把以上放到一个顶层的cloud共享文件夹中对应cloudstation这个名字(与那些顶层的videostation使用video共享文件夹,photostation使用photo一样，这些可作为_current之外又一外层的临时或fromcloud目录被sharesync使用同步到另一台nas)

可以双群晖同步(sharesync)，也可以对云同步(cloudsync加密备份到外部云)，后者可以免除双盘位群晖做raid的必要。当然对于个人来说是这样 - 我买的群晖就是单盘位的ds118。

同步的时候不要去动目标端。同步模式选择为仅上传。本地有同步不要进file station操作数据,其实我一直觉得同步就是个为概念，这就不得不谈到《发表cloudwall中》我们大谈特谈到的同步与云设备的关系。深刻思考这里面的关系你就会发现，永远不要依赖双向同步。仅单向备份同步。可以省却不少麻烦和版本冲突。

2)hack和强化群晖系统？我们目前还不能很好控制的地方
-----

内网同步和nas云存储是群晖的基本作用。群晖是不完善的，对群晖进行深度定制和更改永远是疲于奔命的，业界有黑群晖这样的东西企图使之成为一个通用产品，主要是定制了群晖的kernel和启动技术，使之从不同硬件启动。我也曾解决定制群晖kernel使云硬盘从sda开始认而不是vda开始，实现过把黑群晖装到vps中。甚至考虑过是否一定需要分离式的群晖与终端设备这样的问题（比如asus有一种平板PC二合一电脑，主机是i5系统可装黑群，可分离的屏幕装的andriod可作为终端这样的组合，我还设想过一台大工作站里面有二台电脑，一台装chromebook可取出，一台装黑群固定，二者共享的主机是docker的），这些都最后觉得没有现在群晖一个黑盒子+各种终端好用。

当然在上面提到的DSM的一些固有不便的确存在，比如以上套件其数据有的不能在homes中，video不能像photo一样放到home,sharesync不支持home，考虑将顶层的video,photo做到cloud下统一sync（像moments和drive一样），webstation只持php不支持java开虚拟主机等等，这些还是值得解决的。

当然还有一件事是最迫切需要做的事情。那就是在可选和按需文件下，向外网发布服务使它更像个cloudportal和云主机：为群晖出外网和提供SS支持（这样局域网不用装SS客户端仅http代理即可），以DS118为例。下面谈到的方案适合最苛刻的内网环境（路由器甚至不能DDNS，分配的地址也是ISP的内网IP，只能穿透）：


我们可以下载arm64版的frp,在穿透云主机上装x86的服务器程序(打开群晖ssh登录设置好权限)，打开相应安全组端口，然后把arm代理端装在群晖的cloud->softs->某目录中，配置好，并在群晖控制面板中指定其开机启动:

配置文件：

```
frp
[common]
server_addr = mineportal.xxxx.com
server_port = 7000
token = 这里填服务端设置的token
[22]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 7002
[5000]
type = http
local_ip = 192.168.1.118 路由器上已固定此静态IP
local_port = 5000
custom_domains = mineportal.xxxx.com
[80]
type = http
local_ip = 192.168.1.118
local_port = 80
custom_domains = www.xxxx.com

cow
listen = http://0.0.0.0:1080
proxy = ss://aes-256-cfb:xxx@ss.xxxx.com:8000 ss.xxxx.com为ss所在服务器域名/ip
```

任务计划->开机脚本:

```
frp
cd /volume1/cloud/softs/tools/frp/
nohup ./frpc -c ./frpc_mineportal.ini &
cow
cd /volume1/cloud/softs/tools/ss/
nohup ./cow -rc ./rc &
```

穿透云主机开的安全组：

```
frp
7000:frpserver
7001:frpadmin
cow
8000
```

设置邮件通知，接收命令运行后的结果。

这样，平时走在路上手机设置quickconnect id，回到家同步会自动转为局域网同步方式。想快速访问或外网看视频就用frp方式。平时工作产生的文件也小用quickconnect足够。
群晖功率小，可以一直放心放在家里或宿舍，要租那种移动不限上传带宽的个人宽带。

-------------

之所以自《DISKBIOS2》隔了三个月才发表这么一篇文章，是因为我已经学会使用群晖和满足地使用群晖了，还是那句话，好用的东西不要再去折腾。

下一篇《使用群晖作mineportalbox（2）：利用工具编译群晖spk-以ffsync为例》，整个1ddcodeanddemobase将揭示如何一步步将一个类似群晖的东西用dbtinylinux,cloudwall,rapbian pi三者来具现的过程。与minlearn programming一起组成bcxszy



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337690/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



