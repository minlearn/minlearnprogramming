为tinycolinux制作应用包
=====

__本文关键字：tinycolinux自定义应用包，tinycolinx内存运行，mysql重建/tmp/mysql.sock__

在前面《发布tinycolinux》中，我们重点描述了将tinycolinux安装到硬盘的情况，及处理安装应用到硬盘的情况，这也是大部分情形下的场景，其实，完全可以采取其rootfs放在livecd ram中运行而应用依然安装到硬盘的方式，这样更有利于vm container iaas环境建server farm，这样rootfs是加载到ram中去的。只要重启，一切对系统的更改将撤消。用户就不会轻易破坏系统。

ram中运行rootfs
-----

首先在conf/colinux.conf中root=/dev/ram0,initrd=microcore.gz,cobd0=/imgs/tinycolinux1g,这样启动起来的colinux其rootfs在/dev/ram0中，硬盘介质中仅用于保存用户数据，即conf/colinux.conf中定义的四个挂载地址/opt=cobd0,/home=cobd0,local=cobd0,tce=cobd0，这四个可持化挂载点是colinux那些当且仅当需要修改的地方，所以需要被挂载持久，，我们还可以再定义几个变量加强这个live rootfs的强度，如norestore,

启动，运行。成功进入到tc用户的cmdline.

当然，虽然这个live rootfs系统启动起来了，这个rootfs还是有点raw form和不便的。比如：

通过df命令我们发现定义的四个挂载点，仅挂载了三个到/缺了/tce，但是硬盘中依然生成了四个文件夹/opt,/home,/tclocal对应/usr/local，和/tce，通过tce-load -w发现下载的包在/mnt/cobd0/tce中，这是正确的行为，能用但不好看，这四个挂载点的加载逻辑全在/etc/init.d/tc-config中，所以我们甚至可以重新打包microcore.gz修改tc-config加入缺失的/tce条目。

modprobe也会出错，因为readonly live rootfs是不能加载原initrd.gz注入的lib/modules的，不过同样地，我们可以重新打包microcore.gz手动加入这些文件。

还有一些必要的系统级持久无法完成，比如用户密码更改，它保存在readonly rootfs /etc/shadow中，我们必须这样来完成：

sudo passwd root

输入密码二次

cp /etc/shadow /opt/shadow （做一次备份到硬盘中/opt）

然后修改下/opt/.bootsync.sh，加入以下：

```
cp /etc/shadow /etc/shadow_old 
cp /opt/shadow /etc/shadow
```

其实我们完全可以替换busybox中的passwd，改变/etc/shadow路径到其它外部可持久位置，还比如，vm container子机环境不需要关机，可以去掉busybox中的halt，还比如我们可以编译加入dropbear支持，毕竟sshd是最基本的发行包支持了。

我们就不定制microcore.cpio包了。太累。

组建复合应用
-----

官方提供了很多镜像，这些包都很正交。且还有构建源码，可往往我们还需要lnmp这样的组合包，我们可以按《发布tinycolinux》part2中的硬盘安装应用方法来组合一次性安装包（当然，这样它就不正交了但对一台vm container通常情况下仅需承载安装一次lnmp的情形下来说，非常合理和实用），以下是组合应用逻辑，举例我们用了lnmp，组合到一个lnmp.tar.gz中。

首先，tce-load -w nginx,php5,sqlite3，发现会下载大量tcz到/mnt/cobd0/tce/options中：bsddb.tcz,bzip2-lib.tcz,curl.tcz,gmp.tcz,libgdbm.tcz,libiconv.tcz,libltdl.tcz,libmcrypt.tcz,libpng.tcz,libxml2.tcz,libxslt.tcz,mysql.tcz,ncurses.tcz,ncurses-common.tcz,nginx.tcz,openssl-0.9.8.tcz,pcre.tcz,perl5.tcz,php5.tcz,readline.tcz,sqlite3.tcz，这些都是我们要组合进一个大应用包的基础。一个一个解压它到my文件夹，sudo unsquashfs -f -d /mnt/cobd0/my/ /mnt/cobd0/tce/optional/xxx.tcz

作一些更改（这是因为原tcz全是绿色dropin包）：

nginx conf/nginx.conf，root index加个index.php,把关于php的三条注释去除注释化使其有效，其中SCRIPT_FILENAME改成  $document_root$fastcgi_script_name;且把最大脚本内存由128m改为64mb

usr/local/etc加个my.cnf，内容如下：

```
[mysqld]
socket     = /tmp/mysql.sock
port       = 3306
pid-file   = /tmp/hostname.pid
datadir    = /usr/local/var/mysql
language   = /usr/local/share/mysql/english
user       = tc
```

好了，现在重建数据库，sudo /usr/local/bin/mysql_install_db，，尝试启动mysql:  sudo /usr/local/bin/mysqld_safe & ，成功

然后我们cd /mnt/cobd0/my，打包它们sudo tar zcf lnmp.tar.gz *，，，安装这个大应用测试下：cd到/，然后tar zxvf /mnt/cobd0/my/lnmp.tar.gz，然后在/opt/bootlocal.sh中启动它们：

sudo nginx；sudo php-cgi -b 127.0.0.1:9000；sudo mysql_safe

成功。

-----------

这个我本来还要集成memcached (bootlocal中启动用memcached -d 64m限制最大使用内存)和postfix的。postfix适合另外起一台vm container建一个emailserver的组合包。而不是放到lnmp中。

当然，如果自己要从源码构建php等的新版本，而不是直接利用官方包组合，这需要处理好多东西。恩恩



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336678/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



