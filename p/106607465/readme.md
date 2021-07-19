利用大容量可编程网盘onedrive配合公有云做你的nas及做站
=====

__本文关键字：onedrive打造你的网站图床大文件外链床，sync as network，网盘附件床,onemanager http解发__

在《免租用云主机将mineportal2做成nas，是个人件也可服务于网站系统》中我们提到将nas和个人网站合一的思路，，在《mineportal – 一个开箱即用的wordpress+owncloud作为存储后端》我们提到用网盘做网站图床甚至all webapp backend storage的思路，在《统一的分布式数据库和文件系统mongodb，及其用于解决aliyun上做站的存储成本方案》我们讲到过在阿里云上省事做站的成本考虑。

毕竟，我们应用软件的方式就是在一台机器上装个app，这种思路已经深入人心了，所以我们希望做站也一样（像安装应用一样简单）。碰巧网站应用的附件床是必不可少的，一个用户体验好的网站需要大流量和高速度，甚至大存储。这跟nas处理存储的方式和环境要求很相似(现在网盘能达到百M局域网速能代替nas)，所以我们希望它与nas和同步管理连接起来。（群晖有一个wordpress plugin）。这样服务端的网站数据也像手机端的照片，PC端的物料文件一样，参与到nas中心文件库的备份与维护工作中去。这样就大为省事省心了。


> PS：我们知道，以ecs为基础的建站成本很高。需要强大的带宽，这是个人承担不了的。于是运营商们提出了无服务器建站serveless，，这基本就是以前的弹性扩展，专门化的“网站云，云网站”。即利用oss做静态资源存储和展示，甚至包括cms中的前端静态网站资源，再利用云函数做应用托管，来弥补静态网站动态性不够的问题，所谓函数，也就是相当于一般网站/移动应用的plugin（这些相当wordpress plugin,,etc,,是函数集），它们帮你装好了各种语言backend，也提供了一个类似容器的主机环境，你免去了搭建服务器的工作，这应该也是我storage backend webapp的一种实现，最后利用cdn做分发流量租赁。你租用它们的oss(专门做站的网盘还有小鸟云这些??没用过),cdn,和函数托管能力，（业务逻辑）由你提供，当然你也可以安装他们提供的函数模块。，

但即使这样，其实综合成本上，一套下来也不会低。只是稍为省事。但是如果有了一个函数容器或ecs托管环境再搭配大容量网盘，情况或许就不一样了，网盘的线路是第三方的。不走服务器流量，网盘一般都是cdn加速的且网盘一切都是买断的。相当于为网站服务器准备了一个超大图床，甚至附件线路，sync as network线路,这实际上就是上面提到的更强大的oss替代。网盘如果能作为附件床再好不过，甚至可以当成个人nas用。

我们在前面提到在云上装黑果。本来我们的设想是利用云上osx能利用上icloud的能力，自建icloud且提供一个能运行icloud的托管服务的环境。Icloud是一个非常好的文件异备服务(多端同步防冲突算法非常鲁棒)，icloud是被设计供icloud产品系内部使用的，直接在icloudrive和国内环境下使用icloud速度很快托管在云贵，是一个非常好的小型nas（除了它不支持在线视频等一些局限）。但却不是一个好的附件床。它的外链分享（自2019起支持）是打开一张网页，点其中的按钮下载。目前还没有像oneindex一样的方案将它转成直链（能wget url大文件这种，能通过api展示markdown托管静态资源。）。况且受国内政策影响的,在中国，域名绑定一个国内空间就要备案。免备案的都是走国外路线的。
，分享作为外链全部导到香港,速度大打折扣。况且即使能用，icloud的空间也有限，也没有强大的api支持。不支持其它os开发。我甚至用过用icloud pages+外链分享来做站可是不够省事。

Onedrive是另外一个选项，Windows这二年跟云和linux靠得越来越近了。这一切源于在微软工作的印度小哥的决策，onedrive在同步算法和client app支持方面跟icloud没法比，除了价格，（买断制不带onedrive的单office系列要便宜点，只有订阅制的带onedrive 1t/6t的office365系列618，1111，1212，1231去淘宝买或买可以差不多半价200左右得到，它也有国内网络互联运营版的onedrive。也有官方版走国际线路，国内速度肯定好点。价格也便宜点，但国内和官方版接下来提到的api支持可能不一样，购买时看产品描述问清楚）。，另外它还有对webapp和各种语言的api，这样并不局限于是什么托管环境（vps,虚拟主机，腾讯云函数，herku），借此你可以得到存储在它当中的文件直接外链，配合云主机/虚拟机/云函数环境可以将它做成个人nas和网站图床（不知道免费版5g有没有api可用），这样的程序主要是一些php的，有oneindex,olaindex(这个程序是php cli命令行版，还提供基于aria2 cmd实现的云下载功能),等等。前者可做成web界面的网盘。后者可以做成命令版。其它还有onemanager,onelist,pyone,cuteone,sharelist,fodi,cuteonep。

其中onemanger（https://github.com/qkqpttgf/OneManager-php）也是一个php网站程序，较oneindex,olaindex更强大，它也支持互联版。甚至支持安装在云主机和腾讯scf上。下面通过它来讲解试验。


使用云主机：
-----

建好lamp环境，选择宝塔这样的会比较省事，上传https://github.com/qkqpttgf/OneManager-php代码，创好网站。添加onedrive盘，我们选择国际版，自己申请id和机密，会把你转向到这个链接:

```
https://apps.dev.microsoft.com/#/quickstart/graphIO?publicClientSupport=false&appName=OneManager&redirectUrl=https:%2F%2Fscfonedrive.github.io&allowImplicitFlow=false&ru=https:%2F%2Fdeveloper.microsoft.com%2Fzh-cn%2Fgraph%2Fquick-start%3FappID%3D_appId_%26appName%3D_appName_%26redirectUrl%3Dhttps:%2F%2Fscfonedrive.github.io%26platform%3Doption-php
```

程序提供的转向url中有OneManager&redirectUrl=https://scfonedrive.github.io&platform=option-php这样的一部分参数，可以快速创建，否则需要去https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade手动创建，保存你的应用机密，网页转向，选择php(官方提供一个带php composer的足够环境来支持你的函数)，把注册成功处的应用id也保存下来。以后也可在https://account.live.com/consent/Manage管理你的所有应用。

将客户应用id和机密填入到onemanager处，应用跳转，蹦出一个错，不管，直接进入onemanager url，成功，可以看到用你的域名加文件夹加文件名就可以形成对应文件的直接外链。而你也有一个小nas网盘了（可以上传，也可以下载，可以在线播放，但速度感人，最好是大量小文件，大文件分享获取有些吃力。不过像moeclub用于存镜像，然后港区ecs通过installnet装机不错）。至于如何加密，如何连接多个帐号用完6T(可以把这么多用户想象成oss的buket)。自行研究。

使用腾讯云函数机：
-----

腾讯云函数有一个免费额度的cloudbase套（包月免费自动续费），包括静态网站托管（如果要使用这个，cloudbase应用环境要转成按量，不过也照样享有一定免费额度）和scf，这就是我们上面的云网站，阿里云的弹性扩展一类的东西。

我们使用的是单独作为一个产品的云函数去安装。它会出现在cloudbase免费云函数列表中。去cloudbase中去创一个helloworld云函数扩展，进入https://console.cloud.tencent.com/scf/index会看到被分配到了上海区，下面就在这里安装onemanager。

跟安装在云主机上相比，除了不需要安装环境（云函数受自带language runtime backended），这里的区别，从下面步骤可以看出：

直接选择云函数服务区上海新建，命名空间default，另外一个是你的那个cloudbase云函空间，。我们看到php7.2函数模板里面就有onemanager。尝试选择它直接下一步部署，不进行任何设置直接点完成。 函数管理，解发管理，添加触发方式，选择API网关，勾选集成响应。访问API网关，其它就跟上面一样了。

现在尝试将其复制到cloudbase命名空间，在云函数在函数名列表中看到有个复制，但是无法复制到cloudbase云函空间，提示无法完成。只能在函数代码处先下载为zip包，在cloudbase新命名空间新建php7.2空白函数的函数代码处，删掉默认index.php上传zip包,（直接上传onemanager.git源码zip提示函数不正常，需要修正onemanager.git源码为tcf适用版本为https://github.com/tencentyun/scf-demo-repo/tree/master/Php7.2-SCFonedrive使用再cloudbase命令行上传）。代码中config.json中"memorySize": 改为256，选择http触发，触发路径/onemanager，但是执行失败，看函数日志，Object of class stdClass could not be converted to string in /var/user/function/common.php on line 90,原因不明，看源码像是路径处理错误。还可能是因为cloudbase的云函数只支持http而非上面api网关方式模拟http触发导致（而独立腾讯云函数产品 2019年12月起只支持api模拟http）。

不管了，反正都有免费额度，就使用单独云函数版吧。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106607465/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




