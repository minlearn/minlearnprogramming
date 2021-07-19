在tinycolinux上安装和使用cloudwall
=====

__本文关键字：在tinycolinux上安装和使用cloudwall,同步器as webos，uniform native web appstack__

在《cloudwall:一种统一nativeapp和webapp的appstack》中我们讲到，cloudwall是一种构建在counchdb+counchdbapp之上的管理层APP可直接作为personal cloud hosting 文档和支持cloudwall plugin开发，然而它产生的奇妙效用在于它能作为webos，提供webappstack的效用，类似我们一直追求的engitor：介乎os和app之间的层面，封装domainapp开发栈和开发工具的层面。然而它更强大：它提供本地远程一致的webapp开发和发布方式（以无差streamed to bs和anyinstance + inapp editor的方式）。

其实这一切都基本都是counchdb的效果，它集成HTTP，本身是个DB带存储，与浏览器和JS结合紧密，支持hosting couchdbapp，文档即APP，这个APP仅由HTMLCSSJS构成，这种机制为远程web提供了至少三个栈，满足在其中搭建APP的基本条件：1,它的http部分免去了协议开发,web页面又是易于streamable的，2,它的DB属性免去了存储逻辑开发需要。 它stream到本地和每个counchdb instance（replicate）的结果是一样的，保证了浏览器与服务器之间的数据可以做到本地和远程不断联（in-browser os ），本地和远程，最难跨越的就是这个无缝stream。3，它采用的counchdb使用全栈语言JS，托管在其中的每个cloudwall app本身既是服务端的程序也是客户端程序(nobackend webapp)。然而就像tiddywiki一样：实际上在服务端JS只是静态文档stream到客户端执行，服务端只视一切为文档只是同步器。而tiddywiki这样的东西少了数据库托管。

可以说，正是JS和couchdb的完美结合促成了cloudwall，一个lang一个hostingtime,runtime在B端，这种意义下的“WEBAPP”不分本地还是远程，都是通过数据库stream的一个端，这就是文章标题说的：uniform native web appstack.

下面，我们讲解在tinycolinux上搭建cloudwall，和讲解在使用它的过程中，那些可以作为personalcloud使用的方方面面。

之前几篇文章，我们提出了跨本地/远程的DISKBIOS XAAS系统，并完善了一个/system rootfs，如前面所说，这是一个system与user libary分开的rootfs系统，/下只有三个文件夹/boot,/system,/usr,在/usr下有/local,/tce,/opt,/home,/vz对应于tinycolinux的那些mountable文件夹。那么从本篇开始，我们将管这个新的tinycolinux为dbcolinux，且以后的发布类文章都搬到其上来实践,如下cloudwall即是一例。
在《cloudwall:一种统一nativeapp和webapp的appstack》中我们讲到，cloudwall是一种构建在counchdb+counchdbapp之上的管理层APP可直接作为personal cloud hosting 文档和支持cloudwall plugin开发，然而它产生的奇妙效用在于它能作为webos，提供webappstack的效用，类似我们一直追求的engitor：介乎os和app之间的层面，封装domainapp开发栈和开发工具的层面。然而它更强大：它提供本地远程一致的webapp开发和发布方式（以无差streamed to bs和anyinstance + inapp editor的方式）— 这一切正是我们自bcxszy以来就追求的。
cloudwall何以如此强大？其实这一切都不难理解，因为排除cloduwall,这基本都是counchdb的效果，它明显集成了HTTP，本身是个DB带存储，与浏览器和JS结合紧密，支持hosting couchdbapp，文档即APP，这个APP仅由HTMLCSSJS构成，这种机制为远程web提供了至少三个栈，满足在其中搭建APP的基本条件：1,它的http部分免去了协议开发,web页面又是易于streamable的，2,它的DB属性免去了存储逻辑开发需要。 它stream到本地和每个counchdb instance（replicate）的结果是一样的，保证了浏览器与服务器之间的数据可以做到本地和远程不断联（in-browser os ），本地和远程，最难跨越的就是这个无缝stream（既然WEBOS可以类比为一个云存储based带nativedev likehood appstacks的东西，其必定要有DB一层，所以为何不以DB的replicate直接为网盘同步呢和app sync呢？其实当初W3C的WEB标准也准备是为WEB在BS端准备无差的web storage,web sql,webgl,etc..以提供类nativedev的appstack效果，不过好多实践被逐渐抛弃了）。3，它采用的counchdb使用全栈语言JS，托管在其中的每个cloudwall app本身既是服务端的程序也是客户端程序(nobackend webapp)。然而就像tiddywiki一样：实际上在服务端JS只是静态文档stream到客户端执行，服务端只视一切为文档只是同步器(服务器不保存程序逻辑仅数据又像极了微端。在微端眼中，与B端浏览器搭配最好的服务端的标准设施应该就是DB了而不是logicserver。)。而tiddywiki这样的东西少了数据库托管。
可以说，正是JS和couchdb的完美结合促成了cloudwall，一个lang一个hostingtime,runtime在B端，这种意义下的“WEBAPP”不分本地还是远程，都是通过数据库stream的一个端（而其实couchdb也支持传统的serverside applogic vs synced applogic），这就是文章标题说的：uniform native web appstack.
下面，我们讲解在dbcolinux上搭建cloudwall，我使用的是gcc443 32bit，下的是otp_src_20.3.tar.gz(erlang),js185-1.0.0.tar.gz,apache-couchdb-2.1.1.tar.gz
除此之外，还需要准备3.x的zip-unzip.tcz,icu.tcz,icu-dev.tcz，好了，开始吧

编译erlang,mozjs
-----

由于dbcolinux的rootfs还处在初级阶段，有一些程序编译和运行还需要原来的/下的目录布局，如make meunconfig指令时引用到的/usr/lib一定要存在否则即使安装了ncurses.tcz和ncurses-dev.tcz,还会一直提示undefined reference，所以在这，为了顺利完成以下的编译，我们暂且恢复它，这些空目录只是编译时的权宜，dbcolinux运行不需要，所以编译完后可删除。
要恢复的目录与文件有：

```
/tmp
/bin指向到/system/bin
/lib指向到/system/lib
etc/resolver弄回来,nameserver 8.8.8.8
/usr/bin指向/system/bin
/usr/include指向/system/include
/usr/lib指向/system/lib
```

解压otp_src_20.3.tar.gz，它不支持shadowbuild和configure，直接cd src,sudo ./configure –prefix=/usr/local/cloudwall/ –with-ssl=/system,如果不加ssl，稍后会出现Uncaught error in rebar_core，然后make,make install

现在来编译mozjs,会使用到python,python要编译进ssl才能安装pip，然后被用于接下来的mozjs,改下Python build目录下的Modules/Setup中的SSL段内容为：

```
SSL=/system
_ssl _ssl.c \
-DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
-L$(SSL)/lib -lssl -lcrypto
```

再安装pip，这里就需要用到/etc/resolver这个原先的文件来解析下载地址。安装zip-unzip.tcz,直接cd js-1.85/js/src,./configure –prefix=/usr/local/cloudwall,make,make install,一路成功，注：这里千W不要下到mozjs-45.0.2.tar.bz2这样的源码包，couchdb2只要求185的spidermonkey js，编译mozjs 45要麻烦得多，它要求gcc47,glibc2.12，且接下来与couchdb连接不了，比如：会出现，编译/src/couch_js/*c下文件，c包含C++头文件发生error: unknown type name xxx 的情况，涉及到修改src/couch/rebar.config.script，但是最终不能成功。

接下来编译couchdb,cd src,./configure –disable-docs,不能执行rebar,会发现它引用了/usr/bin，按开头说的先把这目录恢复回来，又发现解压出来的apachecounch权限是乱的，全部弄为root,make时rebar会用到erlang,设export PATH=$PATH:/usr/local/cloudwall/bin,再make release，提示不能发现jsapi.h，修改src/couch/rebar.config.script：

```
{“linux”, CouchJSPath, CouchJSSrc, [{env, [{“CFLAGS”, JS_CFLAGS ++ ” -DXP_UNIX -I/usr/local/cloudwall/include/js”}, 
{“LDFLAGS”, JS_LDFLAGS ++ ” -L/usr/local/cloudwall/lib -lm”}]}]},
```

成功，提示你复制生成的rel到目标文件夹。

安装cloudwall
-----

把生成的rel复制到cloudwall:cp -R rel/* /usr/local/cloudwall，并安装icu.tcz，现在，将js libs 也复制到/usr/local/lib下，然后

改下/etc/default.ini中二个127.0.0.1为0.0.0.0,

cd /usr/local/cloudwall/rel/couchdb,./bin/couchdb，成功。访问,xxx:5984/_utils/#verifyinstall，进fauxton，在左下user处增加默认的管理用户，用户名一定要admin，然后添加一个数据库mineportal，然后在这个数据库的design处创建一个文档出现文档编辑区，下载

https://cloudwall.me/cloudwall-2.2.json，用noteplus打开，复制，粘贴到文档编辑区，保存，提示成功后，访问如下页面：

xxx:5984/mineportal/_design/cw22/index.html

进去，输入admin和密码，inliner是创建文章的地方，code是创建codesippter的地方，inliner file,gallery等都像是ocwp mineportal，一个网盘设施所提供的功能那样齐全。它的强大之处就是可以inplace editor生成app.

为了方便启动，你也可以在网上找到etc/init.d之类的开机启动逻辑

---------------------

cloudwall预定于mineporta3,cloudwall能轻易与elm-lang组合，这二者都有强烈的与JS直接绑定的特点。走的是亲JS的同路子，且一个使用erlang一个使用haskell的非主流路线的风格相像。结合使用应该会有奇效。比如，打造一个能在线调试的inapp visual editor for cloudwall，下文就暂定为《另一种ipy：在dbcolinux上安装elmlang》吧


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339611/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




