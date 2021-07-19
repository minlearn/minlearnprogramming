云主机装黑果实践（7）：继续处理云主机上黑果前后置问题，增加新boot
=====

__本文关键字：fakecpuid apple logo stuck,变色龙fakecpuid,Cascadelake fakecpuid,qemu define cpu model,Skylake-Server qemu hackintosh__

我们知道，云上能启动黑果是上下一条路径上包括硬件，bios,boot,os在内合力的效果，硬件上的CPU又是这条路径上产生变数最多的因素，变色龙处理cpu兼容是一关，kernel处理cpu兼容是它独立的一关，这二个都存在这样的过程。而变色龙与kernel对cpu的定义不一样，存在不一样的处理过程，所以可能导致不能把cpu信息传给kernel最终发生黑屏这样的过程（更何况硬件/模拟器上，qemu对指令集和CPU支持也不一样），因此我们需要重新全盘统筹。

新的qemu和新加的boot
-----

如何判断阿里云上qemu的版本？除了搜索引擎，也许只有实测靠谱，如何在guest体内判断host的qemu版本。好在阿里会把qemu版本信息写在系统里，我们在轻量上执行dmidecode，出来BIOS Information Vendor: SeaBIOS Version: 8c24b4c Release Date: 04/01/2014，System Information：Manufacturer: Alibaba Cloud Product Name: Alibaba Cloud ECS Version: pc-i440fx-2.1。

因此我们重新编译qemu with deepin1511中的gcc630：

```
wget http://wiki.qemu-project.org/download/qemu-2.1.0.tar.bz2
sudo apt install libpixman-1-dev bison flex libsdl-dev autoconf automake libtool
(不加sdl会卡在vnc server，注意这里是sdl不是sdl2) 
(不加auttools会autoreconf: not found，不加flex bison在install时会出错)
./configure --prefix=/usr --target-list=x86_64-softmmu --enable-sdl
make install
```

这个出来的bios是BIOS rel-1.7.5-0-ge51488c-20140602_164612-nilsson.home.kraxel.org，离20140401很接近了。就这样吧。这样qemu版本就适配了

除了qemu，enoch2922本身就是个问题，因为其中根本就没有对最新cascadelake的定义和支持，查看src/libsaio/platform.c,platform.h我们看到它支持的CPU和指令集，三src驱动高重我们的调试思路是确保这三者从上而下，都有同样的关于cpu正确逻辑的继承实现或屏蔽（或者像Penryn一样能启动就行，可能仅仅因为cpu的问题在这这三者不冲突）。但另一个有名的boot：clover可能有:

Clover is a later revision of Tianocore. Both are 'firmware in RAM' replacements for UEFI firmware.
It allows you to boot in MBR\CSM mode and then run Clover which acts as a 'pseudo-UEFI boot manager', allowing you to boot to a UEFI OS from an MBR\CSM boot.2015-10-16 Rev 3289 新增 Skylake CPU 及 核显 及 SMBIOS 支持。而且，clover更强大有更先进的gui wizard，高于mbrpatch指定的Clover r4514+ boot 10.14 fine的版本选择也多，clover下也有直接把log保存在第一个分区的功能（而变色龙仅能得到bdmdg，Bdmsg其实是仅直到变色龙启动完就停止的。/var/log/system.log下有kernel启动的log）。不妨再添加一个boot同时测试？

尝试再添加一个boot为四叶草，直接从http://sourceforge.net/projects/cloverefiboot/files/Bootable_ISO/下载以wowpc.iso同样的方式启动。我们知道clover同时支持引导bios和uefi，也可以在实机或qemu（+pc-i440fx）下启动https://passthroughpo.st/hackintosh-kvm-guide-high-sierra-using-qemus-i440fx-chipset/，一般地，我们使用grub2+memdisk+clover.iso这种干净的老方案（网上还有把clover做成dmg，或grub chainloader boot0ss文件或chainloader pbr文件的方案），然而最新的clover.iso在qemu（+pc-i440fx+hd or virtio）时，会在引导clover时进入一个头部带有上述dmidecode部分信息的图形化虚拟UEFI BIOS 启动界面（这实际上是没有发现盘，因此无法加载EFI文件夹，因此无法加载/EFI/BOOT/BOOTX64.efi这个bios启动文件，在图形化efi界面文件浏览，你也根本看不到盘符），这实际上跟clover iso版本采用的boot也有关系（clover iso有一个cdboot，这相当于clover install较旧版本的默认boot，clover install较新版本没有默认boot，只有多选方案，但也只有二个，一个boot6,一个boot7，Clover ISO cdboot集成的默认是boot6，boot6,7启动效果不同），跟盘性质也有关系(一般clover更好支持bios+gpt+普通ide+fat32，当然它也承诺支持bios+mbr+hfs+)，跟你的硬件/qemu也有关系(除非你编译qemu2.1时加了CONFIG_VIRTIO_BLK_DATA_PLANE，否则virtio在bios下是无法识别的，grub2 insmod hfs hfsplus part_apple ->memdisk>clover.iso也不能把这种效果传递给clover。)

我们最终选择是Clover-v2.4k-4630-X64.iso，分别在实机下测试和云上测试，盘性质为普通ide hd，且EFI位于这个分区内，可以进入到clover选单界面，不受支持的virtio就只能停在图形化efi界面，如何实现让其在virtio或云机上也能用呢？。

我们直接dd把文件弄出来，也不用去编译clover的源码了。
Cdboot:452,608-boot6:450,048=2560字节Byte，512Byte是1簇，是5簇
把cdboot与boot7放一起：然后 dd if=boot7 of=cdboot seek=5 bs=512 conv=notrunc
Fat32 BOOT根分区是不需要放BOOT文件的,只需要把EFI从ISO中复制到这里来，其实移动也可。(对比变色龙，EFI就相当于其Extra)
然后mbrpatch cfg用变色龙那套Remaster Clover ISO

测试可以启动。以后我们就变色龙和四叶草一起测试了。如果你想测试clover install pkg，或得到相关文件，在10.12,13虚拟机guest osx中进行，，复制过去osxkvm10146然后挂载这个raw image,安装到虚拟出的boot区得到相关文件。，不要在host osx中安装（有保护也装不上）也不要在guest osx本机中安装。如果你想替换clover.iso，直接在host osx端挂载raw image编辑boot区放入新调的clover.iso(修改完成不需要卸载，pd端就能反应出变化)，用不着一台具有网络能力的qemu tcpe了。

新的libvirt xml和boot.plist
-----

我们为什么deprecate前文qemu4.2的思路，前面的增删指令集思路没用，因为一个CPU涉及到架构，不是简单的指令集的叠加。因此我们从别的方向前进。我们用上了libvirt xml和vnc，我们使用libvirt是看中了它能自定义cpu，这样我们可以坚守在2.1下，一步一步自定义cpu成skylake或Cascadelake:

```
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>osxkvm</name>

  <os>
    <type arch='x86_64' machine='pc-i440fx-2.1’>hvm</type>
  </os>

  <features>
    <acpi/>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>

  <cpu mode='custom' match='exact' check='none'>
    <model fallback='allow'>Penryn</model>
  </cpu>

  <memory unit='KiB'>4194304</memory>

  <devices>

   <emulator>/usr/bin/qemu-system-x86_64</emulator>

   <input type='keyboard' bus='usb'/>
   <input type='mouse' bus='usb'/>

   <video>
      <model type='cirrus'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>

    <graphics type='vnc'/>

    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' cache='writeback'/>
      <source file='/media/psf/DATA/aliyunosx/osxkvm10146'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
 
    <interface type='bridge'>
      <mac address='52:54:00:c9:18:27'/>
      <source bridge='virbr0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x0' slot='0x03' function='0x0'/>
    </interface>
 
  </devices>

</domain>
```

网络还是用回文章1的tap0而不是使用helper，因为我们想免去boot.sh

```
sudo ip tuntap add dev tap0 mode tap 
sudo ip link set tap0 up promisc on 
sudo brctl addif virbr0 tap0（这句跟xml中bridge chunk中的target作用重复,如果不删掉<target dev='tap0'/>你每次都需要sudo brctl delif virbr0 tap0,PD一旦休眠或暂停，tap0会自动消失因此这个绑定作用失效需要重新执行然后把virbr0停止再开一下，所以测试过程中要把pd中deepin设为永不休眠，否则qemu tcpe网络不稳定)
```

继续作源码修改强化变色龙的调试功能：

因为10.14开始已经不支持32 sdk了。所以最新可用的10.13，PD直接application下的安装app直接做成镜像，过程中要产生一个cdr，这个原理就是我们在文4中提到的，其实可以手动直接createinstallmedia产生供pd使用的可安装镜像,只是在10.15不能执行32位createinstallmedia，制作好的安装程序已损坏，暂时更改主系统的time，例如date 102516242016回车,并禁网络直到安装完成(guest中这样做比较麻烦第一阶段可以过去，但第二阶段选不到菜单调不出命令行，过不去)

10.13支持的是Command Line Tools (macOS 10.13) for Xcode 9.2.dmg，它也没有10.11 command tools下不支持c99 vsscanf等函数发生“ld vsscanf : symbol(s) not found for architecture i386”等的问题:

graphics.c中Bouncing ball .oOo.把动画图标换成文字小棍。prompt.c中加个by 自己的标识
前面说到变色龙的bdmsg就是信息内容复杂点的-v + pause()，它其实是#define DBG(x...)	msglog(x)，可以改动把它保存在可访问的fat32 boot分区内。

sys.c long GetFileInfo，可以得到文件名，用于hfs.c中提取更简单的文件名，你也可以在hfs.c中用vsscanf正规提取*.kext的部分，-v最后出现的加载drive，只要在启动时命令行中同时指定UseKernelCache=No才出现，变色龙默认使用prelinkedkernel，也就是usekernelcache=yes(-f=Yes)，此时不加载s/l/e，一定要指定usekernelcache=no才加载

在qemu src的pcbios下，我们找到acpi-dsdt.aml(i440fx),q35-acpi-dsdt.aml，变色龙也支持DSDT，因为变色龙逻辑中只写了bt(0,0)而我们的dsdt.aml在en(0,0)中，所以丢到extras还不够，还要在boot.plist中写一条key DDST,string dsdt.aml

其实变色龙也支持和clover一样的自定义kernel patch。在kernel.plist中写<key>KernelPatches</key><array><dict>,blaaa..


二个boot，调试cpu尝试让mojave在Skylake-Server下运行解决黑屏问题
-----

Clover和变色龙下，cpu设为Penryn可进入我们前面做好的镜像。但换cpu就进不去。这也在意料之中，cpu不兼容。继续看一段关于处理CPU兼容的问题，你可以三者全实现也可以保持不冲突部分实现，只要能最终启动mojave：

Why Penryn is recommanded beforehttps://forums.unraid.net/topic/84430-hackintosh-tips-to-make-a-bare-metal-macos/:

After digging a lot of code, I have some conclusions why they recommanded Penryn as prefered CPU Model:
1. Penryn is classic, and it missing some features compared to newer generation, which lead to a similar situation with VM.
    1. Penryn do not have a msr 0x00000198 leaf to read the perfstatus (like bus ratio, cpu frequency) which the same as a VM.
    2. Penryn do not have a msr 0x35 leaf to read topology structure, which also most hypervisors haven't implemented yet.Instead, the MacOS will try to get the topology from acpi when it detect a Penryn process, which the same as a VM.
    3. Some bootloaders do have some compatibility issues when using a newer generation in a VM, some causing dividing by zero errors(They can't get correct frequency from acpi or msr, so they may be zero).
2. Some articles have outdated so long from now.
    1. It don't have some cpuid features like avx/avx2/bmi/fma so MacOS won't recognized those features even thought you just passed through.

osx下的cat /proc/cpuinfo：sysctl machdep.cpu，sysctl -a | grep machdep.cpu，sysctl -a | grep hw.optional

更多在clover和变色龙下patch cpu的选项和功能：https://github.com/AMD-OSX/AMD_Vanilla/blob/master/17h/patches.plist：

```
<string>algrey - commpage_populate -remove rdmsr</string>
<string>algrey - cpu_topology_sort -disable _x86_validate_topology (10.14.4+)</string>
<string>algrey - cpuid_set_generic_info - disable check to allow leaf7</string>
<string>algrey - cpuid_set_cpufamily - force CPUFAMILY_INTEL_PENRYN</string>
<string>xlnc - cpuid_set_cpufamily - force CPUFAMILY_INTEL_PENRYN</string>
<string>algrey - i386_init - remove rdmsr (x3)</string>
<string>algrey - tsc_init - remove Penryn check to execute default case</string>
<string>algrey - tsc_init - grab DID and VID from MSR</string>
<string>xlnc - tsc_init - grab DID and VID from MSR</string>
<string>algrey - tsc_init - skip test and go get FSBFrequency</string>
<string>xlnc - tsc_init - skip test and go get FSBFrequency</string>
<string>algrey - lapic_init - remove version check and panic</string>
<string>algrey - lapic_interrupt - skip checks and prevent panic</string>
<string>xlnc - lapic_interrupt - skip checks and prevent panic</string>
<string>algrey - mtrr_update_action - fix PAT (x2) (10.14.4+)</string>
```

很不幸，这篇文章依然不是《最终实现在云机上运行黑果》，依然是前文的继续，我们的目的只是把osx装上云当个nas，代替《聪明的Mac osx本地云》文使之成为iphone,osx的nas mate os，可能这种尝试有点付出太多了。但无论如何，读到这里的小伙伴都赢了：目标已经十分接近了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338317/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



