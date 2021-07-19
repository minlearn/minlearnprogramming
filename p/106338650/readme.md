利用colinux制作tinycolinx，在ecs上打造server farm和vps iaas环境代替docker
=====

__本文关键字:将tinycorelinux装在硬盘上，custom tinycore linux kernel,tcl3安装使用方法,tcl安装到硬盘,自定义linux rootfs,利用colinux代替docker组建容器。单机端口反代重用技术，内网转发复用端口__

在《阿里云上利用virtiope+colinux实现linux系统盘动态无损多分区》中我们用colinux实现了一个uniform pe装机环境的东西，在《利用colinux,免租用云主机将mineportal2做成nas》一文中，我们利用colinux结合mineportal打造了一个多用实用的nas，在《一个设想：基于colinux，是metaos for realhw and langsys，也是一体化user mode xaas环境》中我们提到colinux还可用于类vagrant开发虚拟机，软件兼容层等等之类的东西，xaas环境，而其实，只要上升到host/guest层面，作为guest的colinux还可以完成更多的东西，比如这里接下来要讲到的：利用colinux切分服务器资源，打造你的专属serverfarm --- 其实，它也属于《一个设想：基于colinux .xaas..》提到的xaas的一个应用：它可以模拟docker容器，将ECS分为多个子colinux系统，比如，对于一个1C核1G内存的ECS，我们可以根据128M为粒度大小，利用colinux能在conf中设置mem的大小的能力，将ECS资源划分为512/128=4个colinux容器。除了给内存配额在tinycorelinux wiki中还有定义governers限制CPU的能力和讲解。这一切都不需要利用到docker这种过度虚拟化的方案带来的虚拟地狱缺点：比如docker利用分层联合文件系统，用户难于维护。

为了实现可用性，我们还必须作一系列改进，比如，由于128m内存限制太小，我们必须利用专门的colinux os发行版而不是巨大的ubuntu etc..，比如采用tinycore linux os，它可以做到以至少10M的服务器核心而存在，这跟windows上的boot2docker使用定制版的core linux os是一样的道理。选用tcl的另一个考虑是它有独特的发行包机制，它的发行包比较精简，专用的软件包发行限制可以避免VM用户装乱七八糟的大软件，第三，tcl的处处即挂载的cloud run+可持久机制(整个根可挂载到内存livecd下或其它介质，home目录可挂载,tcz loop mounts必挂载)使得即使给了用户root也不会轻易破坏系统适合VM使用。

对于一个真实可用的vm container环境，还会有其它高级课题，比如，一台ECS只有一个80端口，多个内网colinux VM环境需要重用80端口出网。无论如何，下面先来搞定将tinycore与colinux结合的问题：为colinux制作一个精简的tinycorelinux发行。

制造一个精简server发行版
-----

这一切需要在另外一台linux上完成，比如直接利用我发布的colinux14.04版本:

一开始我参照网上《将ubuntu8.04 iso安装到colinux的方法》先试验下载了最新的各种iso，通过conf文件中直接挂载/dev/cobd1=xx.iso,root=/dev/cobd1,initrd=tinycore.gz的方式企图进入，发现最后都不能进入，有colinux initrd的问题，有tinycore.gz文件系统的问题，有colinux内核的问题。光盘方式看来是不行了。

不会要重编内核，或者重封装rootfs吧？

但其实硬盘方式加+低版本microcore3.8.4.iso是可行的，我重新做了一个1G的硬盘，在colinux中mkfs.ext3格式它，拷贝提取iso中的microcore.cpio直接释放到硬盘，tinycorelinux就开始运行了直到出现登录用户提示符，即cpio -idmv < microcore.cpio到根目录，然后用colinux引导新的硬盘系统（原colinux vmlinux可以无修改，initrd.gz也可以利用上），这样天然地就是tinycorelinux的硬盘模式了。（initrd.gz注入后，再重启，提示scatter harddisk installation mode后进入login提示，用tc用户无密码登录），我把它称为tinycolinux。

tinycolinux的conf中还可以支持tinycorelinux中提到的各种bootcodes，比如root=/dev/cobd0,home=/dev/cobd0,opt=/dev/cobd0,tce=/dev/cobd0这些,由于tinycorelinux会搜索分区上的/tce目录为应用下载目录，所以我也在\新建了一个，否则默认tce-load -wi xxx出来的options会出现在/tmp/tce中，(硬盘模式下home,opt,root都出现在当前所在的硬盘根下，在/下建一个/tce目录，有跟conf中设置tce=/dev/cobd0同样的效果)。

由于一切都是在硬盘完成的，整个文件系统都是可持久的，除了应用安装不用处理其它持久化问题了：

TinyCoreLinux持久化问题-用户数据和应用扩展保留
-----

tinycorelinux一开始定位live iso和cloud模式，体现在它能在liveiso完全无持久,和有持久介质的多种场景下运行，其root文件系统核心集和应用扩展settings在每一次重启后都是fresh的重启就丢了一切会话和应用扩展和其设置数据，因为一切都是挂载到外部持久的条目或加载到ram的tcz扩展镜像，对于前者，它实际上就是挂载到持久介质的入口而已 --- 正由于这些都是挂载hook点，所以可以集中卸载，，更改依然在外，对于后者，ram下天然不能持久，二者可以维护一个干净的重启后环境，

而对于必须带入下一次重启，或整个文件系统的持久化，除非你指定保存逻辑和定义保存条目 --- 注意这句话，稍后就会谈到。

有三个可配置挂载的目录，/home,/opt和/tce，你可以挂载全部三个到可持久外部介质或选一二，设想tcl在完全无持久介质的liveiso情况下启动，它根本就没有持久能力，但若指定了至少一个可持久目录到外部介质后启动，它就可以在外部介质上得到更改保存，但这些改变不会被带入到除了这三部分之外的任何根文件系统的其它部分。

有一种情况比较特殊，当tincorelinux整个文件系统被置于入硬盘并从硬盘启动时，实际上硬盘整个就是可读写的（norestore bootcode天然启用，/home,/opt,/tce都是现成被默认定义了的可持久目录）。

而至于对于要带入下一次重启和根目录文件系统的那部分持久和更改，你可以指定restore：比如bootcodes定义了restore，和/home到硬盘和新增了.filetool.lst中的新条目，它就会产生mydata.tgz备份/home和/opt，到这个持久上---你可以定制包括/home,/opt的可持久路径和bootcode中指定restore所在的路径，进行一次filetool -b，并在下一次由系统恢复。

对于应用，同样的方式(如上bootcode指定)，可以有定义了一个保存在可持久介质上的/tce目录，比如它在硬盘上，/tce下的options和onboot.lst的更改就能持久化。就能将*.tcz动态挂载进来（tcz是一些只读文件系统包，挂载进来的时候是挂到内存中）。要注意在这里，应用加载逻辑是持久过的，但应用依然留在内存运行。且应用的设置部分还没有经过显式持久化。

所以进一步地，如果tcz要带入系统作更改，你依然可以结合.filetool工具和机制，将具体tcz安装后需要持久的部分持久化到可持久中或者ln -s部分目录到硬盘，还可以在在/opt/bootlocal.sh中定义开机利用这些目录持久的逻辑。

 

将tinycolinux打造成完全的硬盘系统
-----

到此为止，应用依然是靠一次性加载到内存来运行的。针对已经安装好的tinycolinux，我们需要纯粹在硬盘上安装运行的应用扩展：

（虽然对于live系统和3个目录组成的定点集中维护来看它是最佳的，但实际上，除非对应用本身进行定制，否则应用安装过程实际上可能会对整个系统文件系统产生更改，而这也是其它linux distro软件包的默认行为），，况且，我发现tiny core linux有几种包不一样，像nginx和mysql，前者会loop mounts，后者不会产生loop mounts，mysql安装tce-load -wi安装后会留在/usr/local目标中持久，而nginx在-wi后重启系统仅留下一些软链接，不能统一处理，这反而给安装造成了困扰，为了追求更自然的类现在包管理机制的方案和统一省事的安装方法，我想出的办法最初是直接下载tcz包释放到目标，因为tcz包是简单的文件系统打包大都释放到usr/local目录也比较容易手动安装：

 1）下载应用时只tce-load -w下载到options

绕过tce-load -wi会创造loop mounts的过程。改用tce-load -w而不安装。

2）从这个地址下载http://mirrors.163.com/tinycorelinux/8.x/x86/tcz/squashfs-tools.tcz，提取二个可执行文件放到把它放在根目录/bin中

利用它来进行手动解压。

3）然后视tcz内容在windows上用7z打开查看看它是要释放到哪的，

其下载地址往往是/opt/tcemirror中条目后加/3.x/tcz/包名.tcz得到的，手动解压，unsquashfs -f -d / /tce/optional/nginx.tcz  （这里以nginx为例，它释放到/下）

处理好各个deps的tcz,对于nginx是openssl和pcre。如果所有的deps都安装了还是发现不了so文件，重启一次必发现。sudo nginx会发现不了libprce.so.0

然后在opt/bootlocal.sh中加入随系统自动启动条目，对于openssh是/usr/local/etc/init.d/openssh start，对于nginx是nginx -s start吧。

 完工！nginx在重启后自动运行！要卸载时只须处理/usr/local目录。

----------

其实tcl这种live机制也可用于装机代替virtiope，这是后话了。 除了这些，当这些colinux vm用于建站时,还需要nginx反代多个VM重用一个80的技术，其实说到内网转发复用端口，《基于colinux打造nas》一文中也可以用它来出网。制造发行版的二大基本组件，os本身已经解决了，还有toolchain的问题。根据我的《host2guest guest2host nativelangsys及cross compile system》一文，虽然tinycore linux有gcc应用包，不过我倾向于把它放在windows hosts，来编译制造tinycore linux可用的目标。好了。这些都不讲了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338650/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



