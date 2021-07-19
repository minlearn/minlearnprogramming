云主机装黑果实践（6）：处理云主机上变色龙启动后置过程：驱动和黑屏
=====

__本文关键字：无显驱vesa方式驱动osx10.14,mojave vga黑屏，云主机的显示器，非n非a卡黑果，waitting for root device，apfs modules stop 1432,appleexclude.kext，can't determine on the same uuid,qemu virtual display,qemu vga glitch，starting darwin x86就黑屏, osx cascadelake 黑屏, 变色龙 skylake 黑屏__

在前面5篇系列文章中，我们讲到了云上装黑果的基础工作和碰到的前置调试问题，我们采取的是用本地尽可能接近阿里云参数的情况下模拟镜像在阿里云端可安装运行（安装镜像/安装，安装后镜像/运行）的测试路径，假设这些工作完成之后，就可能肯定基本离成功在阿里云轻量上运行黑果这个目标很接近了，文1到文5中间做的都是顺利的准备工作，除了准备镜像都是前置启动故障排解，文章5末尾我们描述的一些1,驱动调试相关的问题和2,虚拟机不黑/轻量云黑屏的特别情况，这些都属于变色龙启动后darwin开始接手的地方，属于后置过程了，进入到机型适配了，被普遍认为是机型适配碰到的BOSS级的问题，本文开始，继续文5提到的问题排解和调试，其中有一条是在i440fx下我们无法从virtio blk驱动，因此也无法从其中的系统启动，启动中出现waitting for root device（我们初步判断这种问题有可能是前文说的pci功能位不对，也可能往往是驱动加载不正确，启动不起来，不要动不动就联系到AppleAHCIPort）。

这是因为文章2-4适配的是安装后镜像，是正确的路径，但到了5我们企图得到一个安装镜像，偏离我们原来的测试路径，其实，安装与运行，这二者不能二全，installer镜像是不含任何virtio和virtualgraphics驱动的，安装后的镜像才有，只要我们在qemu配置文章中手动加入功能位，而且使用没有受破坏的安装后的镜像，即，在变色龙启动完成后的kernel verbose界面保证不出现显示virtio驱动加载错误（如果你强行把virtio,virtgraphics相关和依赖kext丢进E/E或S/L/E，这往往又破坏镜像加的权限和缓存，照样会出现apfs modules stop 1432，修复往往不起作用），就可以实现识别virtio blk驱动，从其中启动的，而cirrus-vga显卡驱动，在启动时有花屏，进入系统后却是可以识别并驱动的（只是是800x600模式的SVGA兼容模式，此模式由VESA为IBM兼容机推出的标准。分辨率为800x600，查看设备为00:02.0 "Class 0300" "1013" "00b8" "1af4" "1100" ，1013  Cirrus Logic，00b8  GD 5446，1af4 1100  QEMU Virtual Machine，这说明virtualgraphic也发挥作用了）

驱动问题并没有发展到需要patch kext处理显卡驱动，更没有涉及到加起QE/CI显卡加速来，因为我们是云主机无须显卡加速，但黑屏依然是一个暂时无解的问题，可能是CPU指令集问题，也可能是黑屏与修改edid法修复显示器参数,dsdt法，但这些都可以作为猜想，都需要慢慢找出问题。

重整理的测试参数
-----

我们重新设计一下测试参数：

创建安装镜像cdr:

```
hdiutil create -o "osxkvminstall10146" -size 8g -layout SPUD -fs HFS+J
hdiutil attach "osxkvminstall10146.dmg" -noverify -nomount
diskutil partitionDisk disk2 MBR JHFS+ OSXKVMINSTALL R
直接用mbrpatch脚本免Q5Q6，不要手动复制basesystemdmg/s/l/e或pacifist解压，否则有权限问题，会导致apfs module stop
及时安装变色龙启动,这个dmg和cdr卸载后会变成只读，所以尽早安装变色龙,
hdiutil convert "osxkvminstall10146.dmg" -format UDTO -o "osxkvminstall10146"
```

创建安装后镜像：

```
dd if=/dev/zero of=osxkvm10146 bs=512 count=52428800
hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount osxkvm10146
diskutil partitionDisk disk2 MBR fat32 BOOT 100M JHFS+ OSXKVM R
（mbr的在安装后的osx不可动态调整因此没有必要在尾端保留一个osxinstaller，做成osxkvm,osxkvminstall镜像为一体，用尽25G）
fdisk -e /dev/rdisk3,flag 1,write,yes,quit。
在linux中安装grub2,syslinux,tcpe,wowpc
```

用这套安装,按esc进,用cdr上的启动就可以避免安装时osxkvm所在卷不可卸载:

```
qemu-system-x86_64 -enable-kvm \
 -machine pc-q35-2.8 \
 -cpu Penryn,kvm=off,vendor=GenuineIntel \
 -m 5120 \
 -usb -device usb-kbd -device usb-mouse \
 -device ide-drive,bus=ide.2,drive=MacHDD \
 -drive id=MacHDD,if=none,format=raw,file=./osxkvm10146 \
 -device ide-drive,bus=ide.0,drive=MacDVD \
 -drive id=MacDVD,if=none,snapshot=on,file=./osxkvminstall10146.cdr \
 -boot order=dc,menu=on
```

用这套启动安装后的系统，注意去掉了smp，且其中新手动加的pci插槽和功能位,tcpe下lspci就能看到:

```
qemu-system-x86_64 -enable-kvm \
 -machine pc-i440fx-2.8 \
 -cpu Penryn,kvm=off,vendor=GenuineIntel \
 -m 5120 \
 -device cirrus-vga,bus=pci.0,addr=0x2 \
 -usb -device usb-kbd -device usb-mouse \
 -device virtio-blk-pci,bus=pci.0,addr=0x5,drive=MacHDD \
 -drive id=MacHDD,if=none,cache=writeback,format=raw,file=./osxkvm10146 \
 -device virtio-net-pci,bus=pci.0,addr=0x3,mac='52:54:00:c9:18:27',netdev=MacNET \
 -netdev bridge,id=MacNET,br=virbr0,"helper=/usr/lib/qemu/qemu-bridge-helper"

chameleon.boot.plist中删掉：
<key>UseKernelCache</key>
<string>Yes</string>
```

启动的时候一定要手动打上UseKernelCache=Yes，否则还是识别不了virtio,-v 可以看到识别了virtio blk，在Rebuild caches after update，early boot complete,continue后会出现：notice - new kext com.apple.driver.kextexcludelist, v14.0.3 matches prelinked kext but can't determine if executables are the same (no uuids)，等漫长的timeout之后(我等了五分钟)。可进界面（又是漫长的左上角滚动海浪球），下次启动不会等这么久。

完成！可顺利加载virtio和显卡驱动进入系统，封装osxkvm10146为gz，上传到云主机。你可以把那个第一次进入等待过久的过程固化在镜像中打包，云机中以后就不用等这么久了。


至于如何让Applevirtio手动变成一个boot time driver集成在install镜像或osxkvm镜像中免去使用这个prelinked kernel(一般情况下避免使用prelinked cache,-f是用来强制重新从磁盘s/l/e加载kext跳过缓存的。但并不会重建prelinked缓存，注意把kext与kernel的缓存分开)，我尝试把kext提到变色龙或s/l/e,权限修复工具，手动修复，kextcache 实现的几种方法都无效，KCPM Utility Pro，kext utility 2.6.1这类工具无法给离线系统注入kext，更提示无法给恢复系统注入kext和make cache。都没有成功。

对cache机制的理解：

The kernel is the first component of the operating system to start. It has no other tools available. In particular there is no way to check code signatures, and all file system access is very hard at this point. Apple therefore decided to prelink the bare kernel with all kernel extensions every time the kernel or one of the extensions is updated, and to start only that prelinked kernel at boot time.

Since the prelinked kernel is on a read-only volume, it cannot be updated directly. Apple had to conceive a new mechanism for updates.

When you reboot or shut down your machine, launchd stops all processes. Then it remounts the system volume in read/write mode. This is possible because launchd has the entitlement com.apple.private.apfs.mount-root-writeable-at-shutdown. Then it runs /var/install/shove_kernels to copy the new kernel.
But apparently this doesn’t actually work. So to update a kernel extension you need to disable System Integrity Protection or manually trigger a kernel update after booting into macOS Recovery.

Note: This is fixed in macOS 10.15.1

我用的脚本：

```
#!/bin/sh
sudo chmod -Rf 755 /S*/L*/E*
sudo chown -Rf 0:0 /S*/L*/E*
sudo chmod -Rf 755 /L*/E*
sudo chown -Rf 0:0 /L*/E*
sudo rm -Rf /S*/L*/PrelinkedKernels/*
sudo rm -Rf /S*/L*/Caches/com.apple.kext.caches/*
sudo touch -f /S*/L*/E*
sudo touch -f /L*/E*
sudo kextcache -Boot -U /
```


驱动解决，也不用考虑安装镜像和安装问题，接下来只需探索黑屏问题。

云主机变色龙黑屏探索1：cpu兼容
-----

我们在上文说到，在云轻量上，变色龙读取s/l/e，加载完plist和kext中的可执行文件，直到TMsafetyNet.kext ，从这里开始就黑屏了(控制台可以看到磁盘和CPU都没有读写，应该是halt了。而不仅是虚拟vnc显示器没有信号)，我们上文也讲到Wait=Yes不起作用，可能也是在执行wait=yes前就黑了，因此我们有必要修改变色龙源码，在结束-v所有输出98%内容后输出starting darwin kernel前（或许是判断文件名tmsafetynet后），每条输出都pause一下，看到底在哪里黑屏了。

上面的qemu启动配置，就cpu和display没有真正做到与云轻量对应。如果没有cpu直通，kvm虚拟出来的是vcpu，客户机看到的是基于 KVM vCPU 的 CPU 核，而 vCPU 作为 QEMU 线程被 Linux 作为普通的线程/轻量级进程调度到物理的 CPU 核上。

进轻量云，tcpe cat /proc/cpuinfo，对于cpu，我们发现CPU是变化的，有时是Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz（Cascadelake），有时却是：Xeon Platinum 8163（SkyLake），这是一个1C的主机：processor: 0，physical id: 0，siblings: 1，core id: 0，cpu cores	: 1，也可以看到它的feature指令集。

我们cpu设为最基本的客户机 CPU 模型qemu64,把所有的cpu flags都加上。后面加check,enforce来查看与host的比较情况，最后上面的cpu变成：

```
-cpu qemu64,kvm=off,vendor=GenuineIntel,+fpu,+vme,+de,+pse,+tsc,+msr,+pae,+mce,+cx8,+apic,+sep,+mtrr,+pge,+mca,+cmov,+pat,+pse36,+clflush,+mmx,+fxsr,+sse,+sse2,+ss,+ht,+syscall,+nx,+pdpe1gb,+rdtscp,+lm,+constant_tsc,+rep_good,+nopl,+pni,+pclmulqdq,+ssse3,+fma,+cx16,+pcid,+sse4_1,+sse4_2,+x2apic,+movbe,+popcnt,+tsc_deadline_timer,+aes,+xsave,+avx,+f16c,+rdrand,+hypervisor,+lahf_lm,+abm,+3dnowprefetch,+fsgsbase,+tsc_adjust,+bmi1,+hle,+avx2,+smep,+bmi2,+erms,+invpcid,+rtm,+mpx,+avx512f,+avx512dq,+rdseed,+adx,+smap,+avx512cd,+avx512bw,+avx512vl,+xsaveopt,+xsavec,+xgetbv1,+xsaves,+arat,check,enforce
```

放在本地启动，发现2.8的qemu-kvm -cpu help，Qemu Recognized CPUID flags，是不支持以下的：
qemu-system-x86_64: Property '.constant_tsc' not foundqemu-system-x86_64: Property '.rep_good' not foundqemu-system-x86_64: Property '.nopl' not foundqemu-system-x86_64: Property '.tsc_deadline_timer' not found

```
而且，我的主机cpu是不支持以下的（这并不影响什么，只要qemu支持模拟就对我们的实验结果没有影响）：
warning: host doesn't support requested feature: CPUID.01H:EDX.ht [bit 28]warning: host doesn't support requested feature: CPUID.07H:EBX.hle [bit 4]warning: host doesn't support requested feature: CPUID.07H:EBX.erms [bit 9]warning: host doesn't support requested feature: CPUID.07H:EBX.rtm [bit 11]warning: host doesn't support requested feature: CPUID.07H:EBX.mpx [bit 14]warning: host doesn't support requested feature: CPUID.07H:EBX.avx512f [bit 16]warning: host doesn't support requested feature: CPUID.07H:EBX.avx512dq [bit 17]warning: host doesn't support requested feature: CPUID.07H:EBX.avx512cd [bit 28]warning: host doesn't support requested feature: CPUID.07H:EBX.avx512bw [bit 30]warning: host doesn't support requested feature: CPUID.07H:EBX.avx512vl [bit 31]warning: host doesn't support requested feature: CPUID.80000001H:EDX.pdpe1gb [bit 26]warning: host doesn't support requested feature: CPUID.0DH:EAX.xsavec [bit 1]warning: host doesn't support requested feature: CPUID.0DH:EAX.xgetbv1 [bit 2]warning: host doesn't support requested feature: CPUID.0DH:EAX.xsaves [bit 3]warning: host doesn't support requested feature: CPUID.0DH:EAX [bit 3]warning: host doesn't support requested feature: CPUID.0DH:EAX [bit 4]warning: host doesn't support requested feature: CPUID.0DH:EAX [bit 5]warning: host doesn't support requested feature: CPUID.0DH:EAX [bit 6]warning: host doesn't support requested feature: CPUID.0DH:EAX [bit 7]
```

暂时先在上面的cpu指令集中删掉以上4条。正常启动Qemu，果然，这个cpu下发生了黑屏，跟云机表现一致,重新在deepin15编译qemu高版本，QEMU 3.1.0 in added the Cascadelake-Server CPU,我们下载的4.2.0：

sudo apt install libpixman-1-dev bison flex libsdl2-dev(不加这个会卡在vnc server)
./configure —prefix=/usr —target-list=x86_64-softmmu —enable-sdl
make install

cat /usr/share/libvirt/cpu_map.xml |grep Cas -A100，发现有Cascadelake-Server，但是我们不把cpu指定为cascade lake-Server，我们更相信指定具体的指令集，删掉的4条可以加上了，重新运行qemu，可以运行，但依然会出现host cpu缺失指令集warning，虚拟机表现同样黑屏，跟云机一致。

看来，这个CPU太先进了，只能尝试CPUID fix了（如https://www.insanelymac.com/forum/topic/335650-kernelandkextpatches-1013x1014x1015x-x99x299/），就跟处理前面的msrs一样，变色龙集成了一部分cpu patch，但需要我们做更多工作。不能确定是哪个指令集缺失或什么其它原因导致的问题，多开几台不同CPU的云主机试试，或者在本地不断调整指令集参数作Clear Test测试，然后在相关kext处作patching针对解决。


云主机变色龙黑屏探索2：虚拟显示器注入
-----

在探索1提到，驱动和显示器问题可能并不是黑屏的原因(nv_diable让vesa生效没用,radvesa没用，删s/l/e驱动让Vesa生效也没用，一般不必也导致权限问题，resolustion915 fix也不是)。安装过程中花屏和vnc中glitch现象（https://ostanin.org/2014/02/11/playing-with-mac-os-x-on-kvm/）是24位与36位混乱形成的，但不影响进入系统(想到这里，突然也记起以前用向日控控的时候，笔记本有屏幕，需要拔掉，才能那个网络界面中显示,控控对台机好用自配屏的笔记本不行)。但为了完善我们的测试过程，我们还是考虑可能的显示器edid问题：

The EDID (Extended display identification data) data structure have all the info of your graphic card and other video sources.
EDID是由VESA——视频电子标准协会定义的，并在1994年和DDC标准1.0版一起推出了1.0版本，qemu也支持以下虚拟显示器：

-display sdl - Display video output via SDL (usually in a separate graphics window).
-display curses - Displays video output via curses.
-display none - Do not display video output. This option is different than the -nographic option
- display vnc, 这就是我们云机支持的，

再看几条爬贴参考：

https://www.tonymacx86.com/threads/black-screen-after-boot-menu.165117/

https://www.tonymacx86.com/threads/guide-general-framebuffer-patching-guide-hdmi-black-screen-problem.269149/

新版本的qemu支持edid，但是云主机的qemu是我们无法控制的。考虑这条方向上的尝试不实用。总之探索2最好在探索1完成后再进行。

—————

也许我们以后要深度修改变色龙源码，把usekernelcache固化进变色龙，或加入云专用的kexts，定制一下whatevergreen之类的东西与云主机适配，修改resolustion src为cirrus logic vga所用。尽量避免动到镜像本身。还有那个msr ingore问题，希望在变色龙端彻底解决：msr 0x35 which qemu/kvm not implemented yet then ..mojave和catalina都越来越大了，未来精简osx为更小的镜像比如至仅命令行。据说还有mojave，AppleQEMUHID.kext…


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338301/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



