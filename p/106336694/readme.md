为tinycolinux创建应用包-toolchain和编译方法
=====

__本文关键字：tinycorelinux编译gcc套件,live,vhd二合一colinux,tinycorelinux lnmp__

在前面我们提到，一个linux发行包只要提供了核心部分和cui的基础toolchain部分才算是一个基本完整的linux发行包，因为扩展将来都由这套toolchain编译而来。在《为tinycolinux创建应用包》中我们用简单解压组合tcz的方式组建了一个lnmp环境包(mysql5.1+php5.3)，在这里，我们准备为tinycolinux建立一个toolchain环境，并用源码编译的方式产生高版本的mysql+php的lnmp包,而这也是更通行和更灵活的办法。

>关于编译新gcc套件及处理glibc移殖的问题

>编译GCC可能面临二种需求环境：1) 从本地产生,比如你需要一个bootstrap的gcc低版本来产生高版本，2) 从外部crosscompile而来。

>默认gcc第一遍只需要gmp,mpc,mpfr加gcc，这样--enable-language=c,c++编译出来的gcc支持stdlibc++-dev却不带libc-dev,甚至binutils都不需要，如果目标环境中没有支持是没有实用的。

>完整可用的gcc套件要经过多遍，除了gcc,binutils，甚至还需要附加编译flex,bison这些，

>最重要的问题来了：

>默认gcc仅带libstdc++，这个可以后期添加新版本替换/叠加系统原有版本因为它是built into toolchain的，而glibc的版本是一个linux发行版rootfs中集成的built into rootfs，是最为基础的被引用部分，不可升级/替换,是一个不可移殖项。你需要另外准备平台依赖的libc-dev(glibc-dev)，这可能需要在其它遍次pass,phase的gcc编译中完成。

>其实GCC也算是一个类kernel的复杂包了，ng-crosstools有用类kernel的menuconfig方式产生.config环境。

 以下测试过程全在硬盘版的tinycolinux下测试，live版的不方便。请下载tinycolinux live hd一体包后继续：

组建bootstrap toolchain
-----

以下tcz默认全是4.x的，从4.x的compiletc.tcz的meta包的dep中提取而来，以下底部部分eglibc_base-dev就是glibc开发包，glibc runtime已经在tinycolinux的/lib中了，底部其它的那些是可选开发包，因为比较基础都保留了，gcc为461版本，请手动从某个镜像的4.x/tcz目录下载这些包到/mnt/cobd0/tce/optional，然后在console-nt中直接paste执行以下脚本。

```
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/gmp.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libmpc.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/mpfr.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/gcc.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/binutils.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/m4.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/make.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/pkg-config.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/bison.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/flex.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/gawk.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/grep.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/sed.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/patch.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/diffutils.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/findutils.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/file.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/eglibc_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/gcc_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/linux-3.0.1_api_headers.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/util-linux_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/imlib2_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/e2fsprogs_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/fltk_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/freetype_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/jpeg_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libpng_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libsysfs_base-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/zlib_base-dev.tcz
```

解压完毕后测试 gcc -v,g++ -v，可能你需要sudo reboot重启一次tinycolinux系统才能发现已安装但缺失包。

遵从以上谈到的编译gcc的难点，其实你完全可以用这套GCC461作bootstrap编译出新的GCC如gcc483 gcc491 etc..。以下我们用它测试编译新lnmp：

编译新lnmp
-----

不可直接用lnmp.org的一键包，因为系统集成的工具扩展不一样，一般地，先编译mysql，再php，再nginx，这样php的--with-mysq=就可以引用到mysql的开发包了。以下包都跟上面一样下载来自4.x，整篇文章的tcz都来自4.x（注意cmake例外）：

 1) mysql5.5

```
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/cmake
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libidn.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libiconv.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/ncurses-dev.tcz
```

5.5之后的mysql需要cmake。cmake.tcz是3.x的。4.x的cmake需要安装4.x的额外libstdc++，因此我舍弃4.x cmake选择了混合3.x的tcz。

ncurses nginx和mysql都需要（nginx需要运行库部分，mysql需要dev pkg部分）

mkdir b ; cd b ; cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DWITH_MYISAM_STORAGE_ENGINE=1 .. 

2) php5.6

```
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/curl.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/curl-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libxml2.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libxml2-bin.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libxml2-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/liblzma.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libssh2.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/openssl-1.0.0.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/libpng.tcz
```

./configure --prefix=/usr/local/php --enable-fpm --enable-zip --enable-mbstring --with-mysql=mysqlnd --with-pdo-mysql=mysqlnd  --with-zlib --with-gd --with-curl

(我们选取了能安装owncloud需要的那些包,mysql就不引用实际编译的5.5的包了用了src自带的mysqlnd)

3) nginx1.11

```
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/pcre.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/pcre-dev.tcz
sudo unsquashfs -f -d / /mnt/cobd0/tce/optional/ncurses.tcz
./configure --prefix=/usr/local/nginx
```

 以上编译过程中，如果解压发现不了实际已解压的引用包的，一般是一些含.so的包，需要sudo reboot重启一次guest系统

配置运行部分：
-----

上面php和mysql显然没指定my.cnf和php.ini的目录，但它们默认分别都在/usr/local/mysql/和/usr/local/php/lib/php.ini，自己建2个即可,需要配置php.ini这二个文件，tz.php中才能显示smtp支持和控制php更多行为的那些选项如上传max upload size。

其实大多数可以参照《为tinycolinux创建应用包》中的做法，但还有一些附加处理部分：

mysql中新建一个tmp用来放mysql.sock，其权限要和data一样，都设为0755且归staff下的tc用户所有。这样mysql_install_db才能正确产生初始数据库+pid文件和mysqld_safe产生mysql.sock文件

启动的方法都可以在/opt/bootlocal.sh下加二条：

/usr/local/nginx/sbin/nginx ; /usr/local/php/sbin/php-fpm ; /usr/local/mysql/bin/mysqld_safe &

-----------------------

当然你还可能需要额外编译mta和redis这些，我发现网上chenall(grub4dos作者)有一个类似的tinycolinux的发行包，使用的是473。 它能把/home,/tce都挂载到本地share文件夹中，虽然权限不能继续，但在一定意义上，这是“共盘windows,linux”，这应该不是官方的支持的功能(vmlinux or initrd473)，只是不知道怎么实现的。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336694/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



