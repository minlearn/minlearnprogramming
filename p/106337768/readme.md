使用群晖作mineportalbox（2）：把webstation打造成snippter空间
=====

__本文关键字：网盘作github空间，网盘空间作演示空间，网盘空间作code snippter程序学习空间，群晖当github__

在使用群晖作mineportalbox（1）中，我们提出了一些省事省心使用群晖的想法和经验（为资源设置合理的文件夹结构，设计单向同步的省心策略,etc..），我们还谈到：郡晖不但能用于媒体存储和同步，还可用于codesnipter hosting和同步空间，用网盘来存储数据和运行代码，“网盘即webos”，在整个系列的前面，我们不断谈到过类似概念：对于前者，文章《利用oc+wp打造backend storage oriented cms：oc静态网站空间》,《oc微博记事本》，都是企图将所有数据用文档存储来以网盘存储的方式呈现和同步的努力 ------ 网盘空间即webos fs,这相对好理解。对于后者codesnippter空间，文章《web:visual instant demo run and debugging》,《post as app，paas.engitor one as demolet engine》中的jupyter notebook即是一个好例子：它实际上把.pynb当成了服务端脚本使之成为web空间，web空间天然就是一种codesnippter空间，里面运行的codesnippter即是app,applet ------ 网盘空间即程序后端。

如此看来，codesnippter空间，这听起来像是语言后端+虚拟web空间或者baas+paas容器？甚至docker,git这样的东西，托管在github中的代码并不会执行，dockerhub呢？它更强调存储和运行单元的虚拟化和容器化，超越了仅仅需要同步这样的需求。----- 但其实空间和其中运行什么语言的程序，其实这些都不是质，我们只是追求“可存储为可同步的codesnippter空间”而已。所以一个网盘也是可以的。甚至把owncloud当git空间管理项目codesnippters也是可以的。----- 如此种种，不一而足。看似不一，其实都有相通点。而装配了webstation的群晖也可以是这种webos，它使用的就是虚拟主机概念（加上它自身就是个网盘）。

PS：backend storage oriented webapp，面向以存储为后端的webapp更符合PC的使用习惯，设想在群晖webos上存储后端即是网盘空间，存储在其上的媒体或软件媒体即软件，软件即媒体，都可以统一同步，备份，只是后者可以执行，被hosting，每一个codesnippter都可成为一个应用app，这样的空间+空间上的一份codesnippter as app，即是backend storage oriented webapp ---- 这一切都像极了PC。
还比如我们在前面提到的cloudwall，它以文档存储为FS，其上的js可以是文档也可以是代码app，可以在客户端执行也可以在服务端发挥服务端脚本的作用。所以cloudwall说它自己是webos有一定的意义。

1，如何省心使用群晖的codesnippter空间方面
-----

使用群晖作mineportalbox(1)中谈到了省心使用群晖的基础方面，如果那些只是基础,那么，如何依然还能省事省心地用好群晖托管程序的这一方面呢？本文即更进一步，拟讨论稍高级的这一论题。

如（1）文所讲，使群晖能同时存储媒体托管程序才是合理的。且要能统一备份和同步。好了，下面让我们开始利用群晖的webstation（群晖目前支持的一虚拟空间语言和WEB后端），来搭建一个wordpress.


准备工作：
第一步，安装官方的php5,7,mysql5,10,webstation,apache2,wordpress(它要求mysql10),phpmyadmin等套件，将wordpress,phpmyadmin默认安装在web下，在phpmyadmin中建立数据库，把wordpress安装好，确保一切都运行起来(把wordpress地址填成frp转发后的地址，最终能进入wp)，这只是准备工作，最终我们仅需得到webstation和wordpress,phpmyadmin的源码。phpmyadmin套件,wordpress，mysql10套件要删除掉（保持仅mysql5,webstation这样清希的套件结构，是为了节省资源，也是为了体现将webstation作为上述的codesnippter空间来承载php源码集的方式，比如承载我们下述过程中提取出来的wordpress,phpmyadmin源码）
第二步，wordpress源码，phpmyadmin源码全部从web下移过来到cloud->softs->www下。（所以其实原来的安装方式也是利用虚拟主机这个原理，只是它安装到了web下，我们需要将其移到cloud下的www新目录下统一和cloud下的->media备份）。
通过phpmyadmin备份导出mysql10的数据库，导入到mysql5的数据库，备份下/var/packages/phpMyAdmin/target/synology_added/etc/servers.json，然后卸载wordpress,phpmyadmin,mysql10这3个套件，


现在准备新的虚拟主机：
第三步，把servers.json上传到www,.htaccess从wordpress中放到www根，打开webstation，选择apache2.2,php5.6新建一个虚拟主机，端口81，删除wordpress/wp-include/wp-config.php中群晖新加的东西，否则到时它会调用81端口产生资源失效，而且把数据库调为调用mysql5的pid文件。虚拟主机目录指向到cloud->softs->www ，提示转换权限，注意群晖自动处理的权限不方便，所以我们还需要自动调下，否则无法编辑后保存，也会出现no input files等显示错误。手动调权限：www文件夹自身，和递归子目录权限都定义好为用户http，权限加你当前登入的管理员用户读写全控制。
最后，在frp配置转发文件中定义一个类型为http，local端口81，转发到xxx.xxx.com，然后运行。
成功，wordpress和phpmyadmin都正常运行。

2：继续把群晖用于管理snippter和code note:
-----

这不用我说了吧，继续新建虚拟主机，往里面写.htaccess，放.php文件直接设置即可。你可以视整个www为codesnippter空间，也可新建一个codesnippter与www并列以它为基础新建虚拟php空间。


PS:
其实我以前是不同意在群晖这样的mineportalbox上搭网站的，但现在看来，对于个人这不失为一种省事，打包带走，数据全在身边的省事方案，至少我们关站关掉电源即可。而且，我们做在mineportalbox上的网站可以仅是一个中转，比如上面提到的wp，那么做一个wordpress中转的意义在哪呢？
比如，平时你可在群晖的notestation写文章，然后发表到这里，让它跟外面的网站同步。
还比如，普通情况下，这样的wordpress做成的新站不利于收录，但如果你写的都是原创，就可以申明原创，可以作同步到百度熊掌这样的原创平台。就不怕权重高的网站抢你原创了。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337768/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




