云主机装黑果实践（5）：重得到镜像和继续强化前置启动过程
=====

__本文关键字：,hackintosh Cirrus Logic GD5446，OS X,GD5446 kext，黑苹果 vga兼容 花屏，黑苹果 虚拟机 显卡驱动__

现在是4.4全国哀悼日，没有从天而降的超人只有敢于挺身而出的凡人

在前文中，我们解决了大量启动测试前的关键问题，但还不算进入了真正的黑果适配机型的关键步骤，因为有一些前置问题依然存在，
1，关于前文那个编译后的变色龙有时在grub2+memdisk下启不动的问题，有没有永久解决法，有没有解法使kvm的ignore_msrs在guest端实现？
2,鉴于前面文章提到的云机适配问题，有没有方法在qemu中尽可能虚拟出接近云的guest配置，在本地深度测试后再上传镜像
3，我们在前面编译的trunk2922不是enoch的，跟社区的chris1111（能用于在文章2,3中正常安装启动的boot,cdboot）的boot存在大小相差，引导效果也不一样，前者启动后只进行了很小的起步。尝试chris1111(基于enoch)的可以进行到更多的进度,不加EasyMBR A5,A6可进系统。
4,在镜像中放installer还是完成安装后的osx，osx有没有类winsetup的驱动部署过程。
5,Mojave 1014是否真正选型正确？是否搭载了必备关键驱动。留给我们的工作多不多，有没有不可克服的？
6,是否有改用clover的必要？

我们将继续一一处理：

新qemu配置方案
-----

对于1，暂时没找到永解法，但找了一个新的暂时解法，把linux16,kernel16后面的16删掉ctrlx会halt，然后重新启动，系统进入到变色龙的概率会更大，

对于2，，一般的qemu/osxkvm用的都是q35机型，云主机是1996年的if440x，这种芯片影响了osx的支持设备，与能不能启动直接相关，比如机型的插槽都绑有具体功能，if440x不支持pci上的virtioblk启动iso，会卡在waiting for root device这是硬件导致It doesn't boot with the disk in virtio and scsci-virtio mode but boot in scsi.（启动分二段，第一段是变色龙的启动，稍后会详细说明kernel之后的启动调试也有卡root这里仅是变色龙启动），变色龙检出Cpu is intel xeon platinum 8163 cpu,Intel Xeon Platinum 8163（Skylake）,这是阿里云第四代服务器采用的CPU，但本机模拟不出这种U，vga不能用std，只能用cirrus-vga,但这种显卡在虚拟机中实测启动花屏，但安装过程依然可辨，在云主机上直接黑屏不能继续调试，但还是努力尝试一下，找出一套尽可能在保留if440x全局设定下启动osxinstaller的配置（毕竟阿里云不能改，其它的可以通过在osx端和变色龙端软性更改，比如尝试让virtio在osx端发挥作用前面说到这属于kernel之后的启动调试后来解决），所以以下，硬盘加了achi，显卡暂时没处理：

```
qemu-system-x86_64 -enable-kvm \
 -machine pc-i440fx-2.8 \
 -smp 4,cores=2 \
 -cpu Penryn,kvm=off,vendor=GenuineIntel \
 -m 4096 \
 -usb -device usb-kbd -device usb-mouse \
 -device cirrus-vga \
 -device ahci,id=ahci \
 -device ide-drive,bus=ahci.0,drive=MacHDD \
 -drive id=MacHDD,if=none,format=raw,file=./osxkvm \
 -device virtio-net-pci,mac='52:54:00:c9:18:27',netdev=MacNET \
 -netdev bridge,id=MacNET,br=virbr0,"helper=/usr/lib/qemu/qemu-bridge-helper" \
 -boot order=dc

#暂时不支持未实现
#-device virtio-blk-pci,drive=MacHDD \
#-drive id=MacHDD,if=none,cache=writeback,format=raw,file=./osxkvm \
#使用bridge网络,对于网卡，主机要有tap0先行创立起来，记得启动virtmanager中的那个virbr0哦，最好设为自动启动onboot，上面的device前drive后端定义才起作用。osx支持virtio net才起作用。
sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
mkdir /etc/qemu
touch /etc/qemu/bridge.conf
echo 'allow virbr0' > /etc/qemu/bridge.conf
#简单起见，除了ahci,pci都没有写slot位，功能位等，有默认值也可手动boot进tcpe。tce-load -wi pci-utils.tcz，然后lspci或lspci -t：以树状结构显示PCI设备的层次关系，包括所有的总线、桥、设备以及它们之间的联接,pci' domain='0x0000' bus='0x00' slot='0x07' function='0x2' 有四个域，或lspci -nn，或命令是lspci -vmm查设备ID与供应商ID，然后填上,我看到的是vga2 nic3 hd5
```

未来考虑使用Libvirt xml file format

新enoch变色龙
-----

对于3，我们需要重新checkout编译enoch branch的源码而不是trunk的(变色龙作为Appleboot https://opensource.apple.com/release/macos-10133.html和开源系统OpenDarwin的继承被社区用来作引导白果，通过KernelPatcher和FileNVRAM，变色龙已经支持纯白苹果免改iso只在内存中patching的安装方式https://www.insanelymac.com/forum/topic/299296-kernelpatcher-source-code/)，除了enoch branch,Chimera是另外一个有名的branch,变色龙 Enoch r2880 版本起 功能 变更如下: 1.内嵌入 FakeSMC.kext 2.新增 KextsToPatch,使用文件 /Extra/kexts.plist 3.新增 KernelToPatch,而trunk2922依然没有这些。它的kernelpatcher module和filenvram是外置的（vs enoch内置的功能很欠缺内置只有一个acpi patcher，而且丢进extra/modules，会卡在tmsafey.kext或AppleKextExcludeList.kext ）,启动时没有[ KERNEL PATCHER START ][ KEXT PATCHER START ]。好多都没有发挥作用，—— 奇怪的是，它的make pkg结果居然是有跟enoch 2922一样的kernel patcher项的。有趣的是，我发现chameleon也有menuconfig。

svn -co HEAD http://forge.voodooprojects.org/svn/chameleon/branches/ErmaC/Enoch,Makerule中的CFLAGS	= $(CONFIG_OPTIMIZATION_LEVEL) -g -Wmost -Wno-expansion-to-defined中的Werror去掉,修改一些源码加入从enoch删掉的-v debug部分：

```
在boot2/boot.c中能发现verbose(）的地方加个if (gVerboseMode) pause();
// ErmaCverbose("\n");if (gVerboseMode) pause();

kernel_patcher_internal.c中：
verbose("[ KERNEL PATCHER START ]\n");
if (gVerboseMode) pause();
verbose("[ KEXTS PATCHER START ]\n");
if (gVerboseMode) pause();

把加载kext的详细过程verbose出来并适当停顿，因为有时候在这里也会发生黑屏，卡屏，重启情况。这样肉眼可以看到。
drivers.c中：
verbose("Attempting to load drivers from standard repositories:\n");
if (gVerboseMode) pause();
verbose("Attempting to generate and load custom injectors kexts:\n");
if (gVerboseMode) pause();

我们把加载kext的verbose也做出来,并且10条一暂停。Enoch的Hfs.c里面Read HFS+ file被注释了，我把把它弄回来，依然在drivers.c中,这几句放FileLoadDrivers（）index=0和 ret=getdirentry开始处:
index = 0;int plistverbosecount=0;while (1){	if ((plistverbosecount > 9) & gVerboseMode)	{		pause();		plistverbosecount = 0;	}	
		plistverbosecount++;
		ret = GetDirEntry(dirSpec, &index, &name, &flags, &time);		…	}
….
}这几句放long LoadMatchedModules( void )处:
int exeverbosecount=0;
while (module != 0)内：
	if ((exeverbosecount > 9) & gVerboseMode)	{		pause();		exeverbosecount = 0;	}
	
	exeverbosecount++;

对于启动过程中存在System uuid 不存在，fixing 00112233-4455-6677-8899-AABBCCDDEEFF的错误，这并不是导致卡waitting root device的根本原因。应该是硬件驱动不起来或驱动没识别。但是可以去掉这个错误：
在smbios.plist中加入以下：
<key>SMsystemuuid</key>
<string>88D1F108-02EE-3622-99CE-EA53BEA03BD9</string>
或Libsaio,smbios.c->uint8_t *fixSystemUUID()中修改
```

最后，sudo make clean，sudo make pkg或dist，得到新boot和cdboot

新installer镜像和tcpe，测试启动
-----

对于4，虽然不能找到osx安装没有驱动部署过程的证据，但找到了一个新的简便办法制作installer镜像，发现osx下操作raw image也很方便(linux处理镜像之后的结果在osx的磁盘工具中结构视图是mixed一团的也不好看qemu-img convert -f dmg -O raw也不好用)，可以直接：

```
dd if=/dev/zero of=osxkvm bs=512 count=52428800
diskutil partitionDisk disk3 MBR fat32 BOOT 100M JHFS+ OSXKVM R JHFS+ OSXKVMINSTALL 10G
（installer本来8G就够，但需要手动替换1G的Extensions故10G，installer放最后可在安装完系统可磁盘工具把它合并到OSX，OSX留15G刚好总25G刚好也是轻量的磁盘大小）
hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount osxkvm
（仅attach,没有可装载的文件系统，一定要使用nomount）
挂载OSXKVMINSTALL,安装EasyMBR-Installer1013，不要安装变色龙bootloader,安装后，OSXKVMINSTALL会由日志式变成非日志式hfs
fdisk -e /dev/rdisk3,flag 1,write,yes,quit。
（fdisk: could not open MBR file /usr/standalone/i386/boot0: No such file or directory 没事，略过）
然后整个卸载回到linux装grub2,memdisk和tinycorelinuxpe
（sudo deepin-editor grub2.cfg，删掉所有内容，只保留二段menuentry，和一个全局set timeout久一点）。

在这里，你还可以制作一个带mount hfsplus免linux中关闭hfs+ journal内核（>3.4）的新tinycorelinuxpe。tc7是4.2的kernel，也有一个hfyprog.tcz(仅限x86)，进64bit deepin15编译32bit kernel里，gcc -v是630版，从http://mirrors.163.com/tinycorelinux/7.x/x86/release/src/kernel/Wget linux-4.2.9-patched.tar.xz和config-4.2.9-tinycore，sudo apt-get install gcc-multilib g++-multilib module-assistant libncurses5-dev libncursesw5-dev genisoimage( genisoimage 已取代之前的 mkisofs ）,cd linux src根，Make mrproper,make menuconfig,加载config-4.2.9-tinycore,加上virtio(按/查找virtio，发现有virtio_blk,virtio_net,virtio_pci),再集成hfs文件系统(/查找hfs,发现hfs_fs,hfsplus_fs和hfsplus_fs_posix_acl)，save成.config，再额外打个防vmlinuz启动时出现“failed to allocate space for phdrs system halted”的bug补丁（见https://bugzilla.kernel.org/show_bug.cgi?id=114671t和https://bugzilla.kernel.org/attachment.cgi?id=209601&action=diffr按这个手动改下源码）

之后就可以make bzImage了，cp to vmlinuz,用http://mirrors.163.com/tinycorelinux/7.x/x86/release/distribution_files/core.gz和它组成新的virtiope,然后制作iso:把tc7的isolinux提取出来，cd tcpe,sudo genisoimage -R -J -T -v --no-emul-boot --boot-load-size 4 --boot-info-table -V "tcpe" -b isolinux/isolinux.bin -c isolinux/boot.cat -o ../tcpe.iso . (genisoimage很奇葩，-b 和 -c里面不能写绝对路径，应该写的是相对iso的相对路径，否则找不到boot catalog) 

grub2也可直chainloader boot0，比如fat32装变色龙dd if=/dev/rdisk1s2 count=1,bs=512 of=origbs,cp boot1f32 newbs,dd if=origbs of=newbs skip=3 seek=3 bs=1 count=87 conv=notrunc,dd if=newbs of=/dev/rdisk1s2 count=1 bs=512，menuentry写insmod hfsplus,set root='(hd0,3)’,search --no-floppy --fs-uuid --set beec0ed4-a131-3ad3-9406-35cd8ebfb1b4,parttool (hd0,3) boot+,chainloader (hd0,1)/boot/chameleon/boot0,blkid显示uuid或tune2fs -U c1b9d5a2-f162-11cf-9ece-0020afc76f16 /dev/sda5更改uuid，有些电脑不支持 grub2下uuid的搜索，会Error：no such device:3c7c1d30-86c7-4ea3-ac16-30d6b0371b02，不过我们还是使用自己的grub2+memdisk+iso方案(iso中的文件系统是个hfs+,en0 0，它并不是fat32)。
```

镜像做好虚拟机可以直接用，上传到云机，installnet.sh如果https你要支持https，要wget --no-check-certificate，轻量的vnc后台窗口如需滚动窗口，请将鼠标移动到窗口左右两侧空白处，再进行滚动操作。。

现在要启动了，org.chameleon.Boot.plist几乎是总控文件，所有参数都在这里，kernel.plist,kext.plist只是patchs定义（注意KernelBooterExtensionsPatch to load extra kexts besides kernelcache），Themes文件夹和gui相关就不要加了。Extras/Extensions中不要加fakesmc，因为被enoch内置了，如果你碰到，ebios error，Startup error pause 5s,一般都是不正确kext导致的。

```
重点的org.chameleon.Boot.plist中的启动参数:

<key>Instant Menu</key>
<string>Yes</string>
<key>Default Partition</key>
<string>hd(0,3)</string>
<key>Timeout</key>
<string>10</string>
<key>Kernel</key>
<string>kernel</string>
<key>CsrActiveConfig</key>
<string>103</string>
<key>UseKernelCache</key>
<string>Yes</string>
<key>Kernel Flags</key>
<string>darkwake=0 npci=0x3000 debug=0x100</string>
```

同时kernel flags千W不要加-v，也不要加-f，毕竟这些都可以启动时按任意键手动加，也是后期调试必须手动敲入喂给kernel的部分。加debug=0x100（Don't reboot on panic，pause），但千W不要加wait加了wait=yes会黑屏，至于其它参数也是手动加的不要加进文件内（“-F”可以忽略org.chameleon.Boot.plist中的kernel flags）：

"-x" 安全模式，载入最少的kext。which ignores all cached driver files.
"-f" (默认是 No) Lion 专用，选用 Yes 将载入预链接的 KernelCache，并忽略 /Extra/Extensions 和 /System/Library/Extensions 及 Extensions.mkext。建议在 KernelCache 已内含所有必要的驱动时，才启用。
使用 (-s) 单用户模式进入，可于在开机进入命令模式排除问题。也许bdmsg就是在这里用的，不过这仅限能启动进入console kernel

基本上，解决和完成了问题1，2，3，到这里，变色龙级别的启动是可以无阻进行的，出现问题的，大部分都是darwin kernel启动之后出现的，像上面提到的waitting root device。我们把变色龙的启动和kernel的启动分开，前面讲到的都是变色龙的，这里开始我们讲kernel的启动，，After the KEXTs decompression phase, you should see the kernel booting ，TMSafetynet.kext 是最后一个,之后便会starting Darwin kernel（它本来进入显卡的图像模式，应该换一个分辨率显示文字，）。启动文字在虚拟机上花屏变色（正常是白色），最后IOConsoleUsers: time(0) 0->0, lin 0, llk 1, IOConsoleUsers: gIOScreenLockState 3, hs 0, bs 0, nov 0, sm 0x0错误无法继续，然后apfs module stop，在云主机上黑屏，无后续，免v下，虚拟机图形有进度条，云机无。

看来，即使是这样，虚拟机跟云主机也不是一一对应的，云主机属于一个更小的集合。qemu机->qemu guest机->qemu云主机
就像同显卡qemu方案云主机会黑屏虚机不会。

驱动，以及影响启动的综合未完成问题
-----

对于问题5，前面说的黑屏，卡住都是驱动问题（gIOScreenLockState错误原因是系统无法识别出你的显卡驱动,所以这个黑屏，是kernel的问题，不是变色龙的。），是1014不含这些驱动吗？而且10.14包含virtio,virtgraphics这些云机驱动（只是我们可能作很多黑果方面的工作使它工作：识别和驱动，在前面，我们按EasyMBR-Installer1013脚本A5,A6做的工作也把virtio,virtgraphic这些驱动做到S/L/E中去了，SLE:system,library,extensions）。

所以这问题大了去了，（为什么我们执着于10.14，14较新icloudrive支持高我们黑果的目的主要就是这，10.10-10.15的UI是一系的不排除内部实现有版本大更），我们从综合高度来思考这类问题：

硬件与bios与boot与os包含：QEMU uses the PC BIOS from the Seabios project and the Plex86/Bochs LGPL VGA，Seabios与qemu绑定bundle SeaBIOS 1.9.4 binaries with QEMU v2.7.1，就是变色龙no gui启动时显示的那个qemu，由qemu仿真的cirrus VGA就是一种典型的SVGAhttps://www.kraxel.org/blog/2018/10/qemu-vga-emulation-and-bochs-display/，https://www.kraxel.org/blog/2019/09/display-devices-in-qemu/.但是云主机lspci -vnn | grep VGA，虚拟是的是一块Cirrus Logic GD5446，为1996年pci vga的显卡VBE 2.00(VESA BIOS EXTENSION)，显存2-4mb(云机的bios是什么版本?)。Mojave ships with a driver for qemu stdvga and qemu cirrus vgahttps://www.kraxel.org/blog/2019/06/macos-qemu-guest/.osx从Yosemite开始不支持VGA,从skylake平台开始砍了原生vga通道，最新的Mac电脑都是没有VGA接口的，所以苹果的驱动里面理所当然就没有VGA接口的描述信息，macOS Mojave 10.14.x开始不支持NVIDIA显卡,苹果早就把 Mac下 Intel核显驱动 中 vga接口代码 全部移除了,最惹人注意的就是A卡的一堆、hd3000的几个、N卡的几个加上高通的无线网卡驱动，(没驱动的时候，vga有画面，只要驱动就黑屏。集显DVI没有问题，集显vga有问题)对于已有驱动，如AppleHDA.kext，Apple也已从其中删除了大量的Layouts，苹果就是这样一个追新弃旧派。

驱动工作模式与os支持：vga兼容模式一般是显卡在刚开始启动系统还没有加载显示驱动的时候，ramfb is a very simple framebuffer display device. It is intended to be configured by the firmware and used as boot framebuffer, until the guest OS loads a real GPU driver.If your guest OS supports the VESA 2.0 VBE extensions (e.g. Windows XP) and if you want to use high resolution modes (>=1280x1024x16) then you should use this option.对于linux：You have two Cirrus graphic options in the kernel, CONFIG_DRM_CIRRUS_QEMU is the KMS framebuffer that fit with qemu-kvm with a Cirrus virtual video card. The help say that it's only appropriate for virtualisation with the use of the modesetting Xorg video driver. So here, the use of the cirrus Xorg video driver is not what they recommand. 
You have also the CONFIG_FB_CIRRUS option that is a framebuffer driver for real Cirrus graphic cards. This framebuffer driver support the Qemu Cirrus emulated graphic card, but only tests can say fi it work with. 对于osx：https://www.kraxel.org/blog/2019/06/macos-qemu-guest/中提到virtualgraphics.kext也有Cirrus framebuffer，但并没有深入讲。这有可能是个efi驱动,那个simple osx kvm仓库中*.fd的补丁，就是OVMF（UEFI对qemu的实现）,Note that OVMF has a Display Resolution Setting too.那个virtio blk可能也是。

boot与os的关系：我们知道，能启动osx是qemu+firmware/boot(grub,chameleon)+硬件+osx的合力效果,grub有grub的文件，osx有osx的文件，变色龙有变色龙的文件，grub2与变色龙都属于osx前面一级的boot，这里grub2+wow.pc变色龙，grub2只负责boot，没有 firmware的作用，对于linux它有，比如它传给kernel的参数（这种关系就像变色龙之于osx），对于osx没有，我们发现grub2的mods很多与变色龙相同，而且用memdisk启动，存在依赖关系。会模拟出一个en(0,0)，Grub2和变色龙，也都可以在boot级就改变显示模式和效果。共享同样的boot技术。比如GRUB2也有insmod vga,insmod video_cirrus，it uses VESA BIOS Extensions to display the boot menu with enhanced resolution (and color depth), 当变色龙加osx出现Rooting via boot-uuid from xxx,Waiting on ioproviderclass时我们看到也提示了uuid，跟grub2一样no such device。
结合前面《linux as boot》我们讲了linuxboot和一个集成开发层到boot的东西，这样os下来就全是这种语言的app，甚至不必处理驱动硬件，python能在bootloader运行无须os也是一个例子BIOS Implementation Test Suite(BITS)。可以看出：其实boot就是firmware，它作为boot的意义是很浅一部分，驱动硬件，才是最主要的。

对于问题6，不能，理由无

————————

对于现在的问题，virtio不能在云机osx驱动，虚机cirrus-vga花屏云机黑屏，有没有继续的必要？苹果的驱动里面没有VGA（但不是不可以从别处找到用于Intel GPU的VGA接口的代码，甚至第三方）Apple也已从其中删除了大量的信息，（但修改Injector这些kext修补缺失部分可以设备正常工作。）甚至修改dsdt，edid注入 —— 所以还是值得试下。

下文将力求解决显卡和硬盘驱动问题，真正实现云上黑果。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338272/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



