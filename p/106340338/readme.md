硬件融合的新起点：虚拟firmware，avatt的编译(2)
=====

__本文关键字：硬件融合的新起点 , 融合PC和手机，操作系统并启/网装法,真正的virtual app__

在前面《在阿里云上装黑苹果》时我们谈到clover,它是一种能虚拟硬件的EFI而不仅仅是一个loader，其改写硬件逻辑的作用，是重启后依然存在的。（所以，它有可能对硬件造成损伤，切记不要在白苹果上使用clover，mbp使用clover丢firmware和坏屏的例子不少）。从虚拟EFI，到后来，我们又谈到parallel os bootloader和将hypersior集成到loader的设想，并最终发现了coreboot，而它居然可以集成linux和虚拟机，。。。。。这种loader层和firmware层面的虚拟化层面，为PC和云裸金属提供了一个地道的虚拟化层面。即firmware as infrature技术,它使cb进化到了具有融合硬件的作用，试想一下，现在的PC和移动端越来越强大，注重融合体验，可是都做得不够好，而如果走让不同的融合设备共享同样的infrasture，即统一硬件的路子，，就可以解决很多以前不好解决现今统一上层之后变得很好解决的大部分问题。

其实这个我们在《为你的硬件自动化统一构建root和firmware》也隐约提到过如，虚拟机virtual applicance 这些（vs webapp,nativeapp,etc..）都不好用，purism一体linux phone,laptop。现在放大来讲一下：现在做的桌面移动统一，都是从生态上整合，因为技术上内核从一开始就不一样。上流决定下流，这样会产生很多问题，兼容，适配，开发，separated codebase for different hardware。故，桌面移动统一，需要从硬件生态提出一个firmware层。之后所有的终端，不管你是谁，一律使用用户态OS即可，这种使用从firmware层（技术层）的融合可以解决绝绝大部分问题。比如一开始我们追求的OS并启，OS网装法，还比如类似上面的purism：使笔记本和手机共享无缝使用同一种linux，所有这类问题它都可以解决，以后就不用再使用chroot这样的技术给手机换OS了。还可以带来新的创新视点：还比如，基于融合OS上的融合语言层和融合开发层，如 as Devops tools（这其实是cb的一个辅助作用属工具性质）。还比如使用hyperkit统一虚拟化和容器技术，。使得虚拟机直接成为app runtime，提出真正的virtual app。就像GO语言的整合粒度那样。

关于硬件融合设想
-----

一个人其实基本上会用到各种硬件几件套，手机和PC，还有NAS，,办公PAD，编程pad，游戏pad。作为一个编程者需要接确到的OS，1，PC：装客户OS和NAS OS，同时运行，2，手机端装programming os和nas 客户端OS,试想下这些都能实现在现有硬件上。

因为你不是硬件厂商，我们只能从换firmware开始做融合硬件，有一些coreboot for phone的方案也在慢慢出现。其实coreboot也有for andriod后端，你完全可以通过这种技术给andriod手机写linux OS。那么laptop呢。最好的测试平台是买一台chromebook,测试cb的最好平台也是cb，google的pixel book和pixel手机总是第一时间被coreboot支持，至少早期的硬件性能不够就不要考虑了，如ubuntu touch刷早期 meizu之类的方案，当然对于programmingpad，我们做不了纯粹的融合硬件设想，比如像带实体键盘的programmingpad(实体键盘与触摸主要是一个能不能盲打的区别，这跟键子的边沿设计有关可以用键盘膜)，也不好直接从黑莓,titan上做，就不考虑。

总之，虚拟firmware似乎是融合硬件的终极大法，甚至还有人做成了融合ios:

>> Corellium此类方案其实也是从firmware层着手，如同clover for Mac，Phones using coreboot

>> Corellium iOS Kernel Research Virtual iPhone Hardware. Corellium virtualizes any mobile device hardware through software platform and allows you to test on various different devices without actually having physically had those devices. This is not the same as simulation as we get with Xcode and with other platforms where you would just simulate the iOS on a different iPhone through the Mac. this is actually a virtualized hardware it’s going to be basically a one-to-one clone of the device as if the device was a physical devices.

>> 要使用这个刷机系统和得到想知道的私人信息，你访问他们官网发邮件，回复要100万美元..

后来我们还编译了avatt，研究了coreboot的自动化脚本技术,这里继续：


编译avaat(2)
-----

在前面我们buildroot讲到会生成到vmlinux，vmlinux集成了ramfs的rootfs，在buildrom里会生成到work/coreboot/svn/targets/emulation/qemu-x86/coreboot.rom，最终cp到deploy，所以这是一种从Vmlinux->到payload->到rom的过程，那么我们如何在packer中自动获取这个deploy到host呢。

我本来企图用virtualbox-ose-additions-modules-2.6.33.3-tinycore.tcz在packer中装guest iso和设置packer的shared folder来实现。无奈几次都没有成功。所以，为了省事，最后找到这个，写在dbcolinuxbases.virtualbox中：

dbcolinuxbases.virtualbox中的改动

```
…
{"type": "file","source": "/mnt/hda1/tmp/scripts/buildrom/buildrom-devel/deploy/*","destination": "/Users/admin/Packer/output/","direction": "download"}
```

此除之外加上这二句，让packer变得更像vagrant一点:

```
"skip_export":true,
"keep_registered": true
```

还得加一个tce-load -iw python.tcz在适当位置

make.sh中的改动：

最后，由于packer是按上传给他的sh来检测改动，并在一次retry中生效的，形成一次debug，为避免debug周期过长，我们需要把make.sh多做几个脚本放开调用，这样可以避免修改一个sh后retry的等待时间过长。

上次讲到在coreboot编译时，这里继续

Package/coreboot-v2/coreboot.inc中把svn逻辑改为wget，由@ $(BIN_DIR)/fetchsvn.sh $(CBV2_URL) $(SOURCE_DIR)/coreboot $(CBV2_TAG) $(SOURCE_DIR)/$(CBV2_TARBALL) > $(CBV2_FETCH_LOG) 2>&1 改为：@ wget $(WGET_Q) -P $(SOURCE_DIR) $(CBV2_URL)/$(CBV2_TARBALL)

如果之前没有安装python.tcz，那么这里还会有：virtualbox-iso: ./buildtarget: line 49: python: not found

好了下面来开放那三个在《编译avatt(1)》中禁用的三个虚拟机相关项，qemu和ovzctl,ovzquote，我们只开放后二者，因为kvm在avatt中是不能用的。

vzctl-3.0.23 Vzctl.mk中，我们看到整个br的逻辑其实是一个wrapper.从这里可以看到vzctl采用的是package的自动，属于vzctrl.mk中尾端的那个调用。里面AUTOTARGETS是来自package/Makefile.autotools.in，里面define AUTOTARGETS $(call AUTOTARGETS_INNER,$(2),$(call UPPERCASE,$(2)),$(1)) endef，参照vzctl，找出qemu patchs的位置是package/buildroot-libtool.patch，

我们看到执行vzctl.mk会下载到正确的vzctl-3.0.23，但是编译的时候出错：

```
==> virtualbox-iso: mkdir: can't create directory '/mnt/hda1/tmp/scripts/buildroot/project_build_i586/avatt/root/etc/vz/dists': File exists
```

将vzctl.mk中的VZCTL_INSTALL_TARGET:=YES改为NO。因为这条重复了。

相反，对比vzquota-3.0.12 vzquote.mk，我们看到它主要是mk中的逻辑。没有使用到auto wrapper，其实vzctl本来就是用libtool和autotools编译的，所以会用到此auto wrapper。而vzquota不用。

一路通过。vzctl和vzquote会生成在ram rootfs的/usr/sbin,buildrom-devel/config/payloads 中运行这个rootfs的是COMMAND_LINE=console=tty0 console=ttyS0,115200 rdinit=/linuxrc

好了。最后。让我们本地用qemu测试运行导出的rom。进入本地output文件夹：

qemu-system-x86_64 -bios build/emulation-qemu-x86.rom -serial stdio

不加serial，会发现运行命令的那个console窗口和打开的graphic窗口都不显（这里显示payload,加-nographic可以让这个graphic不出现）。-serial stdio是让一切显示在console,可是我们发现在console窗口中在出现rootfs登录提示符前卡住了，上面的cmdline又是正确的，问题只能出在sket.tar.gz中，修正target_busybox_skeleton和target_skeleton中的/etc/inittab：

```
# Put a getty on the serial port
# ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100 # GENERIC_SERIAL
将ttyS0前面的#去掉。
```

终于出现avatt命令行了，你可以在这里root登录。你还可以在运行qemu-system-x86_64后加-hda xxx.image -m 1024 -net nic加硬盘，内存和网卡，当然这个rootfs似乎并不好用(target中只开放有限工具，甚至fdisk都没)，都甚至不能完整分配硬件资源测试ovz。需要修正。ovz也是32位的只认有限内存。

未完待续。

———

通用系统网装法，现在是网络时代，只要保证bios不死，如果还停留在u盘装机时代就OUT了。我们现在是用qemu运行，那么如何把linux不刷机弄为虚拟efi运行呢。比如放在efi文件夹就如同clover那样，不过这样就不能打造不死bios了。以后尝试。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340338/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



