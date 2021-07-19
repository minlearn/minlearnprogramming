云主机装黑果实践（4）：阿里轻量机上变色龙bootloader启动问题
=====


__本文关键字：处理云主机上大镜像安装问题，编译 enoch 变色龙__

在《云机装黑果实践系列》1-3中，我们完成了直到生成镜像的所有准备工作，现在要上机测试了，进入最困难的boot-机型适配调试了，这也是黑果技术最典型的实践密集点，结合搜索引擎从最最小依赖一点一点添加配置是唯一的流程（基本上，变色龙是appleboot+fake efi as bios发展来的，具体机型千千W，云主机又特点，这种适配和调试工作变数和坑很多），我选的是一台阿里云轻量云主机，第一步是把镜像传上去。对于一个7G展开20G的打包镜像，moeclub的installnet.sh其内部使用的是wget gunzip输出到stdout再dd的管道，gzip版本太低，解压到前面很少一部分就会hang，脚本自动重启，找到那句将其改成wget -qO- '$DDURL' |tar zOx |/bin/dd of=\$(list-devices disk |head -n1);Tar 不要加f和其它参数,版本不够。正常边untar边dd在我这（港区oss与轻量）要50来分钟，镜像正常启动grub2,进入tinycorelinux virtiope，fdisk /dev/vda，p显出正常hfs+分区，或者直接grub2 insmod hfs hfsplus part_apple，ls (hd0,msdos2)/ 也可以看到osx分区上的文件。之后迅速做一个快照，以防接下来的调试破坏系统。

如果说上面解决installnet.sh的脚本问题是小问题，那么大boss来了，r2922的cdboot在实机和kvmqemu机上可以运行，在云主机上根本无法运行（grub2进进入tinycorelinux virtiope，sudo mount /dev/vda1,sudo wget生成的iso，重启进入），不能显示引导界面，也是自动重启。(除了cdboot,按教程直接boot0h,hfs启动hfs分区的硬盘系统不行，用mbr+boot1f32也不行。)

问题可能在哪？这就是前面提到的ignore_msrs,我从memdisk版本问题开始排除，最后聚焦在boot本身上（cdboot这个stage2基本是一个boot+2560kib），猜想可能是版本问题导致的，利用排除法，先在网上海找了一通，能找到的最流行的低版本，就是v5.0.132 enoch r2839，，其cdboot写iso可以启动主机。最大r2841也可以，2842开始就不行。

翻看http://forge.voodooprojects.org/p/chameleon/source/changes/2842/，发现经过了三个commit，重点是源码上的变化:

Commit 2842, by ErmaC : General update
Commit 2841, by Bungo : 1) Dropping DMAR (DMA Remapping table) to use stock AppleACPIplatform.kext - resolves stuck on "waitForSystemMapper" or "[PCI configuration begin]" 2) Added "ACPI" (all capitals) path 3) Small cosmetics
Commit 2840, by Bungo : Sync

剩下就是实际编译出cdboot判断问题到底出在哪个commit了

从enoch的源码中找出变化，实际编译
-----

编译环境是pd上的xcode 8.2.1 for EL CAPTAN 10.11，这套配置可以编译2841->2922，其它的都会有问题。苹果的开发链都很封闭自由度不高。只能选择这个配置了。

下载一个10.11，把它作成pd能安装用的镜像，原理跟mbrpatch打包类似，适合在PD安装老版本osx使用。

```
(以下差不多任意版本都适用)
hdiutil attach /Applications/Install\ macOS\ Sierra.app/Contents/SharedSupport/InstallESD.dmg -noverify -nobrowse -mountpoint /Volumes/install_app

hdiutil create -o /tmp/Sierra.cdr -size 7316m -layout SPUD -fs HFS+J
hdiutil attach /tmp/Sierra.cdr.dmg -noverify -nobrowse -mountpoint /Volumes/install_build

asr restore -source /Volumes/install_app/BaseSystem.dmg -target /Volumes/install_build -noprompt -noverify -erase

rm /Volumes/OS\ X\ Base\ System/System/Installation/Packages
cp -rp /Volumes/install_app/Packages /Volumes/OS\ X\ Base\ System/System/Installation/
cp -rp /Volumes/install_app/BaseSystem.chunklist /Volumes/OS\ X\ Base\ System/BaseSystem.chunklist
cp -rp /Volumes/install_app/BaseSystem.dmg /Volumes/OS\ X\ Base\ System/BaseSystem.dmg

hdiutil detach /Volumes/install_app
hdiutil detach /Volumes/OS\ X\ Base\ System/

hdiutil convert /tmp/Sierra.cdr.dmg -format UDTO -o /tmp/Sierra.iso
```

再https://developer.apple.com/download/more/下载Command_Line_Tools_macOS_10.11_for_Xcode_8.2.dmg装上。

最后下载源码：svn co -r 2841 http://forge.voodooprojects.org/svn/chameleon/trunk/，svn co -r 2842 http://forge.voodooprojects.org/svn/chameleon/trunk/，svn co -r 2922 http://forge.voodooprojects.org/svn/chameleon/trunk/

它们的编译都是cd trunk，然后直接make，经过几次尝试，从最初直接替换libsaio/cpu.c,cpu.h,platform.c,platform.h，到最后仅在2842 src中libsaio/cpu.c找到以下二句并注释掉

```
//	case CPUID_MODEL_SKYLAKE_AVX://	case CPUID_MODEL_CANNONLAKE:

Re make clean
Re make
```

其产出的cdboot都是可以用在云主机上作正常启动的。这也可以被用在2922上。

再一步步调试出能启动云主机的变色龙配置：
-----

到现在为止，终于进入对类似现实机器调试变色龙的流程来处理针对云主机安变色龙的问题了，这就是在上述一次次的“修改参数+打包iso+tinycorelinux上传”的循环（这也是我们当初使用grub2+memdisk的基本考虑）重复调试参数了：云主机较特别，可能会因为一个问题无法最终成功，但至少希望就在眼前。因为我们解决了大量关键问题。

注：碰到上传了正确的cdboot打包的iso，也启不动云主机到界面的问题，但有一个workarouds：2841和2922的wowpc.iso都上传，发现2922不能启动到界面，先启次一次2841，之后2922总可以成功。猜是loader改变了相关mbr参数，是残留的硬件作用。启动一次2841可以将其复位。（或许整个替换cpu和platform编译会根本解决）

最小配置是这样的：

```
org.chamalon.boot.plist

	<key>Kernel Flags</key>
	<string>-v -f</string>
        <key>Timeout</key>
        <string>30</string>
```
下文继续详解。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338248/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




