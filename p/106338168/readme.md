云主机装黑果实践（1）：deepin上qemu+kvm装黑果
=====

__本文关键字：用ubi在osx上打造一分区一安装启动U盘，deepin上qemu+kvm装黑果__

在上文《云主机装黑果》中，我们讨论了其可能性，或许这里面的坑远远不止那文提到的，甚至最终不能成功。—— 但不管如何，实践总要先行。在理论与实践组成的一对学习矛盾中，先见独到的理论思考，与慢悠盲目的实践，都必不可少。现在让我们开始正式的实践。

我们在前面说到，现在最新的流行OS，如windows,ubt发行，osx，都是基于uefi机考虑的。同一个系统，安装在同一台pc的mbr或uefi模式下，表现结果和所需驱动都是不一样的，比如你会发现有些驱动不能工作而有些可以，有些都有驱动的表现也不尽相同，这是因为bios和uefi定义了同一硬件的不同规范。之所以再次提到这个，是因为继续云机装黑果前，我们需要制作一套实践环境。需要在PC上制作一套省事的，统一的osx,windows,linux多系统安装和启动法 ——— 这些同样与uefi相关：

我们知道，uefi提供了将固件放在磁盘上的方式（与OS同一处理方式），这带来的好处就是，所有的启动，不必再mbr式写启动动到磁盘结构了，也不必动到整盘，大家只需要在gpt的efi分区中写入uefi文件即可安装，也可运行，而且这对U盘硬盘，（写入安装）和（启动系统）都是一样的。在各OS运用不同磁盘格式的现实情况下，这也许是是在PC上u盘安装启动多系统的最省事方法。——  但是历史上，有些os,如Windows uefi的安装程序不安分（会创造额外分区，msr,等等），osx uefi和deepin uefi之类的os倒是十分安分,苹果是支持installmedia 刻录到分区安装和启动的（你可能不知道，osx最初的标语是打造一个人性化的os，从note，日历，到icloud，个人认为日常都离不开这些，都是很人性的。所以就连它的createinstallmedia安装盘方式和本地盘方式也是基于分区的。），deepin也可以分区安装。这二者共存就是我们想要的结果：你会在一个装有多OS的GPT盘的efi分区发现多个系统的efi文件夹。至于windows，你可以pass使用它的安装程序，用winpe代替，集成wim安装包，pe里面会有一个nt6setup工具选择efi分区和系统分区的选项，从而达成我们要的结果。

1）用ubi在osx上打造一分区一安装启动U盘，一分区一系统的本地盘格局
-----

下面介绍在osx下运用unetbootin677来制作这样的安装U盘的的方法：Unetbootin的特别之处在于它将安装镜像写入分区而不是整盘，有别于市面其它产品，原理就是syslinux+uefi+fat32，使用时选择镜像后对应分区要事先格式为fat32。但fat32的缺点就是一旦文件有超过4g。就不能使用ubi，但是dvd也有容量限制呢，所以这只是一种折中。

在osx的图形化磁盘工具中，我们将一个32gU盘分成3个10g(或osx 10g,deepin15 4g,windows 10 2019 ltsc 6g，10G作exfat data区放点其它东西 )，每一个分区写入一个安装iso，efi分区是自然生成的，，Ubi是不支持windows的仅支持linux，但是实测也可以按同样对linux的方式强行将windows pe带安装包wim iso刷入，至于安装到的本地多系统，我们也将本地盘分为三到四个区（osx留大一点，最后一个区可以是exfat data放点东西，我是256g,80g给osx,2x40分别给deepin15和windows,90g做data），这样，我们可以把OS X主系统下图形硬盘工具作出的这套结构作为主要的日常视图。日后在finder->位置->下看到清爽的磁盘结构结果,也方便我们接下来的工作（我们需要在这些系统间切来切去而不能使用pd虚拟机来完成以下测试）。

注意，只有osx系统才有开机按alt(option)显示U盘各分区的启动。也许未来在没有这种支持的电脑上也可实现，比如我们可以做个智能U盘，当某系统所属的分区需要被当前使用到，就设置一个把这个分区推出来的功能。来代替平果机的这种功能，而不用使用mbr统一菜单在分区间chain来chain去的那种。（除非某天所有系统都能安装到同一个分区使用同一种格式）

好了，下面继续我们的重点工作，在deepin15中安装qemu+kvm测试安装黑果。

2)deepin上qemu+kvm装黑果：准备工作
-----

由于我的macbookpro是一台2015，cpu是支持虚拟化特性的，所以直接：

Sudo apt-get update
Sudo apt-get install virt-manager bridge-utils libvirt-clients qemu qemu-kvm

发现我的deepin-15.11-amd64在uefi下屏幕开机高亮的问题得到了解决，但是无线网卡没用，弄了个360usb wifi插好对付着用，发现下载到的qemu是2.81的。它的中心是一个libvirt加qemu，加一些类似管理机控制台的命令行和GUI工具，Qemu就是类似osx上bvyve一样的东西。后者也可以直接运行kernel。打开控制台，它里面有一系列虚拟的存储池，网络沲（配置文件和硬盘文件都在/var/lib/libvirt/image，/var/lib/libvirt/network/default.xml，/etc/libvirt/qemu/networks/default.xml)），你可以在GUI里新建一般复杂度配置的vm，但gui功能有限，更多更专业的参数需要命令行完成，如今天谈到的黑果。

根据前文《在阿里云上装黑苹果（1）：黑苹果基础 ，在阿里云上安装黑苹果（2）:本地虚拟机方案研究和可行资源参考》，https://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/index_old.html和http://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/ 这二文基本是有关在kvm上装黑果的技术起源。后者都是别人基于这些研究的一些脚本工具集成。这二文中，old那文是直接用apple boot-132->变色龙(停更)->enoch对变色龙的修改，后来那文是正儿八经地统盘修改ovmf(更干净和接近uefi定制)，，再后来，别人就用clover这个更强大免编译的uefi来代替ovmf的编译了作出更简单的osxkvm方案（如https://www.kraxel.org/blog/2017/09/running-macos-as-guest-in-kvm/的kraxel和https://github.com/kholia/OSX-KVM的kholia）。

PS：Boot-132 is a bootloader provided by Apple for loading the XNU kernel，但是它并不用在真正的Mac上。而Chameleon is a Darwin/XNU boot loader based on Apple's boot-132.Enoch对变化龙的增强可以启动早期osx，甚至到10.11,10.12为止的osx，说到底，enoch与clover做的工作是一样的，也都是efi的，(但enoch是对seabios朝uefi的过渡层，与普通bios机和云主机的bios接近，云主机是seabios)，具体到kvm，enoch是kernel方式喂给qemu的,是bios上浅浅的一层。对Qemu/kvm guest方要求的更改最小，vs clover对uefi的要求最低。可能更适合我们未来将其用受限方式集成到受限的kvm环境如利用linuxboot等方式将其喂给普通阿里云ecs，以上是固件上的，其它工作方面，pc上黑平果的工作就像黑群，从loader着手，必要的时候，也重新打包安装包(vs synology upgrade pat)。因为这里要处理固件驱动注入，修改原包中正常启动参数到适合特定硬件启动，kvm guest机器其实可以类比通用pc上黑果技术的一组硬件特例处理,cloud kvm更是仅明确使用virtio驱动。

所以enoch这种，最接近pc黑果技术的起点，所以我们选择从它开始而不是clover

由于kholia在他的仓库中中把somlo的old文中的前期方案的相关脚本和工具也做了进来，所以这里为了方便我干脆采用它的一并说明，网上搜索kvmqemu osx出来的教程多用enoch_rev2839_boot和Install_OS_X_10.11.3_El_Capitan，所以这里也准备这样做。我clone到的2019.10.25的https://github.com/kholia/OSX-KVM/commit/64fcc1ef4d800197e8f4fc1421dd0f7060a72166，在整个脚本的backup文件夹内（backup之外的则对应最新利用clover的那些工具和脚本）。先看这个https://github.com/kholia/OSX-KVM/blob/master/notes.md，Enoch Bootloader (obsolete)是宣称过时的，意料之中，继续，1）脚本第一步，是create_install_iso.sh(for  10.11.0-10.11.4 and 10.12.0-10.12.5,只能在osx bash下运行)，它将从appstore下载到application/install OS X xxx app中的安装程序进行再打包，使之成为qemu适用的iso。如果你下载的OS X离线安装包，可以手动释放到application，这个过程中，一些必要的驱动和变动，也将被加进去，也就是破解kernel的方式。（这也是常见制作懒人包的方式 under pc黑果s），在安装过程中会用到。2）我下载到的是OS_X_10.11.3 El Capitan,运行，会出错，说tar 解压不到正确的格式包，下载安装Pacifist,用它打开install osx ei capitan.app/contents/sharesupport中的InstallESD.dmg，切换到”软件包内容”“资源”的“资源”，把Essentials.pkg释放到backup下，然后修改create_install.iso.sh中的对应处，成为python "$script_dir/parse_pbzx.py" "$script_dir/Payload" | cpio -idmu ./System/Library/Kernels || exit_with_error "Extraction of kernel failed”，重新运行脚本，得到Install_OS_X_10.11.3_El_Capitan.iso —— 与此同时，发现backup中，没有enoch_rev2839_boot，我从网上搜索下载到这个文件，放到backup中，值得注意的是网上下载到宣称2839的不一定就是对应版本.3）再在backup下创建一个40g硬盘，用来安装，qemu-img create -f qcow2 osxkvm.qcow2 40G。

3)deepin上qemu+kvm装黑果：启动
-----

三大件准备完毕，我们需要先执行一些准备工作：

打开kvm参数：echo 1 > /sys/module/kvm/parameters/ignore_msrs，不这样，就会显示boot:不断重复重启。

网卡和网络设置部分：
Kvm会自己创建一个virbr0和br0（见控制台虚拟网卡default栏），为nat（Installing "virt-manager" automagically creates the "virbr0" local private bridge :-) ）。我们把osx网络与主机网络组成tap：
sudo ip tuntap add dev tap0 mode tap 
sudo ip link set tap0 up promisc on 
sudo brctl addif virbr0 tap0

仓库脚本最后一般是用复杂命令行定义的VM和启动逻辑，这也是网上的教程常讲的主体部分（除了命令行，也有libvirt.xml版，但我们讲命令行版方便说明）：

```
qemu-system-x86_64 -enable-kvm -machine pc-q35-2.4 -smp 4,cores=2 \

	  -cpu Penryn,kvm=off,vendor=GenuineIntel \
	  -m 4096 \

	  -smbios type=2 \
	  -kernel ./enoch_rev2902_boot \

	  -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" \

	  -usb -device usb-kbd -device usb-mouse \
	  -device ich9-intel-hda -device hda-duplex \

	  -device ide-drive,bus=ide.2,drive=MacHDD \
	  -drive id=MacHDD,if=none,file=./osxkvm.qcow2 \
	  -device ide-drive,bus=ide.0,drive=MacDVD \
	  -drive id=MacDVD,if=none,snapshot=on,file=./'Install_OS_X_10.11.3_El_Capitan.iso'

	  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -device e1000-82545em,netdev=net0,id=net0,mac=52:54:00:c9:18:27 \

	  -monitor stdio

kvm=off，并没有使用kvm半虚拟化加速。
penryn,pc-q35-2.4，这是qemu虚拟出来尽量按近mac的cpu和主板机型,q35使用osx硬件采用的ich9芯片组。
-smbios type=2 seabios patch,required for booting [Mountain]Lion
我们注意到device isa-applesmc,osk=“”这行，对应somlo old文，这是Mac专属硬件，用程序算出来的
urhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc这是固定的。
修改好boot,qcow2,iso各文件对应位置。
如果是-usb -device usb-kbd -device usb-tablet要改成-usb -device usb-kbd -device usb-mouse，否则moniter stdio下没有鼠标（usb-tablet后期在系统运行时可以有另外的开源驱动用）。
Ich9-intel-hda声卡
注意到虚拟网卡-netdev 开始的部分，这是最大的坑，作为黑果特例，网卡单一e1000是osx普遍内置的，net0是osx内部用的名字。52:54:00是osx合法的开头，见https://github.com/kholia/OSX-KVM/blob/master/networking-qemu-kvm-howto.txt,你还可给osx guest设置更复杂的bridge网络。
（也可以后期装virtio net pkg）
-monitor stdio使用原生kvm窗口。也可以spice或vnc
```

（这是整个osxkvm黑果的技术主体，也是pc黑果的技术主体https://www.insanelymac.com/forum/topic/278638-enoch-bootloader/，之所以以上都能轻描淡写做进命令行，是因为它们被解决了，从中我们看到somlo old文对应所提到的大大小小的坑,还有日渐功能丰富的qemu也会将它们整合包含(somlo文中一些对kvm本身编译更改的在qemu更新版中不必再做了)，故，以上在考虑将osx移到云主机时就需要逐个攻破，从qemu重新转到guest内，让kvm guest自包含而不是外部喂给，或依赖host qemu）

启动脚本boot-macOS.sh，显示虚拟机启动窗口。开头显示r2839字样，ehci warning: guest requested more bytes than allowed，接processing error - resetting ehci HC,但可以继续。如果不是r2839就启动不了。显示osx的recovery安装界面，进去，格式化磁盘，取个名，cp -av /Extra /Volumes/磁盘名（这个时候我们处在iso的文件系统中，extra就是写入的那些扩展,其实也就org.chameleon.boot.plist一个文件Chameleon configuration:timeout,To automatically boot into OS X without needing to press enter at the prompt，The EthernetBuiltIn and PCIRootUID keys fix the “Your device or computer could not be verified. Contact support for assistance.” error when logging in to the App Store.，主要是配置一些启动参数，没有kexts注入），继续安装。

如果在这里碰到No packages were eligible for install，一种解法是网上说的修改时间，如果点选左下角定制的时候，注意到osinstallsciprt为0kb，应试是镜像做错了，再次做好镜像即可。


——

至于加速的和硬件直通的kvm osx。你需要更深入。这样的方式使用黑果是不是违法的，因为硬件就是 apple规约中的那些，apple并没有规约硬件的外层或外延。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338168/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




 
