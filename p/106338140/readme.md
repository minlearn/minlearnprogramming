在阿里云上安装黑苹果（2）:本地虚拟机方案研究和可行资源参考
=====

__本文关键字：虚拟机黑苹果，osx kvm guest__

在《在阿里云上安装苹果(1)》中我们讲到实机上安装黑苹果是一件难事，但也不是不可能，因为有全套的clover方案和不断丰富的kexts（kexts是EFI专属驱动,不是OS用的），如果硬件有问题，比如CPU不支持osx特有的指令扩展（低标准CPU不能运行OSX,To run MacOS Mojave, you need a processor with support for sse4.1 and sse4.2 instructions），主板不支持UEFI，显卡不受osx支持。我们换了它们就是。而云主机，比如阿里云，（1）我们除了不能控制它的硬件类型如CPU等，（2）还由于它是一种Qemu/kvm/virtio架构。是一种远端host架构，我们没有办法控制host，充其量只能在本地支持VT的实机上自建Qemu/kvm host/guest的虚拟机环境里捣鼓测试，(3)虽然本地都是自己可以控制的全套host/guest虚拟机方案，但虚拟机本身也有它特殊的局限，它不同于实机，是没有UEFI的（4）最后，还由于云主机是虚拟机的一种局限类型和特殊类型，我们除了得不到它的UEFI控制只能得到其guest之外，还要面对其仅支持从mbr+bios中启动系统（5）不论是实体虚拟机，还是云guest only虚拟机，我们都面对需集成virtio的事实。4,5关联紧密。因为UEFI中的驱动跟OS中的驱动联系紧密。 

故在云主机上安装黑苹果就更是一件难事了，每一个新出现的大问题。都有可能使我们的最终目的泡汤。—— 但好在市面上支持osx vm的现行虚拟化方案(使用clover或不使用clover方案的黑苹果，甚至白苹果）也不少，如un-locker+vmware也可以，parallesdesk也可以是最好用的（它是白苹果），qemu/kvm下面的osx guest方案当然就更可以了。最接近我们这里要讲的阿里云的要算qemu/kvm/virtio了 ，在云主机上安osx的方案似乎并没有。但—— 但我们的努力总不致于毫无希望。

解决虚拟机上黑苹果的安装条件:软件/硬件/固件测试环境准备
-----

对于(1)，我们可以换配置的方式来解决，如果不能就换服务商。这个略过。我们仅首先考虑本地虚拟机的情况（2）我们差不多不可能找到一家提供nested kvm和再虚拟化的云主机，除非裸金属，不过其投入很高。同2略过（3）虚拟机上有OVMF这样的东西代替UEFI，OVMF "is a project to enable UEFI support for Virtual Machines”(它依赖edk2的跨平台uefi方案)。不过OVMF只能用在HOST端，做成一文件作为参数传递给虚拟机启动程序，（4）如果说1,2,3都是在实体虚拟机上的解决之法，那么对于4，云主机，我们只能用上clover方案了，因为ovmf在这里不可用，且只有clover是同时支持bios和uefi系统的。那么是否它能用在阿里云机器上启动clover？（5）驱动方面，有一些第三方uefi virtio driver的这些可以被以clover常用的方式用到黑苹果上（将virtio xxx.pkg放到kexts中使用），最近的Mojave 10.13.3/4官方才开始有virtio支持，它是OSX内部的virtio。不过blk只能工作在legacy模式。那么问题来了，是否采用clover的传统模式下绕过uefi，依然可以用上virtio？osx是一定要用到efi的。

PS:（4）（5）这二个问题的结果直接关系到我们接下来在阿里云安装黑苹果的可行性，首先，集中第一个难题：在阿里云上bios+mbr安装clover并使之启动

https://www.reddit.com/r/hackintosh/comments/atliaw/clover_mbr_or_gpt_for_legacy_bios_system/
https://www.reddit.com/r/VFIO/comments/av736o/creating_a_clover_bios_nonuefi_install_for_qemu/
https://superuser.com/questions/1382154/clover-boot-efi-file-on-non-uefi-computer

>> Your question touches two interlocked issues:

>> BIOS vs. UEFI boot

>> GPT vs. MBR boot

>> The problem you run into is, that Clover needs a BIOS boot, but BIOS boot implies MBR boot. So obviously there needs to be some magic involved - turns out, this is quite straight forward: A disk can carry both, an MBR and a GPT partition table, and they don't need to carry the same information - this is called a hybrid partition table.

>> So what you need to do is create a GPT partition table to your needs (including one partition for Clover - best put this first to make the next step easier)

>> Then create a "protective" MBR-style partition table, that contains the Clover partition as bootable, and everything else in a single primary partition of type "EF".

>> After you have installed Clover, when the BIOS-style boot runs via the MBR (installed by Clover), it will start Clover, which will in turn read the GPT partition table to boot the rest. This implies, that while BIOS sees only one bootable partition, Clover can see many.

>> I have successfully used this method to triple-boot between Ubuntu, Windows and MacOS.

总结上面就是说可以连锁boot，实现在云主机上启动clover，然后，virtio驱动方面，Google “clover virtio”,我们在网上还是找到了一些讨论：

http://philjordan.eu/osx-virt/ 不过只有for network没有for blk,这是UEFI层的。
https://github.com/pmj/virtio-net-osx
https://passthroughpo.st/mac-os-adds-early-support-for-virtio-qemu/ 这是OS级的

>> the VirtIO drivers are only in the virtualization firmware OVMF. It's not part of clover

>> Then you can't use VirtIO drivers. They are part of VM firmware to allow faster access to resources via the virtualization features of the CPU. If the VM is booted legacy then clover emulates the firmware over legacy interfaces, even if you built the VirtIO drivers they would not work because there is no access outside of the EFI firmware of the VM. I don't understand why your VM would be restricted to legacy booting, what VM are you using because I have QEMU, VirtualBox, windows server hyperV and multiple versions of different VMWare products and none are restricted to just legacy booting...

总结来说，clover下的virtio目前没有，不过此话的正确性有待查证，因为下面我们看到https://github.com/kholia/OSX-KVM通过clover用上了virtio pkgs.

osx guest方面的例子和自动化安装项目有:
-----

在虚拟机上使用OVMF的例子有，它们是不使用到clover的：

http://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/
https://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/index_old.html 
https://github.com/foxlet/macOS-Simple-KVM

也可以在虚拟机上结合OVMF+clover作UEFI，这方面的例子有：

https://www.kraxel.org/blog/2019/06/macos-qemu-guest/ 
https://www.kraxel.org/blog/2017/09/running-macos-as-guest-in-kvm/
https://github.com/kholia/OSX-KVM

不管如何，下文开始，我们准备硬碰硬，自己实践出真知。彻底实现在云主机上安装黑osx.



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338140/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



