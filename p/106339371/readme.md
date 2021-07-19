免租用云主机将mineportal2做成nas，是个人件也可服务于网站系统是聚合工具也是独立pod的宿舍家用神器
=====

__本文关键字：利用包含nsd的mineportal将个人pc打造成nas,apache的oc透露owncloud静态网站服务,发布portalbox，based on colinux and mailbox,一个网站和个人件，hostos与guestos的最佳组合__

 在《发布基于openerp的erpcmsone》时我曾谈到我用一台ECS作为我的服务器，那里我同时说到我把PC当成了跟手机一样的终端，一个中心多个终端这种星型设想本来也符合很多现有IT布局的情形 --- 比如NAS使用的场景。但云主机空间存储不大，作为一个需要大量存储的用户，虽然阿里云几十G的默认空间当网站空间+网站存储空间也是可以的，但是要当成nas存储大量资源显然不够用。

那么，能否把PC转换成服务器而不是终端，当成NAS呢？比如:可以直接在个人PC上构建这样一个服务器中心(这台PC可能在你上班时租住的宿舍里，可能在你家里)，再买一些大硬盘（现在硬盘的价格也低了，个人购置一个2-4T的存储基地也不是什么不可能的事）进一步地，仅把pc富余磁盘空间当静态存储/同步服务器空间终究太无趣，我们还可以把它当网站空间和可安装各种动态扩展的服务器空间。比如我们可以在其中装一套nas管理OS，这样大空间和大存储都有了，俨然一个paas了。(当然，装在不常在线的家用NAS上始终跟ECS这样的环境不能类比，比如家用nas当网站服务器是不能持续24小时提供服务的，但我们可以做一些强化达到同样的目的，以下会谈到)。

首先是为这样的nas选一套OS，在《一个设想：基于colinux，是metaos for realhw and langsys，也是一体化user mode xaas环境》我们还说到hostos/guestos不宜选virtualbox这样的东西(why not virtualbox,why not sandstorm paas,why not docker,利用guestos as container无限开,colinux配置文件中有资源配额字段，etc..)。而其实每一台pc都有服务器和客户端的双重角色，一台服务器也是如此，比如你需要一个guestos，且在hostos中管理guestos的能力，基于colinux的mineportal刚好满足和紧密对应这些它以带GUI的windows为shell以monolith一块的linux整体为服务器，这样就省去了通过类似web管理lnmp件的方式来管理服务器。在开发机/服务器上，这种布局也很流行(vagrant)。基于maininabox的内置程序有lnmp等，还提供mail,cloud storage,static website hosting等等，可以为这个nas提供最基础的个人件设施,maininabox也带dns服务器，是建立nas的重要条件。

所以我们的总需求和设想是，不仅把PC当NAS，而是把PC当成集服务器和终端的双重体，借助colinux+mineportal2的方式，将其功能发挥出来，只不过这次不是装ECS上而是装装在PC上，当然我们不能做到像黑群晖那样完善，我们就现成的地用mineportal2，把这个portalbox打造成nas而已。

步骤和强化
-----

1). 磁盘布局：

现在的笔记本一般有双盘位加一个固态SSD系统盘，三个盘，我们可以将作为hoster的windows系统安装在SSD上，其它二个2T的盘，一个格式为NTFS，为D盘，一个格式为EXT3，在windows下隐藏，往其中灌入ubuntu14.04.vdi那个文件中的所有内容(先启动现有的colinux，挂载上2t盘，格成ext3，然后cp -ax / /2t，这里/是现有colinux根cobd0，/2t是2T盘挂载的cobd1，注意拷完后在新的/2t中修改fstab调整成正确的cobd xx)，2T的盘格成ext3比较费时，记得加T largefile，这样就快多了，把colinux放在C盘与windows,user,program files等放在一起，安装colinux为服务(那个colinux-install-service.bat，然后服务管理器中由手动调为自动)，修改好ext3分区上/usr-data/owncloud/config中的oc地址，再安装个owncloud客户端连上https://localhost/cloud随机启动，将C盘做成hwindowsgcolinux.gho的备份放在别处。

不认4G内存的处理和解决方法：

windows2008r2之后没有32位，而这之前的所有32位都存在不开启pae识别不了4g内存的问题，经测，win2008 32位在4G内存机上会认出完整4G内存（显示在我的电脑属性页），但这会造成colinux启动蓝屏，这个时候，bcdedit /set {current} truncatememory 3221225472，把4G截断为3G就可，但win7 32就会认出标准的该截断3G左右的内存，在其中执行colinux也往往不会蓝屏，win2k3也可以。

在使用的时候，D盘的文件可随时备份到2t盘所在的nas内，供远程访问，这样整台PC就是同时终端和服务中心了，且避免了设置raid备份的需要，因为windows中的那个2T和服务器整理的文件是有二份的，只要你不删，是可以随时双向恢复的，且笔记本的电池可以当ups用。你可以用https://localhost or 局域网IP/cloud这样的方式访问这个nas,当然这仅限本地，和局域网环境里使用手机终端streaming video,pics的情况

2). 然后就是使内网nat中的colinux对外开放(家用adsl路由方案往往如此),和开启你PC上的远程桌面：

我们可以在家用路由器上用静态IP设置NAS所在的那台PC，然后将这个IP透露成dmz主机，再开放3389,80,443等转发端口。最后在路由器内部申请一个oray的花生壳 DDSN域名指定到这个IP挂上。这样就可以在异地远程桌面和登录/同步nas https://你的花生壳域名/cloud 中的文件了。

3). 一些强化：

为了不用常开机。你可以设置网卡远程唤醒，或买一个极路由带远程开机助手的，不过这往往有点麻烦，最省事的方案不如买一个智能WIFI开关插座，设置电脑通电就启动，然后远程控制通电/断电/开机。

你还可以自建DDNS服务器，代替花生壳在上述方案中的作用。

as pond，以促成可用的网站系统:
-----


用以上这里的nas方案做偶尔远程开机的同步服务器是非常理想的，因为同步毕竟不需要一天24小时开机，但至于用nas做网站系统就不能保证持续服务了，所以我们的方案是：

不直接用这里的nas当网站托管体，而是用这里的nas当远程网站的同步体和辅助体。什么意思呢，远程网站可以是一个很小的php虚拟空间装上一个owncloud，并通过oc的external storage插件把它外挂入我们的这个家用owncloud center repo，这样，我们就把这个虚拟空间当成了类手机一样的终端，参照我的《在owncloud上hosting static website》，在远程虚拟空间有更新时，我们可以把新增/改动的那部分手动复制到家里那个oc center repo，进行二个repo之间的对拷，当然，手机oc客户端是支持auto upload的，这个需要我们手动拉一下文件。

这就是说，静态网站系统生成的那部分才是我们需要备份的，至此，我们可以对静态网站和动态那部分作个概括了：一个网站系统的动态部分不需要透露给用户使用、只供管理使用和功能使用，静态部分仅供展示的，就是我们这样的oc托管静态空间。至于在虚拟空间端，我们需要设置一下.htaccess（如果是apache）：

apache的oc透露owncloud静态网站服务

```
<IfModule mod_rewrite.c>
RewriteEngine On 
 RewriteBase /

 #绑定域名到具体目录，事先将owncloud的data弄得跟owncloud目录并列，将data设为0755，去除oc的htaccess 
 RewriteCond %{HTTP_HOST} ^www\.shaolonglee\.com$ [NC]
 RewriteCond $1 !^(owncloud|oc)
 RewriteCond %{REQUEST_URI} !^/data/minlearn/files/~www/posts/ 
 RewriteRule ^(.*)$ data/minlearn/files/~www/posts/$1?Rewrite [L,QSA]  

 #静态html,记得在管理器中把index先htm第二php
 RewriteCond %{REQUEST_FILENAME} !-f
 RewriteCond %{REQUEST_FILENAME} !-d
 RewriteRule ^([^\.]+)/$ $1.htm [NC,L] 
</IfModule>
```

终于，我们知道为什么把这套方案称为pond了（我们不知道为什么喜欢pond这个词，感觉它表达的意思是指去中心化之后形成的一个一个孤岛，本地数据中心，联网时可以在线，不联网时就不用对公透露，P2P的个体），去中心化免ECS投入是这个nas的中心价值。所以题目称，这是一个免租用云主机将mineportal2做成nas，是个人件也可服务于网站系统是聚合工具也是独立pod的宿舍家用神器。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339371/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



