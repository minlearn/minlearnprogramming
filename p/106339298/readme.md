mineportal2：基于mailinbox，一个基本功能完备的整合个人件
=====

__本文关键字：mailserver backed new mineportal,邮箱附件外链，owncloud backend static web hosting,阿里云省事建站，如何借助mineportal低成本把虚拟空间+email玩成个人portal，one place webapps for personal use__

在《发布erpcmsone》一文中我曾谈到我缺少一种能发现我需求的网站程序 — 能把我日常要用的东西整合到one place且生态统一易扩展的东西,我以为odoo是这样的产品，但现在看来，它其实更像是一个groupware for team use，在《mineportal:个人云帐号/云资源利用好习惯及实现》和《发布mineportal – 一个开箱即用的wordpress+owncloud作为存储后端》我谈到适合个人用的网站程序选型 —- 比如那些个人用户对于互联网的起点性需求：比如邮件,IM，存储，网站和网站存储，等等一个人急迫需要而不用依赖第三方的东西，能整合起来kept in one place随身带走takenable away的那些，,基本上它们是一些网上能发面的破碎的实现品.

vs mineportal，我把它们称为个人portal2，其实所谓portal,它是一个联系线下到线下的入口。强调这个入口是尤为重要的因为它是portal所指，比如存储和邮件是这个入口以下的私密部分，当它用于网站时，存储可当图床，IM可当网站聊天框，还比如，它能真正当portal使用—类opensocial用一个ID作通往各大网站的social connector+自动注册以代替原始的电邮注册，手机号注册，—— 可以看出，这些不全是web方式也不全是线上的。—- 最重要的一点：因为还有很多功能要扩展，所以最后这些都要生态统一，且可被扩展。最好有清晰的appstack划分：

什么是appstacks:
-----
 
一般来说，用中间件的视角，可以将语言，应用，基建服务都实现为中间件。应用那部分可以做成appstack.
应用层的appstack中间件，及这些中间件规范出来的应用级开发发布支持：这些程序使得面向它开发出来的东西直接就是app，使得开发和发布变得micro app/service的东西。它的存在，使得软件的开发件和发布件之间有了一个中间层。
appstack中间件，可以使得语言系统中间件，xaas层可以与appstack层分开。并且，独立出来的appstacks可以做成组件，所谓组件，就是可以被开发层内欠调用的运行件，并且可以做成原生实现，这样执行效率很高。比如游戏有gameappstack，这样开发游戏扩展，就是写一些有限的脚本。
比如nginx是一种xaas/iaas，mysql是一种common storage appstack,,kbe是一种game appstack，blaaa
在这种情况下，那二篇文章中谈到ocwp有些局限，它仅能提供网盘和网盘存储，终归它缺少一些重要的东西如邮件,ocwp只包含oc+wp自身。从整个ocwp产品体系看nginx不算其appstack组成只是视为xaas/iaas，，虽然围绕oc可以扩展更多，oc有插件可以外挂external webmail client进来，但那始终是外挂。没有至关重要的邮件产品，不如做成自包体，比如将mail servers包含进来做成appstack.

为什么集成了mailservers的ocwp不是完善的portal
-----

那么，如果将 mail servers集成到ocwp呢？如果是内置的，我们可以直接在ocwp中加一套mail servers，如果是外置的，我们可以比如，让oc支持从php imap扩展中读取附件 — 比如，用fc_mail_attachments和mail attachments这样的owncloud插件将你的EMAIL空间变成网盘，我还看了一下如pydio imap也支持，这基于以下一种事实：imap协议可以允许文件夹里的邮件带附件，且邮件是天然的消息系统，把邮件当实时消息，我们就得到了另一种sns，反正，什么都能围绕消息体和附件展开 —- 就这二块就足够让email based成为一个personal portalware。在使用上，一些邮盘客户端如imapbox能做到同步（虽然并不是那么完善），基本上能用邮件收发模拟发贴。比如邮件列表可以做成内部论坛如googlegroup之类的东西，

很久以前，我曾很崇尚这类邮盘之类的结合体和imapbox之类的产品，可是它的缺点在于：它改变了应用方式。一切既有实现都颇为不完善。单纯以邮件为后端的模式也不能提供如网站托管这样的个人portal应用，比如没有www件的支持，它不能真正让附件变外链（上面的oc to imap插件只是将imap里的附件镜像到了其内），邮盘空间也不能hosting website。

其它例子和the last saver：
-----

再来看一些其它例子

“SmarterMail: A revolutionary secure email server with file storage, group chat, team workspaces, sharing and more”
 
缺点：虽然是appstack以下的，且功能完备,15.x及以下的版本占用资源少，但是它附件共享支持有限，不能供网站当文件夹图床用。而且它是闭源的。
 
“Like Syncany, Cozy provides flexibility to user in terms of storage space. You can either use your own personal storage or trust Cozy team’s servers. It relies on some open source software’s for its complete functioning which are: CouchDB for Database storage and Whoosh for indexing. It is available for all platforms including smartphones.”
 
cozy有点类似sandstorm，它不是应用，是paas,对nginx,langsys等进行了集成，不好归纳出一套appstacks。且cozy bug太多占资源太多。不过其内部apps有很多social connectors，这是亮点。
 
 
而一些纯工具性质的也不行，它们太简单不足于组成一个appstack及其下的app扩展：
如一些聚合工具，如thinkup php，it should be a standalone portal but not just a tool that can collabrate all these infratures.
或单纯的webdav,webdav is just a sync proctol, oc has its own general storage cheanism,,oc is more a groupware framework than a file sync server,,,we can build games around it
或单纯的客户端实现imapbox,what i need is not barely a php/phalanger imap attachment outlinker/saver prog.. in the totoal effort to mimic what all imapbox does
 
最后我找到了这个：

mail in a box:
Take back control of your email with this easy-to-deploy mail server in a box. … DNS configuration, spam filtering, greylisting, backups to Amazon S3, static website hosting, and … Mail-in-a-Box is basedon Ubuntu 14.04 LTS 64-bit and uses
如果以它为基础重新建构mineportal2，将基本满足一个人上网存储，简化登录，将所有资料和网上活动最大聚集到一处的基本需求。它的优点易于搭建虽然它本身只是一个工具，可是着眼于组成它的整个体系可分离出以mailservers为主的appstack和与oc为主的app，且功能合理扩展性到位：

1，默认是没有wp的，它支持从oc文件夹进行static web hosting，且支持在oc内部通过ownnote,note这样的app直接保存为.html通过static web hosting输出，有了oc支持的后端app支持，也这并不会失掉去除wp后减少的交互性(可以调用oc的聊天，评论系统等)。

2，而且，oc支持好多social connector自动登录，如果联接的平台越来越多，它最终会变得像个人版的passwords repos，谈到这里要谈到手机平台上的微信登录等，微信是手机APP方式的自动登录，跟web通过api跨网站登录的方式有点区别，然而，oc本身是比微信要强大得多的，要说有差异微信是以sns而oc是以存储形成生态，实际上微信小程序这样的东西，就是相当于oc的apps，而OC比微信要更有资格担起personal portals，这个”portals”的角色与任务的，因为微信缺少至关重要的一部分，就是它整个都是公开的。没有私有的存储部分。而且它一切都是线上。

—————

mailinbox在aliyun上用脚本安装会出现service postgrey start failed的bug，自己处理一下，还有每用户static web hosted的地址user-data/www/(username)可以做soft linking到owncloud具体用户下。还有sqlite是默认的数据存储方式改成mysql+redis可以大大提高体验。最后，整个程序是unix系的，或许以后我会把它搞成windows版本。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339298/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



