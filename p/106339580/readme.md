在tinycolinux上安装sandstorm davros
=====

__本文关键字：git更新失败tlsv1，源码编译nodejs，提取sandstorm中的davros为免sandstorm版本__

在《发布mineportalv1:ocwp》，《发布mineportalv2:odoo,seafile》中，我们不断提到“以中心存储为后端的webapp设计”，因为以存储为中心符合个人操作PC的习惯。对于服务器和运维人员也是一样，网站体APP也可以产生海量数据，对于迁移和备份是十分重要的，这种存储后端支持要么被集成在appstack中（像seafile使用专门的repo server,odoo使用postresgl），要么被app级自身提供如owncloud的图床(其实我更倾向于不使用专门的appstack组件的方式比如seafile的文件服务器，它破坏了logic server就是langserver的事实，复杂化了不应常变动的appstack单元，odoo用专门的文档数据库来做这个文件服务器倒是更顺眼一些)，也有程序从api交互级维护，我不知道buzz是不是这样的设计-忘了名了，一个社交聚合管理系统(维护一套应用协作)，更有一些程序从paas级促成这样的结果如sandstorm：

sandstorm就是一种以维护中心数据为一体的app管理程序设计，为了达到这个目的，它先提出一个paas，这就是sandstorm，它用类似docker的方式为每个应用准备一个沙盒环境，但在管理框架级---也就是sandstorm自身---它本身就是一个js应用，维护所有app产生的实例（grains）数据备份，打包，它的grains就是用来做集中化存储的，且各个grains可以共用用户验证机制这样可以进行API级的交互，还有它下面的一个app-davros:一个类oc personal cloud storage的东西，如果做起plugin来完成可以达到像oc一样的规模。

该如何看待sandstorm呢？它其实还是一种web os的东西。

继我们的《在tinycolinux安装chrome》，群晖就是这样的web os的典型 as nas os，而OC,sandstorm，也其实是类似群晖桌面的东西，它们都是用web ui做appstore和webapp管理的本质作用-类似native desktop shell，只是工作在不同的层次---而其实也差不多：oc纯php实现管理php apps,sandstorm纯js实现，但管理混合语言stack apps，群晖synology都是混合。如果说考虑与langsys绑定的关系，sandstorm和群晖一样可以管理应用混合语言架构的东西,可以装各种appstack和各种appstack下的apps，但它与oc一样也可以全是one lang webstack的:oc是php,sandstorm是js。

为了让它成为一个纯粹的类oc的只管理js web apps的项目，其实可以把sandstorm关于paas的部分截取去掉，（比如将其放在xaas层次的VM管理中做一个管理入口到sandstorm，这样sandstorm不光有apps,grains且有vms管理跨langsys,xaas,appstack,etc..），保证它的生态下全是类davros之类的js应用，共用一个类似mean的单appstack，且纯粹面向webapps

《elmlang》后，我们关注了大web的js语言，而且还将基于一体化存储后端的webapp系统的关注点从php,py移到js，那么这个davros，它就是我们的mineportal3。我们要做的，就是先提取sandstorm中的davros为免sandstorm版本先用起来因为它本身也是一个独立的单js app。分离掉与sandstorm的xaas管理层薄薄的联系就可以了。

在tinycolinux上编译安装nodejs和npm
-----

tinycolinux上gcc481最高最能编译7.10.1 ,8.0.0和8.0.0以上会提示ArrayVector(v8::internal::StringStream::FmtElm [])相关的错误, 最新的894要求gcc494，

sandstorm自身用的是nodejs8.9.3,官方使用的davros 0.21.7 spk中使用的nodejs6.4.0，所以在这里我们使用6.4.0版本,首先装好git，然后装好py，下载nodejs640其源码,cd到其中，执行：

./configure --preifx=/usr/local/nodejs && sudo make install

cd到/usr/local/nodejs,export PATH=$PATH:/usr/local/nodejs/bin，执行nodejs发现需要libstdc++高版本，把libstdc++.so.6.0.18(这个是编译cmake时也需要的库，参见以前文章)换到/usr/lib下，接着执行npm install -g git://xxx，发现调用git时不能下载https里的git repos内容，提示SSL routines:SSL23_GET_SERVER_HELLO:tlsv1 alert protocol version Completed with errors

这是由于最近2018.2.1github不采用低级的加密方法了，git依赖cur,curl 命令行依赖 openssl 库才能使用 ssl 和 TLS。当前一般认为 TLSv1.1 及 TLSv1.2 才是安全的，很多 https 服务器仅支持这2个协议，不再支持 TLSv1.0 及 ssl。但是 openssl 是从 1.0.1 才支持 TLSv1.1 及 TLSv1.2。系统当前安装的openssl-1.00.tcz+curl不支持。查看已安装的ssl和curl，执行：curl -V(大写）发现openssl是1.0.0k,curl是7.30.0

我也不想去其它的5.x的tinycolinux中去找了，自己编译吧。好像5.x的是1.0.2的去掉了sslv2v3的？所以还是自己编译安装吧。

我下载的是openssl 1.0.1src和curl-7.15.0.tar.gz,首先安装perl5,openssl编译需要perl5,cd srcroot,./config --prefix=/usr/local shared && sudo make install就可以（注意不是./configure）一定要加/usr/local，否则安装到/usr/local/ssl中去了，加shared可免去下列错误:x86cpuid.s:(.text+0x2d0): multiple definition of `OPENSSL_cleanse' ../libcrypto.a(mem_clr.o):mem_clr.c:(.text+0x0): first defined here

接下来编译新的curl7.30.0，./configure --enable-shared --with-ssl=/usr/local

查看新的openssl版本

/sbin/ldconfig -v
openssl version -a

查看curl是否引用了刚编译安装的1.0.1版本

curl -V(大写的），发现使用的是openssl1.0.1

现在git会自动使用ssl3,npm install -g git://xxx或https://可以用了。

准备davros代码并编译运行,失败
-----

现在准备davros,我下载的是https://github.com/mnutt/davros中的davros-ca480aea708d0e9ae4b63342a4583660609f331f的0.21.7 release,将davros的根中的所有内容全选，上传到/usr/local/nodejs根目录，cd到此

我们看到js npm的包管理还是蛮好的，每一个包都维护一个package.json，申明它向前依赖的项。应用即包本身，各个包组成一个树形关联关系组成一个大应用，davros作为大应用，可以看到其根下有npm用的根package.json,bower用的根bower.json,etc..

根据https://github.com/mnutt/davros的说明，先sudo npm install，但是发现奇慢，加tb的mirror吧npm install -g cnpm --registry=https://registry.npm.taobao.org,再sudo cnpm install发现快多了（这是在安装src root下那个package.json的依赖关系包括bower）。它可以代替默认的npm，匹配不到的它会从默认从github下载。如果有紫红色的就是出错的

接下来，sudo bower install后会提示找不到bower,把生成的node_modules/bower/.bin中的那个链接文件移到/usr/local/nodejs/bin中，并修改指向对应位置

然后sudo npm install 和 sudo bower install --allow-root，发现git出错：error: SSL certificate problem: unable to get local issuer certificate while accessing

git config --global http.sslVerify false一下会在home/tc/下产生.gitconfig文件，再sudo bower install --allow-root这下能继续了。

我们也无法去追踪到底安装了多少东西了。

然后按照https://github.com/mnutt/davros的说明，sudo PORT=3009 ember serve，发现ember也没链接进/usr/local/nodejs/bin中（在src root package.json中它跟bower一样是要被安装的也一路并没有出错），直接执行吧，不做了：sudo PORT=3009 node_modules/.bin/ember server，发现ember的确在后台打开了守护，根据github的readme.md说明，这时本地桌面客户端可以连接了。

但其实我们根本不用这样做，因为这个后台守护会耗尽内存， top中会看到内存占用一直涨，最终命令行也显示heap out of memeory，尝试失败！！

按理说，这里要ember build一次，之后会将ember一系列东西，包括davros src root的app文件夹下面的东西全打包在生成的srcroot/dist下一个davros打头的随机文件名中。是不是这样呢，我们也没时间追究了，只能换个死方法了，我们直接从spk中取来所有ember build好的东西：

直接提取spk的已编译好的davros运行,成功
-----

在另外一台机器上安装一个sandstorm，然后连上进入winscp,进那个spk的目录，我的是/opt/sandstorm/var/sandstorm/apps/e813a833d983fbc38d87da62ea461fa7/opt/app，全部打包下载，清空原来nodejs的根目录，重新/tce/nodejs460下make install一次，然后把新的spk中的包的内容全部上传到这里，./sandstorm中的launcher.sh弄出来到根，稍微修改下其中的路径，建立data,data-temp,samplefiles等目录，执行sudo ./launcher.sh(它其实就是nodejs执行根下的app.js)，注意8000端口不要被占据，成功！！不光oc的桌面客户端访问。网页端免sandstorm也可以进入。 可见它与sandstorm管理框架和ember build过程是没有太多导致运行失败上的关系的。

当然，这个免sandstorm是没有认证机制的，如果是自用的话，随便写个认证逻辑就可以，这个服务端比oc的服务端快多了。

 ----------

 下文《mineportal demobase总成：一个类sandstorm +sandstorm appsystem的东西》，这应该是在《tinycolinux上装chrome as 客户端》之大webstack对应于服务器的东西。服务于bcxszy教学和实用。

 关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339580/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


