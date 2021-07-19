利用hashicorp packer把dbcolinux导出为虚拟机和docker格式(2)
=====

__本文关键字：Cross-compile 64-bit kernel on 32-bit machine__

在《将tinycolinux以硬盘模式安装到云主机》和《在tinycolinux32上装tinycolinux64 kernel和toolchain》2篇文章中我们手动构建了32到64的tinycolinux的基础部分，而且用的是非cross compile，即直接在一台ubt64上产生这个kernel，再后来，针对文章1 —— 以硬盘方式安装iso到主机，我们利用虚拟机devops的构建工具packer自动构建了基本的tinycorelinux pe，现在我们继续文章2，利用devops packer自动构建直接从32位编译出64位的kernel。

这就需要用到cross compile。废话不说，直接上源码。这些源码可直接附在《基于虚拟机的devops套件及把dbcolinux导出为虚拟机和docker格式》文章所提到的源码的适当部分，然后以相同的方式构建，结果会产生一个可用的64 kernel+64 toolchain的dbcolinux：

新加的script1部分
-----

注意这里的不同，root=/dev/hda1

```
#HD INSTALL
cd /mnt/hda1/
gunzip -c /mnt/hda1/boot/microcore.gz > /tmp/microcore.cpio
cpio -idmv < /tmp/microcore.cpio
echo menuentry \\"dbcolinux hd\\" { >> /mnt/hda1/boot/grub/grub.cfg
echo  linux /boot/bzImage com1=9600,8n1 loglevel=3 user=tc console=ttyS0 console=tty0 noembed nomodeset root=/dev/hda1 tce=hda1 opt=hda1 home=hda1 restore=hda1 >> /mnt/hda1/boot/grub/grub.cfg
echo } >> /mnt/hda1/boot/grub/grub.cfg
rm /tmp/microcore.cpio
```

packer脚本部分
-----

将这些放进provisioners中的适当部分。注意下面的_comments会引起错误，这里只是为了说明，使用时得去掉。

```
{
    	"type": "shell",
    	"pause_before":"1s",
	"_comment": “以下安装32位gcc443的官方tczs，因为grep,gcc_libs在scripts1中安装过了，所以这里不必另外安装，另外,scripts1生成的tmp得改一下权限才能在下一步上传文件”,
    	"inline":
		[
		"tce-load -iw gcc.tcz",
		"tce-load -iw binutils.tcz",
		"tce-load -iw bison.tcz",
		"tce-load -iw diffutils.tcz",
		"tce-load -iw file.tcz",
		"tce-load -iw findutils.tcz",
		"tce-load -iw flex.tcz",
		"tce-load -iw gawk.tcz",
		"tce-load -iw m4.tcz",
		"tce-load -iw make.tcz",
		"tce-load -iw patch.tcz",
		"tce-load -iw pkg-config.tcz",
		"tce-load -iw sed.tcz",
		"tce-load -iw base-dev.tcz",
		"tce-load -iw linux-headers-2.6.33.3-tinycore.tcz",

		"tce-load -iw perl5.tcz",
		"tce-load -iw ncurses.tcz",
		"tce-load -iw ncurses-dev.tcz",

		"sudo sh -c 'chown tc:staff /mnt/hda1/tmp/'"

		]
	},

	{
	"_comment": “这里开始上传编译所需的源码文件”,
  	"type": "file",
  	"source": "pkgs/3.x/src/64kernelandtoolchain",
  	"destination": "/mnt/hda1/tmp"
	},

	{
	"_comment": “由于这里用了sudo，在scripts,2,3中所有命令默认都是经过了sudo的”,
    	"type": "shell",
    	"pause_before":"1s",
    	"execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'",
    	"scripts":
		[
		"./scripts/2.compile64toolchain.sh",
		"./scripts/3.compile64kernel.sh"
		]
    	}
```


compile64toolchain and compile 64 kernel
-----

将这些放进scripts/2.compile64toolchain.sh文件中，不要轻易动这里的东西，否则格式不对产生的tls.h不能发现错误很诡异。

```
export PATH=$PATH:/usr/local/sbin:/usr/local/bin

#我们现在是在32位上创建for 64的交叉，从64到32不叫交叉，反之要交叉
#三元组,machtype在在正常的linux32上会输出i686-pc-linux-gnu字样,tinycorelinux上是空。但是不产生问题
export BUILD=$MACHTYPE
export HOST=x86_64-pc-linux-gnu
export TARGET=x86_64-pc-linux-gnu

#我们全程不必用sudo，因为packer文件中exe command写好了
cd /mnt/hda1/tmp/64kernelandtoolchain/
tar jxvf linux-2.6.33.3-patched.tar.bz2
tar zxvf binutils-2.20.tar.gz
tar zxvf glibc-2.12.1.tar.gz
tar zxvf mpfr-2.4.2.tar.gz
tar zxvf gmp-4.3.2.tar.gz
tar jxvf gcc-4.4.3.tar.bz2
mv gmp-4.3.2 gcc-4.4.3/gmp
mv mpfr-2.4.2 gcc-4.4.3/mpfr

cd linux-2.6.33.3 
make headers_install ARCH=x86_64 INSTALL_HDR_PATH=/mnt/hda1/usr/local/gcccross/$TARGET

cd ../binutils-2.20 && mkdir b && cd b
../configure -prefix=/mnt/hda1/usr/local/gcccross -target=$TARGET -disable-multilib
make
make install

#编译器执行文件
cd ../../gcc-4.4.3 && mkdir b && cd b
../configure -prefix=/mnt/hda1/usr/local/gcccross -target=$TARGET -enable-languages=c,c++ -disable-multilib
make all-gcc
make install-gcc

export PATH=$PATH:/mnt/hda1/usr/local/gcccross/bin

#标准C库头文件和一些必要启动文件
cd ../../glibc-2.12.1 && mkdir b && cd b
#在glibc眼里，这里的build和host一个意思，与标准的其它三元组意义不同
../configure -prefix=/mnt/hda1/usr/local/gcccross/$TARGET -build=$MACHTYPE -host=$HOST -target=$TARGET -with-headers=/mnt/hda1/usr/local/gcccross/$TARGET/include -disable-multilib libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes
make install-bootstrap-headers=yes install-headers
make csu/subdir_lib
install csu/crt1.o csu/crti.o csu/crtn.o /mnt/hda1/usr/local/gcccross/$TARGET/lib
#利用新编的64gcc处理
x86_64-pc-linux-gnu-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /mnt/hda1/usr/local/gcccross/$TARGET/lib/libc.so
touch /mnt/hda1/usr/local/gcccross/$TARGET/include/gnu/stubs.h

#编译器本身的库文件
cd ../../gcc-4.4.3/b/
make all-target-libgcc
make install-target-libgcc

#标准C库
cd ../../glibc-2.12.1/b/
make
make install

#标准C++库
cd ../../gcc-4.4.3/b/
make
make install
```

将这些放进scripts/3.compile64kernel.sh文件中

```
export PATH=$PATH:/mnt/hda1/usr/local/gcccross/bin

#使用刚编译出来的gcccross，记得末尾的-
export ARCH=x86_64
export CROSS_COMPILE=x86_64-pc-linux-gnu-

cd /mnt/hda1/tmp/64kernelandtoolchain/
# 这里奇怪之处：1，不作把/usr/local/ncurses*相关的库移到/usr/下会大量符号undefined，2，在guest os中测试时如果不用sudo，依然会提示无法找到ncurses相关库，3，sudo make clean没用，一定要sudo make mrproper。4，在virtual box play make menuconfig那行时会一直闪烁，在guestos中测试是正常的,5.所以我们用了make defconfig的方法,但这里还是保留menuconfig相关的脚本
cd linux-2.6.33.3
cp -R /tmp/tcloop/ncurses-dev/usr/local/include/*.* /usr/include/
cp -R /tmp/tcloop/ncurses-dev/usr/local/lib/lib*.a /usr/lib/
cp -R /tmp/tcloop/ncurses/usr/local/lib/*.* /usr/lib/
#直接放文件可能需要重启才能被引用生效，所以这里ldconfig一下
ldconfig
export TERMINFO=/usr/share/terminfo
export TERM=linux
make mrproper
cp -f ../config-2.6.33.3-tinycore64 arch/x86/configs/x86_64_defconfig
make x86_64_defconfig
make bzImage


cp -f arch/x86/boot/bzImage /mnt/hda1/boot/bzImage64
echo menuentry \\"dbcolinux 64\\" { >> /mnt/hda1/boot/grub/grub.cfg
echo  linux /boot/bzImage64 com1=9600,8n1 loglevel=3 user=tc console=ttyS0 console=tty0 noembed nomodeset root=/dev/hda1 tce=hda1 opt=hda1 home=hda1 restore=hda1 >> /mnt/hda1/boot/grub/grub.cfg
echo } >> /mnt/hda1/boot/grub/grub.cfg

rm -rf /mnt/hda1/tmp/64kernelandtoolchain/

#这是一条触发ask的错误语句,放在你猜想可能出现错误作为中断点的地方，但是它还会继续
Dsfasdfdf
```

好了，scripts3文件结束，产生的镜像文件可能很大。需要针对处理一下。


————

关注我


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336800/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





