在dbcolinux上安装cozy-light
=====

__本文关键字：js个人云存储,cozy,node-legcay和谐模式__

在前面的《appstacks》,《apps》系列文章中，我们大力涉及到带存储支持的云程序，与语言选型放一起，我们写了py的2个(seafile,odoo)，php的2个（owncloud,mongopress），js的一个davros。并一直扩展它们的意义，认为它是一种小可视为与common storage based webapp合作(ocwp ownnote,mongpress,odoo),大可扩展为paas,webos的东西(sandstorm,cloudwall)，在《设想：cloudwall与树莓派》一文中，我们又把cloudwall与通用移动硬件的树莓派结合，提出了真正云硬件的概念。

这类OS应该仅是聚合？还是应是一个paas虚拟化结合的东西？

拿sandstorm来说

在前面《在tinycolinux上免sandstorm安装davros》时我们谈到了sandstorm和它与群晖OS等WEBOS的对比与意义:它提供了一套UI SHELL管理程序的安装,这是它的webapp聚合方面，davros就是sandstorm的file app,相当oc的file app,oc可以将一个external app加进它那个后台管理，而davros也是这样的方式被加进sandstorm的，因为它独立也能运行。

而它其实也是作为PAAS存在的它包装了一个node appstack(meteor)，却允许任何程序如php等安装入其中，它的PAAS还在于它的虚拟化，其实我之前一直很抵抗sandstorm的，它跟docker一样用到了分层文件系统这种虚拟方案，但其实sandstorm主体是没有分层文件系统的（它不管理虚拟机层的虚拟化iaas，离线的vagrant与它仅有SPK格式导入这层联系,也不属虚拟），它的grains才开始用到了一点虚拟化且仅仅是分层文件系统。(但其实仅是虚拟文件系统就能做好很好的PAAS了，比如docker,openvz）

换言之，sandstorm这样的东西才是一个全面工程，它涉及到了xaas,langsys,appstack,apps的改造和利用，即便它是一个轻量的xaas：跨xaas,langsys,appstack,apps的东西。，davros也需要sandstorm才变得合理，。但是因为sandstorm的编译涉及到一系列中心，我并不打算写文实践它。

值得一提的是，为了将这一切上提到OS和硬件层面，我们提出了dbcolinux慢慢将其打造成云OS,如将linux kernel作为共用的核心和装机中心，将/usr/local分给各种用户就可以打造openvz这样的东西。在《发布DISKBIOS》《/system,/usr分离式文件系统的linux发行版》中，让它直接管理虚拟机或实机装机,这种装机还考虑了运营对接到应用中的各种角色，后来我们的发布类文章都转到这个版本上,,我们甚至关注了对couchdb的使用甚至rapsian pi，让云OS寄托于专用可移动硬件。

好了，说了这么多，作为js personal cloud且采用cloudwall的另一个例子，现在来看我们的cozy:

cozy其实在《发布mineportalv2》中我们早提到过cozy,就像sandstorm也被我忽略一样cozy也是，不过cozy现在好像整个重构了。直到第一眼看到它的简化版：cozy-light我感到很惊艳，因为它支持的app中包括了html5 game app. 

它也有paas成份,聚合的webos成份：

Cozy is a platform that brings all your web services in the same private space.  With it, your web apps and your devices can share data easily, providing you with a new experience. You can install Cozy on your own hardware where no one profiles you.

most of the apps are runnable without Cozy Light

cozy也使用了pouchdb,couchdb的那种replicate协议是用来取代http的，，，默认加入同步网络的节点满足这类协议的，，，甚至都省了传统BS云同步中的同步终端，它们是满足协议即可当同步器/终端也可当同步中心。所以,cozy也有同样的功效，但是它在server端用的是leveldb。

好了，下面开始尝试在dbcolinux上安装它: 

安装启动cozy-light
-----

cozy-light好像2016年之后没人维护了，它的最新版本是0.4.9，相反它的APP在维护就够了，安装cozy-light分为安装cozy-light和各种支持APP支持,由于这二部分不是同步更新开发的，涉及到相同的东西有时会二处有不同的版本编译需求，比如pouchdb-4.0.3.tgz在app和cozy部都会被安装一次，都会用到leveldb，一个是120，一个是114,要找一个兼容这二者的js，我选择是的0.12.18带npm2.15.11，否则能编译完cozy-light是处处充满陷坑，稍后会提到为什么这么选.首先，node0.12.18安装https://nodejs.org/dist/latest-v0.12.x/,再装git,由于node 0.12.18属于老版本了，我们需要为/usr/bin/node建立一个shell wrapper开启它的和谐模式，否则会出错，把node重命名为nodejs，/usr/bin下新建以下内容文件并加起执行权限，引用nodejs：

 
```
#!/bin/sh
rdlkf() { [ -L "$1" ] && (local lk="$(readlink "$1")"; local d="$(dirname "$1")"; cd "$d"; local l="$(rdlkf "$lk")"; ([[ "$l" = /* ]] && echo "$l" || echo "$d/$l")) || echo "$1"; }
DIR="$(dirname "$(rdlkf "$0")")"
exec /usr/bin/env nodejs --harmony "$@"
```

npm install cozy-light -g会自动从github下载0.4.9到/usr/lib/node_modules/cozy-light，我在香港主机装的，所以外网速度快,/cozy-light/node-modules有它引用到的submodules各个submodules有它subsubmodules，node的modules就是一个树形结构，没有ln这样的引用，同一个工程不同的部分引用相同的模块的不同版本会重复存在，这也就是如上为什么一个项目要选一个兼容node版本的另一原因。不指定 -g会安装到PWD，编译过程中会调用node-gyp编译leveldb120,出了一些warnning:gyp WARN EACCES user "root" does not have permission to access the dev dir "/root/.node-gyp/0.12.28"，但是没关系,安装正确结束会输出一个cozy-light的模块树形表,直接启动它建立到/usr/bin/cozy-light的文件,cozy-light -p 80 start，启动失败,以下错误在设置了和谐模式后依然存在:

```
/usr/lib/node_modules/cozy-light/node_modules/pouchdb/node_modules/request/node_modules/hawk/lib/server.js:506
            host,
                ^
SyntaxError: Unexpected token ,
```

目测是request版本问题，查看其所在安装目录，发现安装的是最新的版本可能需要降级，我们用自定义位置的安装法：在具体模块树级层次中运行npm install。不依赖整体-g:打开/usr/lib/node_modules/cozy-light/node_modules/pouchdb/package.json，将"request": "^2.61.0",改为"request": "2.68.0"，为2016年1月的版本，删除pouchdb/node-modules下的request，进入/usr/lib/node_modules/cozy-light/node_modules/pouchdb/下执行npm install，再次执行cozy-light -p 80 start 成功。cozy-light再次启动会有bug，cozy-light stop后再start也不行，最好重启一下。

但是挑战不是这里，挑战和难度是安装app:

安装personal cloud distro
-----

cozy-light install-distro personal-cloud

apps全被安装在于./root下，/root/.cozy-light levelDB的数据都在这里,这次node-gyp编译的是leveldb140，有出错，整个过程中，我先后尝试过4.x-latest,5.0-latest,6,0-latest，都有出错:nan_implementation_12_inl.h error: no matching function for call to ‘v8::Signature::New，追踪一下，依然是版本的问题:time@0.11.1'引用的nan 1.6.2,仅跟0.12适配,这也是为什么我选择0.12的原因，安装其它app或distros时，也会有其它的问题，app/distors安装跟cozy-light一样，受上面说的工程各层次级引用不同nodejs版本的原因导致出现node-gyp将库链接到不同node版本出现问题,在0.12下以上personal cloud distro全程通过。

还存在一个warning : An uncaught exception has been thrown:{ [Error: spawn ENOMEM] code: 'ENOMEM', errno: 'ENOMEM', syscall: 'spawn' },要打开swap参见我以前的《在tinycolinux xxx》文章增加swap部分

以上personal cloud distro只安装了tasky,contacts,simple-daskboard,,等几个app，安装一下files:cozy-light install cozy-labs/files,启动cozy-light后为其设置密码：cozy-light set-password,启动和进入files app时会现如下错误：

```
An error occurred while initializing notification module -- Error: connect ECONNREFUSED
[Error: No instance domain set]
Error: connect ECONNREFUSED
```

相信不难解决。自己解决。

-----------------

files app是cozy,cozy-light共享的，所以也应该可以用同样客户端同步吧，另外cozy有一个cozy-fuse,都可值得试一下。

关注我。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339565/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




