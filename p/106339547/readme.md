在tinycolinux上编译seafile
=====

__本文关键字：tinycorelinux上从0源码编译seafile，uswgi方式配置运行seafile__

计算机科学和编程艺术起源于西方，在基础建设级很难发现中国人的建树，比如在C系相关的系统领域国内是没有什么作品广泛使用并让别人记住的，，但一个有趣的现象是，py域和应用域中国人异常活跃，且有不少佳品的，比如coco2dx，还比如我们要谈到的seafile，《在tinycolinux上编译odoo》一文中我们把曾odoo称为mineportalv2 - 它是groupware，vs odoo，seafile更接近personalware，其实更适宜用来打造mineportalv2，mineportalv1 oc只是一个复杂的图床加面向同步的webdav支持，而seafile有独立的fileserver，支持免文档数据库的切片文件系统，独立的standlone fileserver targetting storage domain logic implented as enginx appstack componet也有利于我们研究将其与enginx中的其它部件集成及深入《发布enginx》一文中的课题研究，且程序实现上鲜明的c+py混合编程特征和综合web+websocket的自然混合程序设计，更适合被用来作为教育目的，当然它不像OC一样仅需要虚拟空间就可以运行，只是这在云主机低价盛行的今天也不是什么大事了。因此接下来我们在tinycolinux上一步一步编译它：

编译seafile的五大件：
-----

我们首先编译出GCC481和CMAKE,python+pip,nginx等，按《在tinycolinux中编译cling混合c和py在线学习系统》中说的一步一步完成，且准备gcc的autotools支持和git支持：

autogen.tcz，automake.tcz，autoconf.tcz，libtool.tcz，intltool.tcz，perl5.tcz，git.tcz，openssl-1.0.0.tcz

然后编译出五大件，我下载到的版本是:

jansson-2.10.tar.gz(一个json解析库,C项目，cmake或autotools构建)

libevhtp-1.1.6.tar.gz(一个强化libevent的http库,c项目,cmake构建)

ccnet-server-6.2.5-server.tar.gz(seafile 自己的rpc库，c和py混合项目as py lib，autotools构建)

libsearpc-3.0-latest.tar.gz(seafile rpc库，c+py混合项目as pylib,autotools构建)

seafile-6.1.1.tar.gz(seafile的,c+py混合项目as pylib,autotools构建。)

seafile-server-6.2.5-server.tar.gz(seafile负责文件传输的业务服务器,c+py混合项目as pylib,autotools构建)

seahub-6.2.5-server.tar.gz(纯py,django app，seafile的前端部分)

按依赖和先后顺后编译，使用到autotools一般都是先sudo autogen.sh，然后./configure，如果sudo autogen.sh之后产生不了makefile.in基本是libtool的问题，确认安好libtool.tcz解决它，一一./configure --prefix=/usr/local/seafile之后，基本都能完成，使用到cmake一般要shadowbuild，即sudo mkdir b到src下，然后cd b,sudo cmake .. && sudo cmake build ..，其中evhtp要sudo cmake -DEVHTP_DISABLE_SSL=ON ..，libevhtp-1.1.6.tar.gz中cmakelists.txt中取消三个test的编译需求。编译configure或link过程中的时候会调到下述tcz：

acl-dev.tcz,acl.tcz,bzip2-dev.tcz,bzip2-lib.tcz,bzip2.tcz,curl-dev.tcz,curl.tcz,expat2-dev.tcz,expat2.tcz,fuse.tcz,glib2-dev.tcz,glib2.tcz,guile-dev.tcz,guile.tcz

libarchive-dev.tcz,libarchive.tcz,libattr.tcz,libevent-dev.tcz,libevent.tcz,libffi-dev.tcz,libffi.tcz,libltdl.tcz,liblzma-dev.tcz,liblzma.tcz,libssh2-dev.tcz,libssh2.tcz,popt-dev.tcz,popt.tcz,vala.tcz

基本上，，都可以在4.x的tinycorelinux tcz repos中找到。自己整理一下对应关系，假设在第一步我们上述五个除seahub外都是安装到/usr/local/seafile的，所有成功结果会是这样：在/usr/local/bin下产生各种bin，在/usr/local/seafile/lib/产生ccnet,seafile,serpc的so,la，甚至在/usr/local/bin中也产生了seafile-admin：没有py后缀shebang为py，作为脚本使用)。这个脚本很重要，下面细说.

安装配置seafile并用nginx+uwsgi方式启动：
-----

首先创建一个仓库（相当于odoo刚装完或重新配置时，要进入web/database/manager删减数据套件一样），seafile-admin就是用来产生这个套件的总工具，并负责调用seahub根下的manage.py来启动，下面我们用官方方法-即seafile-admin来产生套件并启动它：

在任意目录新建一个data文件夹，然后产生data/seafile-server/seahub的空文件结构，把五大件中的seahub改名替换/data/seafile-server/seahub中的seahub，四大件要么作为后端，要么sudo make install到并作为python lib，seahub中也有一部分要作为python lib，因此，export PYTHONPATH=/xxx/seafile-server/seahub/thirdpart一下，除去所有这些不可见部分，此后的seahub就相当于整个seafile website了。------- 现在，可以执行产生数据仓库(我们把它称为数据套件吧)的总脚本了，就是那个seafile-admin setup，回答所有问题后发现正确配置完成，pip install gnicore后即可访问，我们看到帮助文档中配合nginx是转发gnicore的数据，现在，我们将django的这种方式，换成nginx+uwsgi，去掉gunicore的必要。这实际属于django nginx uwsgi搭配问题。

首先，我们有如下发现：/usr/local/seafile/data/seafile-server/seahub/seahub下有一个wsgi.py和settings.py，这符合我们在《发布odoo》中用nginx+uwsgi将其启动的改造方式，也就是说，它可能天然支持纯uwsgi且seafile也保留了这种方式，那么究竟是不是呢？

进一步通过观看seafile-admin我们进一步明确了这种设想：它负责配置逻辑的产生（django app settings），且它调用的manage.py仅是一个wsgi.py的wrapper（为了seafile-admin中统一gunicore,fastcgi,etc..），所以，在seafile-admin->manage.py->wsgi.py的调用路径中，这样seafile-admin既是产生套件的工具，也用于统一启动，而原本这一切：用于seafile-admin中读取配置的部分settings.py+负责启动的部分wsgi，在无外头wrapper即seafile-admin情况下，它们是分离直接放进seahub根下的settings.py和wsgi.py中的:

现在既然有数据套件和套件配置了，所以尝试直接配置uwsgi和nginx启动这个套件下的seafile就够了，其它可按《odoo》一文中的来，成功！：

nginx配置逻辑： 

```
include uwsgi_params;
uwsgi_param UWSGI_CHDIR /usr/local/seafile/data/seafile-server/seahub/seahub;
uwsgi_param UWSGI_MODULE uwsgi-server; (不需要.py)
uwsgi_param UWSGI_CALLABLE application;
uwsgi_pass 127.0.0.1:8000;
```

启动的逻辑：

```
/usr/local/seafile/sbin/nginx

/usr/local/seafile/bin/uwsgi --socket=:8000 --master --uid=tc --gid=root --daemonize=/usr/local/seafile/bin/uwsgi.log 
```
 

 好吧，自己DIY着去HIGH吧。恩恩


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339547/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





