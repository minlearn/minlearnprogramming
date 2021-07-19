在tinycorelinux上安装lxc，lxd (2)
=====

在《在tinycorelinux上安装lxc，lxd（1）》中我们讲到源码适配gcc443，由c11退回c99的一些处理，这里依然要处理大量gnu11的事。

准备工作，及编译golang
-----

Grub 加个swapfile=hda1进去。编译go1.12.6内存起码1g。准备git，git我们用4.x的，需要expat2.tcz和openssl-1.0.0.tcz,都用3.x的,
按《在tinycolinux上安装sandstorm davros》编译openssl1.0.1覆盖1.0.0 —prefix=/usr/local,make install,sudo ldconfig，再编译curl 7.30.0 —with-ssl=/usr/local,make install,sudo ldconfig,不用编译git,为防出现unable to get local issuer certificate git，运行git config --global http.sslVerify false

安装bash.tcz，下载并解压go1.4-bootstrap-20171003.tar.gz，Go 1.4 was the last distribution in which the toolchain was written in C，cd go,sudo ./make.bash，不要export GOROOT_BOOTSTRAP=/mnt/hda1/tmp/go，这个没用，还是得mv /mnt/hda1/tmp/go /home/tc/go1.4，下载go1.12.6.tar.gz,cd go-go1.12.6/src,sudo ./make.bash没有之前的swap设置这里过不去，
为了让go生效。export PATH=$PATH:/mnt/hda1/tmp/go-go1.12.6/bin

lxd源码处理
-----

安装libcap.tcz,acl-dev.tcz,下载并解压lxd-3.0.4.tar.gz,cd lxd-lxd-3.04,处理一下lxd src:

> 第一个问题，还是那个问题，我们使用的gcc443不是gnu11，go默认调用gnu11，会出现Unknown command line -std=gnu11
> 在lxd src中，找到// #cgo 有-std=gnu11的去掉它，对，注释的起作用的，大约有16个文件,然后，在/home/tc/go/src中新建github.com->lxc文件夹,cd lxc,直接mv 修改过的lxd到这里，保证名字是lxd
> /lxd/shared/idmap/shift_linux.go中，
> /lxd/shared/netutils/netns_addrs.c中，

然后是makefile:

> Sudo vi Makefile最上面加shell=/bin/bash,default中去掉deps的判断ifeq ($(TAG_SQLITE3),)中的ifeq改成ifneq，进一步来分析一下makefile中这个默认make deps的逻辑：

> 它以home/当前用户/go/为GOPATH,维护这样一种结构(GOPATH)/deps/，所以我们mkdir -p ~/go/src,cd ~/go/src,mv /mnt/hda1/tmp/lxd-lxd-3.04 ~/go/src

> 继续分析makefile，有5个deps:sqlite,uv,raft,co,dqlite，文件中有4个地址，没有libuv的，稍后处理，但因为这5个deps都可能编译出错，make deps一执行，总是会强行从0开始拉取（sqlite无条件拉取，其它四个判断拉取），所以不可能通过本地修改deps sqlite的相关文件，调试影响make deps使之最终通过。我们只能定制sqlite仓库，然后在makefile中替换其地址：

> Sqlite:
> 2019.4.19左右的sqlite：https://github.com/CanonicalLtd/sqlite/3c5e6afb3d8330485563387bc9b9492b4fd3d88d,你必须fork 它的github仓库，作如下修改，并改动makefile中的GitHub repo调用地址参数来跳过这个
> 在src/sqlite3.h.in中：
> 删掉这句 typedef struct sqlite3_wal_replication sqlite3_wal_replication;
> 然后下面typdef struct sqlite3_wal_replication{…}的sqlite3_wal_replication的前面统统加个struct，有五行
> 才能避免make deps编译时可能出现redefinition of typedef ‘sqlite3 wal replication’，gcc 4.7之后才支持c11的typedef重定义-Wtypedef-redefinition,，gcc 443是不支持的,

其它四个deps可以分别git到/mnt/hda1/tmp修改，尝试make install，

> libuv:
> Git clone 2019.6.28左右的https://github.com/libuv/libuv/commit/1a06462cd33fb94720d639f40db3522313945adf
> Sudo ./autogen.sh,./configure,make install

> Raft:
> Git clone 2019.6.26左右的，https://github.com/CanonicalLtd/raft/commit/ee097affa3dfff53f0c5af096a55d8b7dacecdc3
> 会出现error implicit declaration of function aligned_alloc，因为C11中添加的函数aligned_alloc（）
> 你可在configure.ac 161行找到implicit-function-declaration相关行注释掉，这样它就是一个warning而不是error
> ./configure —disable-example,否则会有TIME_UTC is a macro in C11,TIME_UTC is macro in glibc 2.16

> libco:
> https://github.com/freeekanayaka/libco,目前是v18,没有make install
> 复制 lib.pc到 /usr/lib/pkconfig/
> 手动复制安装下/usr/为prefix然后ldconfig

当然你也可以像对待sqlite一样将修过改的后4个deps的新仓库地址放进makefile中，尝试Sudo make deps，找不到libuv时到那个deps下make install下再sudo ldconfig重新make deps，这样更方便统一。

以上lxd src和dep的src处理，因为go或makefile会将文件不断下到go path，调试的时候，如果有新的错误，记得清空/deps/或src/github.com/中相应的文件夹让makefile或go get重新应用新逻辑。

编译deps和lxd
-----

Make dep,最终成功!!之后需要设几条export，编译完后会提示：
Export CGO_CFLAGS=“-I/home/tc/go/deps/sqlite/ -I/home/tc/go/deps/dqlite/include/“
Export CGO_LDFLAGS=“-L/home/tc/go/deps/sqlite/.libs/ -L/home/tc/go/deps/dqlite/.libs/”
Export LD_LIBRARY_PATH=“/home/tc/go/deps/sqlite/libs/:/home/tc/go/deps/dqlite/.libs/”
如果是手动生成的，对应地址会是/mnt/hda1/tmp/xxx

最后，在整体make (default)前需要处理一下：

> 在这里会有很多陷阱和挑战，主要是golang的包下载需要用到外网线路而且go没有一个可以换mirror的准法。为省事我们将手动补全：src中新建golang.org文件夹->x文件夹，cd x,依然git clone github.com/golang/sys/,github.com/golang/net/, github.com/golang/crypto/,这是因为golang.org的包全部被墙,还有一些虽然没被墙但是较大的包，手动下载,比如下到gopkg.in的mgo v2，cd gppkg.in,git clone https://github.com/go-mgo/mgo/,mv mgo mgo.v2,cd mgo.v2,git checkout v2，v2是它的一个branch

sudo make。会自动下载其它包，没被墙的gopkg.in被依然下载到/home/tc/go/src/gopkg.in。然后自动开始编译，如果在这里出现找不到deps的h,lib往往是make deps后的几条export没设好，没关系，这里可以进一步export覆盖补全。

最后，lxd也编译完成。完工！


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337502/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>






