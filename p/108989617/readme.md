普化群晖将其改造成正常磁盘布局及编译源码打开kernel message
=====

__本文关键字：群晖显卡支持，dsm 6 console tty,打开console，让3.x linux使用graphical terminal as console，改变黑群原生磁盘布局，一种动态调整压缩镜像文件的方法。压缩0空间，及在linux上离线操作raid/llvm分区的方法，离线折腾黑群镜像,调整黑群剩余空间为存储空间。initrd加载二个gz,dd会复制uuid吗,debian上grub-install会自带device-map导致grub rescue__

在《Dsm as deepin mate(3):离线编辑初始镜像，让skynas本地验证启动安装/升级》中我们讲过离线编辑镜像的需要，但是黑群是一种使用了软raid的系统（在云上，软raid是没有意义的。因为云盘可靠性很高）。它的镜像是一个多区段镜像有三个raid，主启动分区倒不是raid，然而被放在末尾而且很小只有32MB（nasyun老骥伏枥正是因为发现这个32m可以变动一下发明了它的脚本，稍后我们会贴出其镜像布局）另外内置在镜像中的存储空间也未能被认到，由于这个镜像的系统里并不推荐也不能重建存储分区，（本来就是一次性镜像），那么无妨将其纠正为正常的布局，把启动前变1G并提前，顺便将整个镜像当初定的50G缩小至一个较小的尺寸比如20G更为适用，识别/重建可用存储空间的工作也可以一起给做了，

> 以上我们正是为了把原生群晖改造成正常的linux系统的方式,方便以后集成PE和通用booter。实际上群晖是专用硬件附属系统，但因为它用了linux，普化群晖为linux发行也变得可能，比如还有人在群晖上加入其它系统移殖过来的binary，运行sshd,telnetd，甚至整个debain chroot fs等，我们本文一部分也正是这样的路子，我们先为群晖开tty（dsm61启动时停在booting the kernel，但serial口还在正常输出，群晖的serial output我们当然能在虚拟机上加个--serial或redirect to a file能试出来。我们尝试把它还原为直接在console上输出，类普通linux kernel）以便接下来调试，然后再变更原生磁盘格局。


建立本地测试环境local pve img create/edit/hosting and restore enviroment
------

我们将之前的《一键pebuilder，在云主机上安装deepin20，云主机在线安装dsm61715284》创建/编辑/下载镜像的方案，现在我们转到pve上来，我是在osx->pd上安装pve6.3上的（定义一个虚拟机打开nested虚拟化），实际上这货应该安装在baremetal上来（与exsi一样）。----- pve使用了我最喜欢的debian与js界面(只是这货安装时没有使用di有点遗憾)，而且它支持很精细的cpu类型，这就可以为我们之前《kvm上安装osx》的时候无法确定cpu类型提供帮助，

> 如何将以前的创建/编辑/下载方案弄到单点pve上来。而且做到一一对应
> 切换使用localstroage来存储镜像：
pve安装完后创建vm存放disk images的位置是默认的local lvm存储，这是一种虚拟逻辑存储，为了测试我们希望镜像直接存放在filesystem中，供我们dump用。
使用pvesm set local --content images或在web gui 的serverview->data center->storage的local上那里双击为local增加一个images options(这个默认是没有打开的)，这个打开的情况下，就可以从local storage中make disk image了，否则只有一个local lvm供你选。除此之外，你也可以删掉local lvm改挂载逻辑为local，将local lvm改为local，这个不推荐因为lvm存储在真正部署时还是有优势的。
只是这个界面不能像上传iso一样上传diskimage file，你可以使用qm (Qemu/KVM Virtual Machine Manager)pvesm (“Proxmox VE Storage Manager”)等工具通过命令行操作上传/转换镜像。
> 控制镜像细节：1,定义一个20G的磁盘，在定义虚拟机时填入20就可以了,2,或者进pve的命令行(ssh root@pve ip)，在里面打命令制造特殊布局的raw image。3,上传已有raw image，那个lxc就是raw image的上传处，不要怀疑,模板iso可以直接在web界面上传，也可wget到这里 /var/lib/vz/template/iso/ ，在定义虚拟机时就能看到了
如何获得pve中经过编辑的dd镜像,还是我们在ssh pve中的python -m SimpleHTTPServer大法(停虚拟机先)。
> 另外一个问题就是这种虚拟机中套虚拟机的方式，使得磁盘性能大大降低，没办法。忍受一下只是制作镜像用，另外据说pve维护一个仓库，可用普通debian安装这里的源，自己造一个pve，不过我们关注app级的虚拟化，并把它集成到os（造nativecloudstack），而不是os的虚拟

除了这个pve用来建镜像并servering，我们还得一个用来dibuilder.sh安装前一个VM restore做出来的镜像的虚拟机，测试dibuilder.sh的有效性（实际上pve上测试镜像通过就通过了我们这里只是测试dibuilder：利用《将pebuilder变成dibuilder.sh,将di tools集入boot层(2)》中的dibuilder.sh支持raw url as dd image src，可以在其中直接dibuilder -dd 'local result img'），用任意linux即可

一个不大不小的问题：我们发现在debian jessie 8.11上创建初始布局镜像 时发现：debian grub-install（apt search grub-pc，显示grub2-common/oldoldstable,oldoldstable,now 2.02~beta2-22+deb8u1 amd64 [installed,automatic]）与前面《利用增强tinycorelinux remaster tool打造你的硬盘镜像及一种让tinycorelinux变成Debian install体的设想》tinycorelinux 11上的grub-install表现不一,虽然最终grub-install都是no errors，但是debian上做的启不动，提示no such device,只认到hd0,ls没有分区。可能是debian上的grub集成的grub/i386-pc里面的脚手架文件不全，也可能是已完成安装的grub2在grub-install到镜像文件上时会带当时系统上自身device map，debian下的vi也有问题退格删除傻傻不分，但是有nano editor代替可用，故在一台tc11上而不是debian 811中，完成后的镜像我们上传到pve，

为了可以循环重来，我一般是在PD虚拟机中新建上面二台vm的当前快照，这样快，失败了也可以迅速恢复重来，

还原console
-----

为什么把这个放在前面，因为它可以使控制台观察群晖的输出变可能，作调试用。------ 在之前的文章中我们多次提到busybox init对tty的处理(1，我们主要是创建/dev/console(实际的终端）和/dev/tty0(主虚拟终端）设备文件,实际上tty0这个文件动态生成可以不要手创，否则为什么叫虚拟终端，2，然后在/etc/inittab文件中添加如下实现：ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100 # GENERIC_SERIAL(用vt100类型模拟):这里面的说法是，init使用生成（respawn）一个守护tty的getty过程，负责登录开一个终端（注意getty只是开终端不负责内核输出）你登录时会看到了login。最后 grub cmdline中append一条"console=tty0 console=ttyS0,115200")将必须的参数传给内核，完成内核信息开机启动所有的过程和登录接入，包括《avatt2编译》及最近的《将pebuilder变成dibuilder.sh,将di tools集入boot层(1)》中，我们都处理过。

我们来综合介绍一下：

> TTY 是 Teletype 或 Teletypewriter 的缩写，原来是指电传打字机，为裸机作输入输出交互服务，后来在出现了OS之后，这种设备逐渐被键盘和显示器取代。虽然现在物理终端实际上已经灭绝了，但它们都由硬件概念，逐渐演化成了os里软件模拟的终端概念(比如用graphic console/tty模拟text console或physical serial console,这实际上也是一种必要，比如我们需要ssh终端这种东西)，------ 其实，不管是电传打字机还是键盘显示器，还是软件模拟的显示设备，都是作为计算机的终端设备存在的起输出输入交互作用，所以 TTY 也泛指计算机的终端(terminal)设备。
> 提到终端就不能不提控制台 console。控制台的概念与终端含义非常相近，其实现在我们经常用它们表示相同的东西(比如显示器键盘是终端也是控制台，但在计算机的早期时代，控制台指大型控制设备终端仅限输入输出设备，基本上，控制台是计算机的基本设备，，而终端是附加设备，但它们都有输出信息的终端能力，比如系统启动时的消息只能首先输出到第一个控制台终端上，你也可以接入其它终端来定位这些信息)。所以说法上，有终端作控制台也有控制台as终端的说法。各倾向于指主要输出设备和次要输出设备的意思。
> 终端和控制台可能会有多种形式，，比如，除了上面提到虽然SSH,Telnet 等等作为p终端实现了对服务器的管理通过远程访问进行，还有比如模拟的物理串行s终端，ssh出来的p终端(用户级的)，本地显卡显示器出来的虚拟终端（os视角下的终端，(我们把它称为c终端，这也是一种kernel级tty)），他们各有各的用处对应各种场景，比如有些时候有些情况下，使用远程访问是无法解决全部问题的，如处理一些导致系统crash的错误或者bug的信息，这些需要通过Serial console来访问，它们需要模拟成软件层的虚拟终端接入OS。所以我们最终面对的是OS视角下的各种终端，
> linux说法下的专用虚拟控制台终端term=linux,有一些设备特殊文件与之相关联：tty0、tty1、tty2等。当用户在控制台上alt+f1-6切换时，使用的文件始终是/dev/console，或者你启动时没有设定 console= option (或者使用 console=/dev/tty0),此时的/dev/console就是/dev/tty0，------ To use a serial port as console you need to compile the support into your kernel - by default it is not compiled in. For PC style serial ports it's the config option next to menu option:
Character devices ‣ Serial drivers ‣ 8250/16550 and compatible serial support ‣ Console on 8250/16550 and compatible serial port，You must compile serial support into the kernel and not as a module. ------to use a graphic framebuffer console,你可以关注这些内核配置项，VGA text console,Support for frame buffer devices (EXPERIMENTAL),VESA VGA graphics console编译完成后，你的内核就支持framebuffer驱动了

> 对于init，有sysvinit和upstart这种，linux有0-7几种启动级别，在sysvinit中，/Linuxrc 执行init 进程初始化文件。主要工作是把已安装根文件系统中的/etc 安装为ramfs，并拷贝/mnt/etc/目录下所有文件到/etc，这里存放系统启动后的许多特殊文件；接着Linuxrc 重新构建文件分配表inittab；之后执行系统初始化进程/sbin/init。/mnt/etc/init.d/rcS 完成各个文件系统的 mount，再执行/usr/etc/rc.local；通过rcS 单用户脚本可以调用 dhcp 程序配置网络。rcS 执行完了以后，init 就会在一个 console 上，按照 inittab 的指示开一个 shell，或者是开 getty + login，这样用户就会看到提示输入用户名的
提示符(getty就是守护登录而已一个作用，外带开启终端)。/usr/etc/rc.local 这是被init.d/rcS 文件调用执行的特殊文件，与Linux 系统硬件平台相关，如安装核心模块、进行网络配置、运行应用程序、启动图形界面等。/usr/etc/profile rc.local 首先执行该文件配置应用程序需要的环境变量等。
> 但是很可能会发现在您的计算机上找不到/etc/inittab 文件了，这是因为它是sysvinit的，现在的新发行版要么使用一种被称为 upstart 的新型 init 系统要么使用systemd。sysvinit的缺点是串行同步的，这大大影响效率，UpStart 解决了之前提到的 sysvinit 的缺点。对于设备接入导致的复杂init变动等这些业务需求都可以满足，它采用事件驱动模型,工作配置文件存放在/etc/init 下面，是以.conf 作为文件后缀的文件。
目前不仅桌面系统 Ubuntu 采用了 UpStart，甚至企业级服务器级的 RHEL 也默认采用 UpStart 来替换 sysvinit 作为 init 系统。此外云计算等新的 Server 端技术也往往需要单个设备可以更加快速地启动。

我们的参考：

先去sf下载源码，怎么选呢？黑群5的XPEnoboot mod是可以直接像普通linux一样显boot message的，我们要修改的是61，6之前最通用的版是5.2，因此比较这二个版本：

> 它的源码分类是这样的(按硬件软件发布组合命名)，
toolchain下是dsm xx版本/cpu架构+linux版本+cpu型号的文件夹。比如我们选择dsm6.1/Intel x86 linux 3.10.102 (Bromolow)，在里面下载toolchain,https://sourceforge.net/projects/dsgpl/files/Tool%20Chain/DSM%206.1%20Tool%20Chains/Intel%20x86%20linux%203.10.102%20%28Bromolow%29/bromolow-gcc493_glibc220_linaro_x86_64-GPL.txz/download
nas source是具体版本分支号/cpu型号，比如我们选择15152branch/bromolow-source，在里面下载kernel source，https://sourceforge.net/projects/dsgpl/files/Synology%20NAS%20GPL%20Source/15152branch/bromolow-source/linux-3.10.x.txz/download
> 群晖的系统（硬件与采用的linux版本，只有一个版本号如ds3615xs）与硬件系统（系统磁盘上的rootfs，有二个版本号如5.2,6.1，和15152,5644）分离跨度很大是分开开发的，比如很多硬件系统还停留在引用3.x的linux版本上，兼容很稳定不冒进。

编译61源码，以下来自https://xpenology.com/forum/topic/7341-tutorial-compile-xpenology-drivers-in-windows-10/，在debian系上是一样的操作：```
apt-get install mc make gcc build-essential kernel-wedge libncurses5 libncurses5-dev libelf-dev binutils-dev kexec-tools makedumpfile fakeroot lzma
sudo sh -c "echo alias dsm6make=\'make ARCH=x86_64 CROSS_COMPILE=/root/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu-\' > /etc/profile"
source /etc/profile
cp synoconfigs/bromolow .config
dsm6make menuconfig

# 所幸，发现群晖把他们对linux的增强的地方都放在meunconfig enhancement section跟linux主体分开的，在serial driver栏可以看到tts0,ttys1是被保留使用的而且swap了,另外这些项也要关注一下，CONFIG_FRAMEBUFFER_CONSOLE,SYNO_X86_TTY_CONSOLE_OUTPUT,CONFIG_TTY_PRINTK,CONFIG_EARLY_PRINTK，调整这些参数，如果这个命令发现不了，可能你用了sudo)
# 在群晖中，上面提到的serial tty都是打开的，由于群晖这种东西早期是没有视频和视频驱动支持的，所以它只能用串行s终端作console来输出信息。或者ssh出来的p网络终端。而不倾向于使用本地显卡显示器framebuffer出来的控制台tty终端。所以一般linux都会有默认的kernel级tty支持和显驱支持，群晖内核配置中也当然有相关的选项。

dsm6make(这个时候千W不能想当然make否则调用的不是cross compile)
arch/x86/boot/bzImage

#出来的结果就可以用来置换内核了，大家都是x86 ds3615xs出来的内核+jun mod所以可以互换，测试时用任何rootfs as initrd arg即可，其实单纯一个linux kernel就可以启动计算机无须initrd，
```

发现61不带dmesg输出，同理下载5.2-braswell-gcc473_glibc217_x86_64-GPL.txz，5644-braswell-source.txz：这个由1.6G的重打包而来，编译5.2的源码(5.2碰到locale.c assert错误执行export LANG=C)并调整，发现也是不带dmesg输出的。看来52能输出是xpe mod的修改导致的：

于是去看52 iso(其实不必，在linux kernel启动就可以看到它支不支持console output dmesg，跟rootfs无关，我这里只是列举细节)，

由于xpenology的XPEnoboot_DS3615xs_5.2-5644.4.iso是将initramfs打包在内的(它用的lilo，而grub2和linux内核有tty驱动/mod支持，比如在grub2中也可设置fb分辨率)，难于解包，所以直接装一个进里面查看里面的文件，我们可能还要准备https://global.download.synology.com/download/DSM/release/5.2/5644/DSM_DS3615xs_5644.pat，得出观察结果：

群晖5使用upstart(在/etc/init/tty.conf中发现了exec tty，各处conf中发现了console output，dev/下发现了正常的/dev/tty*,ttyS，tty显示/dev/console,export 看到term=linuxecho dsfdsf>>/dev/console出现在当前，启动中发现ttyS0 enabled字样, synobios open /dev/ttyS1 success)，,dsm6版本之后使用用了一套很精简的init机制比如synoservice -restart ssh-shell,来启动ssh,6没有telinit,upstart和tty.conf，6 ps有getty，但6貌似在crontab启动它？

> https://smallhacks.wordpress.com/2012/04/17/working-with-synology-hardware-devsynobios-and-devttys1/这篇文章提到synobios和/dev/ttyS1是群晖内置二驱动二硬件，比如ttyS1用于响应面板上的按压操作。还提到它们与闭源的scemd的关系。
> 顺带去了xpenology的坛子，junmod这个折腾之源的出处(它的原理是，在grub cmdline中initrd一次加载二个rd.gz和extra.lzma，这样这二个就二合为一了。跟群晖menuconfig一样，在单独的patch中放置修改跟主体分开。）发现quicknick的mod ： XPEnology.Configuration.Tool.v3.0.Bootloader.DSM.6.1.x.zip，它似乎提到enable tty方面的东西。
https://xpenology.com/forum/topic/6566-xpenology-configuration-tool-bootloader-dsm-602-8451/
这个家伙把文件藏好深。 img里第4分区有一个.quicknick（包含quicknick.lzma）隐藏，xz quicknick.lzma还有一个.quicknick，这里是它主要定制的地方。他们俩的东西基本在这你都可以下到https://xpenology.club/downloads/


针对修改我们的dsm62尝试改动：


```
#修改extra.lzma中的init
#以下放在bin/patch后
/bin/sed -i '/SYNOLoadIPv6()/i\\TERM=linux' /etc/rc
/bin/sed -i '/SYNOLoadIPv6()/i\\export TERM' /etc/rc
#启动服务直接守护在console
/bin/sed -i '/SYNOLoadIPv6()/i\\/sbin\/getty 115200 console &' /etc/rc
#以下放在exec /sbin/init之后作测试
echo "test" >> /dev/console
echo "test" >> /dev/console

#修改grub.cfg中的函数
function showtips {


terminal --timeout=3 console serial

    terminal_input --append console
    terminal_output --append console
    
    if [ -n $has_serial ]; then
        terminal_output --remove serial
    fi
    echo "Screen will stop updating shortly, please open http://find.synology.com to continue."
   .....  
    echo
    if [ -n $has_serial ]; then
        terminal_output --append serial
    fi
}
```

失败，mark一下，以后解决！


新建一个镜像文件重建磁盘布局,重建p1,还原p2,p3内容(按扇区dd)
------

进入我们的目标讨论：将启动分区前提并扩大，缩容存储分区和整个镜像，有二种路子：除了按照《一键pebuilder，实现云主机在线装dsm61715284》从0开始产生定制尺寸的镜像，也可以直接针对现有镜像作缩小镜像并压缩0空间处理作出目标镜像（但是linux上的磁盘扩展压缩没有在windows下方便和工具支持多，如果不是当初设计为lvm镜像来的，后期的resizefs之类的程序操作并不保险。parted也支持用它分区出来的镜像的动态扩展但局限也多不是一种好方案），除了前面二种，实际上也可以在原来的镜像上重建分区格式然后DD得到，这种方式最简单直接而且干净，也是本文要讲述的方案。

我们新建一个20G的空白镜像，（其实可以把原镜像重分区缩为20G(不需要的部分不分区进mbr表，或删掉不让它们进入分区表)。然后直接dd，不用在新镜像上进行。）但为了过程更清楚化，我们还是利用新建镜像然后格式化的方式。我们用的是parted而不是fdisk，仅用fdisk -l,你如果在fdisk中n手动创建，可以创建4再创建3，这样可以跨过3自由定制4的大小。。原有50G镜像格式为：

```
sudo fdisk -l ./dsm61715284
/dev/loop0p1         2048   4982527                4980480  2.4G fd Linux raid autodetect
/dev/loop0p2      4982528   9176831             4194304    2G fd Linux raid autodetect
/dev/loop0p3      9437184 104652799           95215616 45.4G fd Linux raid autodetect
/dev/loop0p4 *    9177088   9242624             65537   32M  e W95 FAT16 (LBA)
```

我们看到很不美观，我们将启动分区提前为扩展为1G，方便塞入dipe和更多东西，其它分区不变，猜想这虽然不是黑群的原生布局，但猜肯定能启动和正常工作

准备三个脚本create.sh,mount.sh和umount.sh：

这是create.sh,part one:

```
# 脚本用的tce-load -iw parted dosfstools util-linux，losetup要换成/usr/local/sbin/losetup，home/tc而不是/home/td)。
#sudo apt-get install parted dosfstools

sudo dd if=/dev/zero of=dsm61715284new bs=1024 count=20971520
dev_dsm61715284new=`sudo losetup -fP --show ./dsm61715284new | awk '{print $1}'`
dev_dsm61715284ori=`sudo losetup -fP --show ./dsm61715284 | awk '{print $1}'`

# 分区1，分配2097152个扇区=1G，为了保持以后的分区对齐
sudo parted -s "$dev_dsm61715284new" mktable msdos
sudo parted -s "$dev_dsm61715284new" mkpart primary fat16 2048s 2099199s
sudo parted -s "$dev_dsm61715284new" set 1 boot on
# 从这里开始的是dsm的磁盘布局(跳xxx，跨xxx)
# 分区2，从上个分区的最后一个簇+2048+1作为开始（跳2048），+4980480-1作为结束(跨4980480)
sudo parted -s "$dev_dsm61715284new" mkpart primary 2101248s 7081727s
sudo parted -s "$dev_dsm61715284new" set 2 raid
# 这里有一个没对齐warning，没关系
# 分区3，从上个分区的最后一个簇+1作为开始(不跳)，+4194304-1作为结束(跨4194304)
sudo parted -s "$dev_dsm61715284new" mkpart primary 7081728s 11276031s
sudo parted -s "$dev_dsm61715284new" set 3 raid
# 分区4，从上个分区的最后一个簇+260352+1作为开始(跳260352)，100%剩余空间作为结束(跨至尾-1)
sudo parted -s "$dev_dsm61715284new" mkpart primary 11536384s 100%
sudo parted -s "$dev_dsm61715284new" set 4 raid

# 我们看到结果是正确的，mkfs有一个小warning没关系
sudo fdisk -l "$dev_dsm61715284new"
sudo mkfs.msdos "$dev_dsm61715284new"p1
```

安装grub2复制boot文件开始，sudo apt-get install curl binutils，bintuils是为了运行dibuilder.sh时能有ar
然后wget,chmod并运行dibuilder.sh -dd 'http://d.shalol.com/mirrors/dsm61715284.gz'，在完成等待10秒时ctrl+c，在/boot下得到initrd.img和vmlinuz,开始安装grub2

脚本part2:

```
#sudo apt-get install curl binutils
#sudo wget -cq http://10.211.55.2:8000/dibuilder2.sh 镜像中用的 http://10.211.55.2:8000/mirrors/debian/
#dibuilder.sh -dd 'http://d.shalol.com/mirrors/dsm61715284.gz'

mnt_dsm61715284new="/home/td/newp1"
mnt_dsm61715284ori="/home/td/orip4"
sudo mkdir -p "$mnt_dsm61715284new" "$mnt_dsm61715284ori"
sudo mount "$dev_dsm61715284new"p1 "$mnt_dsm61715284new"
sudo mount "$dev_dsm61715284"p4 "$mnt_dsm61715284ori"

sudo grub-install --boot-directory="$mnt_dsm61715284new"/boot "$dev_dsm61715284new"
sudo mkdir -p "$mnt_dsm61715284new"/boot/DI
sudo cp /boot/vmlinuz "$mnt_dsm61715284new"/boot/DI
sudo cp /boot/initrd.img "$mnt_dsm61715284new"/boot/DI

#然后你就可以继续往grub中增加东西,原来老骥伏枥那个是grub2的啊，一直以为是grub1的
sudo cp -r "$mnt_dsm61715284ori"p4/boot/grub/grub.cfg "$mnt_dsm61715284new"p1/boot/grub/grub.cfg
sudo cp -r "$mnt_dsm61715284ori"p4/boot/grub/DS3615xs "$mnt_dsm61715284new"p1/boot/
sudo cp -r "$mnt_dsm61715284ori"p4/boot/grub/themes "$mnt_dsm61715284new"p1/boot/
#sudo vi grub.cfg,把timeout改为50，把调用ds3615xs的地方也修正下，三条有意义的set ds3615xs img的menuentry最上面加个set root=(hd0,msdos1)，把判断tcpe7.2处的$cmdpath改为cmdpath，把调用$cmdpath的地方改为($cmdpath)，改为调用/DI/xxx

sudo dd if="$dev_dsm61715284"p1 of="$dev_dsm61715284new"p2
sudo dd if="$dev_dsm61715284"p2 of="$dev_dsm61715284new"p3
#不能直接sudo mkfs.ext4 "$dev_dsm61715284new"p4，要等到变成md2时格
#sudo mkfs.ext4 "$dev_dsm61715284new"p4
sudo umount "$mnt_dsm61715284new"
sudo losetup -d "$dev_dsm61715284new"
```

以上p2-p3按扇区复制，p1也处理了只有p3没有处理(大小不一不能dd)，我们重新还原这个布局里的文件,sudo apt-get install mdadm:

这里是mount.sh，调用方法 mount.sh 镜像文件名

```
# 以下参照《https://www.howtoforge.com/how-to-set-up-software-raid1-on-a-running-system-incl-grub2-configuration-ubuntu-10.04-p2》
# 从这里开始弃用dsm61715284ori和create.sh
[ $1 = dsm61715284new ] && dev_dsm61715284new=`sudo losetup -fP --show ./dsm61715284new | awk '{print $1}'`
[ $1 = dsm61715284 ] && dev_dsm61715284ori=`sudo losetup -fP --show ./dsm61715284 | awk '{print $1}'`
sudo mkdir p1ori p3ori p2new p4new

# 群晖会把每个新加的盘的第一第二分区都作为md0/md1成员加进去作raid
[ -b /dev/md0 ] ||  echo 'y' | sudo mdadm --create /dev/md0 --metadata=0.90 --uuid=5d279172:1ed20bf2:3017a5a8:c86610be --level=1 --raid-disks=2 missing "$dev_dsm61715284new"p2
sudo mdadm --detail /dev/md0 | grep "UUID"
# 如果dd过，那么下面一句可省
# mkfs.ext4 /dev/md0
sudo mount -t ext4 /dev/md0 p2new

[ -b /dev/md1 ] || echo 'y' | sudo mdadm --create /dev/md1 --metadata=0.90 --uuid=188d0705:2995b6c6:3017a5a8:c86610be --level=1 --raid-disks=2 missing "$dev_dsm61715284new"p3
sudo mdadm --detail /dev/md1 | grep "UUID"
# mkswap /dev/md1
# swap区mount不了
# sudo mount /dev/md1 p3new

# 单盘basic，注意与上面的不同
[ -b /dev/md2 ] || echo 'y' | sudo mdadm --create /dev/md2 --metadata=1.2 --name=yunnas:2 --uuid=61203c7b:83ed31ed:29334798:82e3a353 --level=1 --raid-devices=1 --force "$dev_dsm61715284new"p4
sudo mdadm --detail /dev/md2 | grep "UUID"
# 这个被mkfs.ext4过一次，因此可以mount
# mkfs.ext4 /dev/md2，也可以brtfs
sudo mount -t ext4 /dev/md2 p4new

# 现在把dsm61715284ori也mdadm --create，sudo mount所有/dev/mdxxx,如果挂载失败你可以把looppx单分区按开头结束chs挂载，像《《Dsm as deepin mate(3):离线编辑初始镜像，让skynas本地验证启动安装/升级》》文一样，最后复制文件-p把权限也带上：

[ -b /dev/md3 ] || echo 'y' | sudo mdadm --create /dev/md4 --metadata=1.2 --name=yunnas:2 --uuid=61203c7b:83ed31ed:29334798:82e3a353 --level=1 --raid-devices=1 --force "$dev_dsm61715284ori"p3
sudo mdadm --detail /dev/md3 | grep "UUID"
sudo mount -t ext4 /dev/md3 p3ori

# 最后，p4按文件复制,p2其实也可以这样只是被DD过了 # cp -dpRx "$mnt_dsm61715284ori"p3ori "$mnt_dsm61715284new"p4new

sudo cat /proc/mdstat
```

在mdadm.conf上使用的是RAID设备的UUID,而不是文件系统blkid等出来的UUID.supperblock应该就是相当于raid的“pbr”之类的东西，在create时就会写入，DD会保留这种块信息

以上uuid是从打包到extra.lzma中获取的（解压： sudo sh -c 'xz -dc extra.lzma | cpio -id' 压缩： find . | cpio -c -o | xz -9 --format=lzma > extra.lzma），也是p3/etc/mdadam.conf中的内容,最好自己解包确认下。

以上脚本，还有就是我也不能确定超级快信息不能用多次create来覆盖，这是否一个once写入动作，所以首先查看原来的superblock,使用mdadm -E命令查看一下/dev/loop0px的情况：mdadm -E /dev/loop0px,如果有，可以使用：mdadm -As /dev/md0 --config /xxx/mdadm.conf重新启动这个raid，而不是重新创建


这是unmount.sh

```
sudo umount p2new
sudo mdadm -S /dev/md0
#sudo umount p3new
sudo mdadm -S /dev/md1
sudo umount p4new
sudo mdadm -S /dev/md2
sudo umount p3ori
sudo mdadm -S /dev/md3

sudo cat /proc/mdstat
```
umount.sh可多运行几次直到所有md显示stopped(-S不会抹除md块信息),我是直接在umount.sh前就tar dsm.gz dsm.raw的，这样镜像会有点不干净,打包的时候会显示镜像文件已更新，重打包即可)。

管不了那么多，进入系统(md0,md1不认出来也会停在安装设置界面)，结果失败，mark！以后再弄！

看来，要让群晖承认它的非原生布局，必须要绕过/usr/syno/bin/synocheckparition?它在闭源的scemd中,或者暴力修改linux.syno脚本中的判断，另外，(群晖里/usr/syno/sbin/synopartition --check也可在开放ssh的群晖里面使用。你可以用它check下列镜像的一个缩小版本)，至于那个存储分区(md2)（volume识别是无关紧要的问题，应该主要是修改/etc/mdadm/mdadm.conf里的md2.不过听说存储区重建还有《利用整块化自启镜像实现黑群在单盘位实机与云主机上的安装启动》结尾处提到技术，《群晖二合一版本无需进PE系统或DiskGenius 扩容的方法》，《引导二合一群晖系统的一键脚本动态扩容！》等技术）。


-----


我们可以把deepin,winsrv2019core,osxkvm都可以做成这样的含dipe,tdpe的多区段镜像。grub2也有windows版，甚至ntboot也做了一个类似linux/initrd的命令(http://bbs.wuyou.net/forum.php?mod=viewthread&tid=417545 《NTBOOT & wimboot for UEFI GRUB2》不过可能与winsrvcore19的bootmgr配合不了，所以可以用类似以前《黑苹果》方案中用memdisk+clover.iso的方式用memdisk+ntboot.iso


