利用hashicorp packer把dbcolinux导出为虚拟机和docker格式（3）
=====

__本文关键字：gcc enable shared cross compile,multiarch gcc,multilib gcc,cross compile gcc under tinycorelinux__

在  《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式》1，2中，我们实现了基本的hd pe环境和释放到hd /能启动的硬盘环境，且编译出了for 64的gcc cross toolchain和64 kernel，那么现在，我们需要使之释放到hd /system下还能启动，且未来将继续这个/system下的32/64混合rootfs的一些强化工作，将之打造成一个高可用的系统.

我们先来完成最基本的效果，即实现《在tinycolinux上组建子目录引导和混合32位64位的rootfs系统》一文提到的效果：初步组建这个/system rootfs，并验证/system下的rootfs能不能够启动。——— 本文完成的我们称它为3264rootfsbase，未来继续强化至3264systemfull.

我们的计划是，按上面的安排，分二步，形成phase1->phase2的效果，phase1即dbcolinuxbase，=前面的toolchain+kernel+基本rootfs的打造。phase2即build3264systemfull,在以后的文章中《打造子目录引导和混合32位64位的rootfs系统全部》再讲述和丰富。第一步的dbcolinuxbase做成一个pvm，即parallels的"type": "parallels-iso”模式，第二步的后续dbcolinuxfull要从这个iso生成的pvm生成，即parallels的"type": "parallels-pvm" builder模式，对应2个phases，这样便于集成，和调试。尝试packer的新东西。且phase2可以直接复用phase1的结果持续集成.

1,packer文件，其它准备工作
-----

由于换了osx加parallelsdesk，我们需要改变一下基本的packer文件,以下只提到加的部分，文章1，2中有的部分会省略：

builders中是这样：

```
"type": "parallels-iso",
"guest_os_type": "linux",
"hard_drive_interface": "ide",
……
"parallels_tools_flavor": "lin",
……
"boot_wait": “4s",
"shutdown_command": "sudo poweroff",
……
"prlctl": [["set", "{{.Name}}", "--startup-view", "window"]],

"boot_command":
[
……
"tce-load -iw parted.tcz<return>",
"tce-load -iw grub2.tcz<return>",
"tce-load -iw squashfs-tools-4.x.tcz<return>",
……

整合了这里为compiletc,base-dev包含linux header
"echo 'setup gcccore pkgs...'<return>",
"tce-load -iw compiletc.tcz<return>",

"echo 'setup gccextra pkgs...'<return>",
"tce-load -iw perl5.tcz<return>",
"tce-load -iw ncurses.tcz<return>",
"tce-load -iw ncurses-dev.tcz<return>"
……
]
```

可见，为了调试，1,2步方便持续集成，我们还把整个gcc放进了bootcommand，还整合进了一些必要库和工具。

再准备文件，与packer文件同层的src文件夹中，分成了/base和/others中，这样的文件夹安排，base对应phase1,是原来的编译toolchain,64kernel所用的源码文件夹+.x tinycorelinux 中的 src repos/busybox-1.19.0patched.tar.gz,busybox-1.19.0-config。，而others对应phase2所要用到的源码包。有autoscan-devices.copenssl-1.0.1.tar.gz,openssh-5.8p1.tar.xz,sudo-1.7.2p6.tar.gz,libzip-0.10.tar.xz,make-3.81.tar.gz,etc..

所以，"provisioners”部分会是这样：


```
{"type": "shell","pause_before":"1s","execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'","inline":["cp -R /tmp/tce ~/"]},
{"type": "shell","pause_before":"1s","execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'","scripts":["./scripts/1.bootstrap.sh"]},

由于我们稍后的bootstrap中会不再释放microcore.cpio,这个tmp要事先生成。
……
"sudo sh -c 'mkdir /mnt/hda1/tmp/'",
"sudo sh -c 'chown tc:staff /mnt/hda1/tmp/'"
……
{"type": "file","source": "src/base","destination": "/mnt/hda1/tmp"},

{"type": "shell",
"pause_before":"1s",
"execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'",
"scripts":
[
"./scripts/2.1.build64toolchainandkernel.sh",
"./scripts/2.2.build3264rootfsbase.sh"
]
}
```

可见原来的2，3整合了成了2，这里的script3是要新加的,为了方便调试，在packer的输出窗口中简便查看起见，包括上面的packer尽量将简单的树形xml写在行内，接下来的脚本中，我们也作了一些小的改变和与原来文章中的不同调整：

我们把重要的注释换成了echo “ing …”，脚本中，解压文件的都不加verbose,涉及到编译toolchain,kernel的地方。都./configure CC=“gcc -w”，且以上统一都 > /dev/null，还有，toolchain单独一个文件夹，与基础rootfs所需的/lib,/lib64分开，我们还将切换native gcc/刚建的cross gcc的方法不再使用export CROSS_COMPILE，而直接用文件名。

调试语句上，发现前面文章写的，一句letitdebug的无义语句，放在位置放在脚本尾或强行断点处并不能让on error ask断下来，reboot会产生断点让后面的语句无法运行，但也会丢失当前环境，packer -debug还是最可用的，除了它每执行一步都问一句有点烦之外，packer不支持-debug-on-error。

这些大部分都需要在脚本层更改，我们一件一件来：

2,bootstrap脚本的变动
-----

因为我们要形成纯净的/system,所以去掉了/下的系统(以下注释掉了部分)，整个bootstrap会是这样:

```
echo "HD INSTALL..."
#以下适合/的情况
#cd /mnt/hda1/
#gunzip -c /mnt/hda1/boot/microcore.gz > /tmp/microcore.cpio
#cpio -idmv < /tmp/microcore.cpio

#tce-load不能sudo,只能unsquashfs方式安装
#cp -R /tmp/tcloop/squashfs-tools-4.x/usr/ /mnt/hda1/
#unsquashfs -f -d /mnt/hda1/ /mnt/hda1/tce/optional/gcc_libs.tcz
#unsquashfs -f -d /mnt/hda1/ /mnt/hda1/tce/optional/openssl-0.9.8.tcz
#unsquashfs -f -d /mnt/hda1/ /mnt/hda1/tce/optional/openssh.tcz
#ldconfig
#tar zxf /mnt/hda1/mydata.tgz -C /mnt/hda1/

#/下，tce=hda1实测运行tce-load是无效的
#echo menuentry \"dbcolinux hd\" { >> /mnt/hda1/boot/grub/grub.cfg
#echo  linux /boot/bzImage com1=9600,8n1 loglevel=3 user=tc console=ttyS0 console=tty0 noembed nomodeset root=/dev/hda1 swapfile=hda1 tce=hda1 opt=hda1 home=hda1 norestore >> /mnt/hda1/boot/grub/grub.cfg
#echo } >> /mnt/hda1/boot/grub/grub.cfg

#以下适合/system的情况
mkdir /mnt/hda1/system/

#除了grub条目，所有其它东西要一点点靠组装，或编译出来,注意这里加了root,init，以及norestore,copyfiles tce是为了restore，这里用不着
echo menuentry \"dbcolinux systemhd\" { >> /mnt/hda1/boot/grub/grub.cfg
echo  linux /boot/bzImage64 com1=9600,8n1 loglevel=3 user=tc console=ttyS0 console=tty0 noembed nomodeset root=/dev/hda1 init=/system/linuxrc swapfile=hda1 tce=hda1 opt=hda1 home=hda1 norestore >> /mnt/hda1/boot/grub/grub.cfg
echo } >> /mnt/hda1/boot/grub/grub.cfg
```

3，patch源码部分
-----

这些文件是经过修改的，在接下来的scripts会被释放覆盖相应的源码文件夹中的对应层次，形成patch的效果。所以它的结构保留了原源码文件夹的结构。然后打包成xx patch.tgz文件，有：linux-2.6.33.3_patchs，glibc-2.11.1_patchs，busybox-1.19.0_patchs,patched的部分请去文章3中去对照。也可以直接去源码库中找。

4, /scripts/2.build64toolchainandkernel.sh
-----

这里主要是文章2，原来脚本2，3的强化和修正：

```
……
echo "compiling binutils ......"
#这里设成shared，因为lib64中的ld要突出so，ld-*稍后需要复制一份出来
#shared模式下编译64bit bfd会出现ar:file truncated问题，所以要编译二次，fix一次tce-load compiletc里那个,它会安装到/usr/local/bin/
cd ../binutils-2.20 && mkdir b && cd b
../configure CC="gcc -w" -enable-64-bit-bfd > /dev/null
make > /dev/null
make install > /dev/null
cd ../../binutils-2.20 && mkdir b2 && cd b2
#只要binutils -enable-shared，那么用它编译的gcc必定也是-enable-shared,，会影响接下来c libs.cpplibs全是shared
../configure CC="gcc -w" -prefix=/mnt/hda1/system/toolchain -target=$TARGET -enable-shared -disable-multilib -enable-64-bit-bfd > /dev/null
make > /dev/null
make install > /dev/null
……

echo "compiling c lib startups ......"
#标准C库头文件和一些必要启动文件
#毕竟，通过export cross compile切换cross compile是不好的，我们干脆手动指定，一了百了
#如果cross compile gcc出错，出现fenv.h找不到那就是你改动了一些prefix文件夹，如果出现tls.h找不到，那么就是x86_64-pc-linux-gnu-gcc与gcc你没用对,compiling c libs，二处都是x86_64-pc-linux-gnu-gcc而不是gcc
tar zxf /mnt/hda1/tmp/base/glibc-2.11.1_patchs/Archive.tgz -C /mnt/hda1/tmp/base/glibc-2.11.1/
cd ../../glibc-2.11.1 && mkdir b && cd b
#在glibc眼里，这里的build和host一个意思，与标准的其它三元组意义不同
../configure CC="x86_64-pc-linux-gnu-gcc -w" -prefix=/mnt/hda1/system/toolchain/$TARGET -build=$MACHTYPE -host=$HOST -target=$TARGET -with-headers=/mnt/hda1/system/toolchain/$TARGET/include -disable-multilib libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes > /dev/null
make install-bootstrap-headers=yes install-headers > /dev/null
make csu/subdir_lib > /dev/null
install csu/crt1.o csu/crti.o csu/crtn.o /mnt/hda1/system/toolchain/$TARGET/lib > /dev/null
#利用新编的64gcc处理
x86_64-pc-linux-gnu-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /mnt/hda1/system/toolchain/$TARGET/lib/libc.so > /dev/null
……

echo "compiling c++ libs ......"
#标准C++库
#这是一个bug1
ln -s /usr/local/bin/file /usr/bin/file
cd ../../gcc-4.4.3/b/
……
make CC="x86_64-pc-linux-gnu-gcc -w" bzImage > /dev/null
make CC="x86_64-pc-linux-gnu-gcc -w" modules > /dev/null
make modules_install INSTALL_MOD_PATH=/mnt/hda1/system > /dev/null
ln -s /system/lib/modules/2.6.33.3-tinycore64/kernel /mnt/hda1/system/lib/modules/2.6.33.3-tinycore64/kernel.tclocal

cp -f arch/x86/boot/bzImage /mnt/hda1/boot/bzImage64
```

5,scripts/3.build3264rootfsbase.sh
-----

这是新加的整个脚本3：

```
export PATH=$PATH:/usr/local/sbin:/usr/local/bin

cd /mnt/hda1/tmp/base/
tar zxf busybox-1.19.0patched.tar.gz

echo "making basic rootfs lib,lib64......"
#gcc test.c一个demo，/usr/bin/ldd测试它，就可以找出要运行它的dys
#注意这里，复制32 ld-*,dylibs部分
#/mnt/hda1/system/lib/在前面脚本安装lib/modules时被创建了
cp /lib/lib*.so* /mnt/hda1/system/lib/
cp /lib/ld-2.11.1.so /mnt/hda1/system/lib/ld-2.11.1.so
chmod +x /mnt/hda1/system/lib/ld-2.11.1.so
#再来套/lib64下的
mkdir /mnt/hda1/system/lib64/
cp /mnt/hda1/system/toolchain/x86_64-pc-linux-gnu/lib/lib*.so* /mnt/hda1/system/lib64/
cp /mnt/hda1/system/toolchain/x86_64-pc-linux-gnu/lib/ld-2.11.1.so /mnt/hda1/system/lib64/ld-2.11.1.so
chmod +x /mnt/hda1/system/lib64/ld-2.11.1.so
cp /mnt/hda1/system/toolchain/x86_64-pc-linux-gnu/lib64/lib*.so* /mnt/hda1/system/lib64/
#链接,file ./ld-linux.so.2可查出来
ln -s /system/lib/ld-2.11.1.so /mnt/hda1/system/lib/ld-linux.so.2
ln -s /system/lib64/ld-2.11.1.so /mnt/hda1/system/lib64/ld-linux-x86-64.so.2

echo "compiling busybox ......"
tar zxf /mnt/hda1/tmp/base/busybox-1.19.0_patchs/Archive.tgz -C /mnt/hda1/tmp/base/busybox-1.19.0/
cd busybox-1.19.0
make mrproper > /dev/null
cp -f ../busybox-1.19.0-config configs/i686_defconfig
make i686_defconfig > /dev/null
#还可以加CROSS_COMPILE=
#有些文件编码问题会导致下面的-变成...，如果出现/usr/local/bin/ld rpath=/system/lib no such file no such file or directory，可能就是这类错误了,很诡异
make CC="gcc -Wl,–rpath=/system/lib -Wl,–dynamic-linker=/system/lib/ld-linux.so.2" CONFIG_PREFIX=/mnt/hda1/system/ install > /dev/null

echo "make basic rootfs data..."
mkdir /mnt/hda1/system/dev
cd /mnt/hda1/system/dev
mknod console c 5 1
mknod null c 1 3

mkdir /mnt/hda1/system/etc
cd /mnt/hda1/system/etc

touch fstab
echo proc /system/proc proc defaults 0 0 >> fstab
echo sysfs /system/sys sysfs defauts 0 0 >> fstab

touch inittab
chmod +x inittab
echo ::sysinit:/system/etc/init.d/rcS >> inittab
echo console::respawn:-/system/bin/sh >> inittab
echo ::ctrlaltdel:/system/sbin/reboot >> inittab
echo ::shutdown:/system/bin/umount -a -r >> inittab

mkdir /mnt/hda1/system/etc/init.d
cd /mnt/hda1/system/etc/init.d
touch rcS
chmod +x rcS
echo #!/system/bin/sh >> rcS
echo /system/bin/mount -a >> rcS

mkdir /mnt/hda1/system/proc
mkdir /mnt/hda1/system/sys

#clean
#rm -rf /mnt/hda1/tmp/base/
#reboot
```

—————

最后，进入新生成的grub系统条目，出现命令行要先export PATH=$PATH:/system/bin:/system/sbin，应该是patchs没做好。下回做。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336888/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



