将虚拟机集成在BIOS和EFI层，vavvt的编译(1)
=====

__本文关键字:corebootv3低版本编译,让dbcolinux用上buildroot,在tinycorelinux上编译coreboot,kvm-coreboot,ovz-coreboot__

在前面《为你的硬件自动化统一构建root和firmware》文中，我们讨论了为平台和硬件自动构建linux dist的buildroot和coreboot(buildrom)方案。在那里，我们研究的是最新版的源码。文尾我们还谈到要把这一切port到cohdevbox srctree中且使用osx vagrant+plain qemu thru somewhat “libvirt plugin” under linux(毕竟qemu既是个模拟器也是虚拟机)，却发现遇到了几个困难：1)市面上却并没vagrant-libvirt to run qemu commands directly，vagrant与packer的插件都是面向kvm/qemu的。2）且osx下，与libvirt/kvm/qemu等同的方案却是xhyve thru vagrant-xhyve，3）第三，cohdevbox/dbcolinux的packer构建中我们是基于tinycorelinux384 32bit来进行对dbcolinux深度目录定制的，里面的kernel都是2.6.x的。且4）第四，要考虑集合进openvz/kvm as coreboot payload，即把buildroot和coreboot一起使用，比如coreboot会使用buildroot出来的结果。

—— 所以，源码的版本选择可能需要更好处理，困难4也可能会难于解决，不过，幸好之前我们也进行过在dbcolinux中集成ovz的工作研究。另外，我们还找到了avatt : all virtual all the time，avatt is a buildrom-based buildsystem that can be used to build a virtualization-aware BIOS based on coreboot using KVM or OpenVZ (latter is currently work in progress)，是20090705的一个项目https://repo.or.cz/avatt.git/commit/1ee14af6661f1ebb0f18b9d1241ef1a67278c5ab。它整合了buildrom(coreboot早期的自动化构建脚本，此功能最新版coreboot已整合)和buildroot并加了自己的一些default configs和patchs，avatt/buildrom可参见https://repo.or.cz/buildrom.git/commit/4f38d5c4d9e8cdb99fcb2bd5aac859605d46e4ef，其大约是coreboot V3一期的东西，至于avatt/buildroot,也是mirror的buildroot同时期的东西，值得一提的是其主要选择了ublic和i586作为target。

这成为我们本文将buildroot融合到cohdevbox srctree中的最好尝试。鉴于困难1，2我们依然选择使用packer而不是vagrant，这里使用virtualbox而不是PD（主要是PD business版商业化太麻烦，另外，还记得在linux上我们使用过packer+vmware?）。

准备工作
-----

定制一下packer的启动。在start.sh中。

>> export PACKER_CACHE_DIR=~/Packer/cache，加了这个可以将cache移出当前目录。比如你可以将它放到与>> output_directory（~/Packer/output）同根下一起。
>> packer build -force -on-error=ask dbcolinuxbases.virtualbox，加了force可以在每次clean后不用手动去删临时文件。

新的dbcolinuxbases.virtualbox文件里会是这样：


```
>> "type": "virtualbox-iso",
>> "guest_os_type": "linux",
>> "hard_drive_interface": "ide",之前在PD上，现在VB上这二个都是默认的others,ide，但一定要加。以免不创建硬盘。空>> 间是默认大小的。Guest os一定要是某种linux，否则无法启动某些TC grub条目。
>> “cpus”: 2,
>> "memory": 4096, 其默认"guest_os_type": “other”,只有512m，不足于编译GCC时内存用量，加这个就可以在不用提及>> guest type的情况下配置内存用量。
>> 我们调整了srctree下目录，将各种src和tc.iso,tc下各种3.x/pkgs全移到res下。srctree/中只保留res和scripts(里面是整个>> avatt源码)
>> "http_directory": "res/",
>> 我们固定了port，而不用脚本中使用HTTP_PORT参数。虽然硬编不够灵活，但可以免除一些麻烦
>> "http_port_min": 8000,
>> "http_port_max": 8000,
>> 在上文《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式（3》的基础上。我们在"boot_command”中新>> 增这二个包。
>> "tce-load -iw bash.tcz<return>",
>> "tce-load -iw texinfo.tcz<return>”,
>> "tce-load -iw tar.tcz<return>”,
>> "tce-load -iw quilt.tcz<return>"
>> 除此之外，"sudo sh -c 'echo http://10.0.2.2:{{ .HTTPPort }}/ > /opt/tcemirror’”,因为我们换了网络地址。
>> "provisioners”中，除调用Bootstrap.sh保持不变。传src，也变成传scripts，和一条调用make.sh
>> {"type": "file","source": "scripts","destination": "/mnt/hda1/tmp"},
>> {"type": "shell","pause_before":"1s","execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'","scripts":>> ["./scripts/make.sh"]}
```

make.sh的内容：


```
>> cd /mnt/hda1/tmp/scripts/buildroot/target/generic/
>> tar zxf sket.tar.gz
>> 如果不加以上，上面的传scripts会不成功。所以需要处理源码，除了generic中的，删掉scripts其它所有>> target_busybox_skeleton，target_skeleton，把generic这二个文件夹打包成sket.tar.gz放在generic下。
>> cd /mnt/hda1/tmp/scripts/
>> sudo make defconfig
>> sudo make
>> buildroot的整体逻辑与流程。我们看到defconfig实际上就是buildrom-defconfig加buildroot-defconfig，先后顺序很重>> 要。都是把.config先复制出来，然后sudo make
```

在packer中测试编译Buildroot
-----

上面准备工作完成之后，启动start.sh,进入条件测试，如果没有bash.tcz，可能会出现相关错误
Retry，会出现一些常见下载错误。首先是wget命令。这是因为tc384中的wget版本过低。
==> virtualbox-iso: echo "2009.08-git" >/mnt/hda1/tmp/scripts/buildroot/project_build_i586/avatt/root/etc/br-version
==> virtualbox-iso: wget: invalid option -- 'n'
Buildroot.config里面，去掉BR2_WGET="wget --passive-ftp -nd”中的-nd，retry

接下来的这些下载错误，属于文件没找到错误，基本在buildroot.config或packages/xxx/xxx.mk中可以找到修改地址的地方。将其找到，一一修改为正确的调用packer http sharing的逻辑：

>> 找不到linux-2.6.26.8.tar.bz2，BR2_KERNEL_MIRROR="http://www.kernel.org/pub/" 按路径改这个>> http://10.0.2.2:8000/src/dl/
>> Retry,找不到uClib-HEAD.tar.bz2,去uClib.mk，UCLIBC_SITE:=http://10.0.2.2:8000/src/dl/uClibc/snapshot。snapshot>> 等这种命名是ulibc的源码发布方式。这里的uClib源码要找0.9.30，在本地解压重打包成uClib-HEAD.tar.bz2，>> 记得里面的顶层文件夹uLibc-0.9.30改成uLibc-HEAD，如果直接用官方最新的，1)可能编译过程会找不到配置需要你手动>> 配置。也会出现error: pthread.h: No such file or directory对应不到头文件。2)高版本gcc编译低版本uclib及kernel会出>> 现unifdef.c 中的 getline 提示重复，将extra/scripts/unifdef该文件中getline全改成get_line，有三个地方。
>> Retry,找不到mpfr-2.4.1.tar.bz2，mpfr.mk中，改MPFR_SITE:=http://10.0.2.2:8000/src/dl/mpfr-$(MPFR_VERSION)，>> patches文件是mpfr-2.4.1.patch，同样放到mpfr-2.4.1/下，会在虚拟机自动被下载并命名成mpfr-2.4.1.patch。
>> Retry,找不到binutils-2.18.50.0.9.tar.bz2，改BR2_GNU_MIRROR="http://10.0.2.2:8000/src/dl/gnu”，binutils包放在>> src/dl/linux/devel/binutils下。
>> retry,如果出现binutils/objdump] Error 2，则是因为makeinfo没有安装，如果出现unknown command `cygnus’，临时>> 解决办法是将shell texinfo降级到4.13，通过下载编译源码编译安装比较老一点的版本。幸好tc384的makeinfo足够低。
>> Retry,找不到gcc-4.3.3.tar.bz2,gmp-4.2.4.tar.bz2,前面改过BR2_GNU_MIRROR,这里仅需将gcc包放到>> res/src/dl/gnu/gcc/gcc-4.3.3,gmp包放到src/dl/gnu/gmp
>> Retry,如果出现bug出错字样,可能内存不够导致编译gcc失败。可以retry一次或者在配置文件中多加一点内存，预留的2G足>> 够。
>> Retry,找不到ccache,改ccache.mkCCACHE_SITE:=http://10.0.2.2:8000/src/dl/ccache，包放好
>> Retry,找不到module-init-tools,改module-init-tools.mk，包放到src/dl/linux/utils

>> Retry,找不到linux-2.6.24.tar.bz2,之前那个2.6.26.8应该只是取得头文件用，这里才是编译真正的kernel用，改>> buildroot.config里面BR2_KERNEL_SITE="http://10.0.2.2:8000/src/dl/ovz,，注意前面的kernel是改>> BR2_KERNEL_MIRROR这里是KERNEL_SITE，有所不同。这里要放的包是ovz的linux kernel linux-2.6.24.tar.bz2，跟>> uClib一样要处理一下得到。直接下载legacy openvz的2.6.24-ovz008 branch 最新src tarball即可。重打包时保证包内根目>> 录为linux-2.6.24
>> 因为这里我们要绕过BR2_CUSTOM_LINUX26_PATCH="patch-ovz008.1-combined.gz"的下载及使用，因为这个2.26.24>> 是本来就经过了patched的ovz src，也发现二个BR2_LINUX26_CUSTOM=y和>> BR2_KERNEL_LINUX_ADVANCED=y要作何处理呢，这二不要动否则不会生成vmlinux。patch source的修改处为>> target/linux/Makefile.in.advanced：
>> 1)$(call DOWNLOAD,$(LINUX26_PATCH_SITE),$(LINUX26_PATCH_SOURCE))
>> 2)toolchain/patch-kernel.sh $(LINUX26_DIR) $(DL_DIR) $(LINUX26_PATCH_SOURCE)
>> 一个是下载，一个是apply patch，统统#禁用即可,其实也可在buildroot.config里找到开关：>> BR2_KERNEL_PATCH="$(BR2_CUSTOM_LINUX26_PATCH)”，置空””即可，我选择是前一种方法。

>> retry，找不到busybox-1.13.4.tar.bz2,改BUSYBOX_SITE:=http://10.0.2.2:8000/src/dl/busybox
>> Retry,找不到e2fsprogs-1.41.3.tar.gz，改e2fsprogs.mk中>> E2FSPROGS_SITE=$(BR2_SOURCEFORGE_MIRROR)/e2fsprogs和>> BR2_SOURCEFORGE_MIRROR="http://10.0.2.2:8000/src/dl”。
>> Retry，找不到pciutils-3.0.1.tar.gz，改pciutils.mk中PCIUTILS_SITE:=http://10.0.2.2:8000/src/dl/linux/pci和>> PCIIDS_SITE:=http://10.0.2.2:8000/src/dl/linux/pci/
>> Retry,找不到qemu-0.10.5，改qemu.mk中QEMU_SITE:=http://10.0.2.2:8000/src/dl/qemu/
>> Retry，如果没有tar.tcz，可能会出现tar解压错误unrecognized option '--strip-components=1'。
>> Retry，相当于在qemu srctree中make，结果出现
>> ==> virtualbox-iso: Makefile:3: config-host.mak: No such file or directory
>> 于是，在buildroot.config禁用BR2_PACKAGE_QEMU=y，BR2_PACKAGE_VZCTL=y，>> BR2_PACKAGE_VZQUOTA=y（Qemu应该是往kernel中加入kvm，后二者是加入Ovz?日后解决）
>> Retry,找不到fakeroot_1.9.5.tar.gz，改fakeroot.mk中FAKEROOT_SITE:=http://10.0.2.2:8000/src/dl/fakeroot/
>> retry，找不到genext2fs-1.4.tar.gz，改ext2root.mk中GENEXT2_SITE:=$(BR2_SOURCEFORGE_MIRROR)/genext2fs,>> 至于BR2_SOURCEFORGE_MIRROR在前面改过了。

终于到这里了，脚本将利用target_skeleton打包成，至于# BR2_TARGET_ROOTFS_EXT2 is not set，表示它并不会被使用，ext2的rootfs在嵌入式上很流行。但脚本默认使用了INITRAMFS,即BR2_TARGET_ROOTFS_INITRAMFS=y，它利用生成的rootfs.i586.initramfs_list文件。然后写入到echo "CONFIG_INITRAMFS_SOURCE=\"$(INITRAMFS_TARGET)\"" >> \$(LINUX26_DIR)/.config产生作用，这个linux26_dir是project_build_i586/avatt/linux-avatt-openvz，会在构建vmlinux时自动将rootfs整合到vmlinux内部。

整个此buildroot全程构建时间约<20分，若成功会在buildroot/binaries下会生成vmlinux(约4.4M大)，供后续buildrom使用

在packer中测试编译Buildrom
-----

virtualbox-iso: /mnt/hda1/tmp/scripts/buildrom/buildrom-devel/bin/fetchsvn.sh: line 9: svn: not found
不需要tce-load -iw subversion,我们只需要按与buildroot那些下载逻辑一样。在buildrom的sources下载到它需要的Mkelfimage-svn-3473.tar.gz即可，Mkelfimage-svn-3473.tar.gz怎么来的？在github.com/coreboot/coreboot搜索带mkelfimage的commits，https://github.com/coreboot/coreboot/search?q=mkelfimage&type=Commits，发现https://github.com/coreboot/coreboot/commit/ebc92186cc9144aaacd37ca1ae94fcff60ec577a，这条刚好是以前通过svn导入的3473：git-svn-id: svn://svn.coreboot.org/coreboot/trunk@3473 2b7e53f0-3cfb-0310-b3e9-8179ed1497e1。下载其包文件解压源码，将其移到一个svn文件夹下，然后打包成mkelfimage-svn-3473.tar.gz，放到src/sources中。

再做以下必要修改：在mkelfimage.mk，改MKELFIMAGE_URL=http://10.0.2.2:8000/src/sources/mkelfImage，然后把下面一句由：
@ $(BIN_DIR)/fetchsvn.sh $(MKELFIMAGE_URL) $(SOURCE_DIR)/mkelfimage	$(MKELFIMAGE_TAG) $@ > $(MKELFIMAGE_FETCH_LOG) 2>&1 改为：@ wget $(WGET_Q) -P $(SOURCE_DIR) $(MKELFIMAGE_URL)/mkelfimage-svn-$(MKELFIMAGE_TAG).tar.gz ，这句是参照接下来unifdef.mk中取来的

retry,找不到unifdef,unifdef.mk中改UNIFDEF_URL=http://10.0.2.2:8000/src/sources/unifdef
Retry,这里没有安装quilt.tcz会出现quilt:command not found
retry,找不到lzma443,lzma.mk中改LZMA_URL=http://10.0.2.2:8000/src/sources/lzma

脚本在将结束时将寻找前一步产生的rootfs和vmlinux。生成elf coreboot payload bios.然后启动coreboot的编译。

Retry,提示找不到coreboot-svn-3772.tar.gz，buildrom是将coreboot作为它的一个package的，在https://github.com/coreboot/coreboot/commits/master?before=91eb2816faa4b2689f30ca47ff2585cf79ac53f3+28735找到目标：https://github.com/coreboot/coreboot/commit/544dca4195a6fb77286299bf42d4db5813dc28ac，按上述类似的方法处理。

未完待续。

-------

我们要做的，会慢慢精简它到固定版本的buildroot脚本,然后把定制目录的dbcolinux移进去。至于使用这个rom，我们可以将其刷入bios中（当然，涉及到很多适配工作）也可以放在efi分区中。用处呢？我们可以利用其虚拟机管理功能建立devops，或用于类exsi裸金属式装机，在这个rom中加入moeclub的脚本作netboot系统安装运维达到osx network recovery的效果且更灵活,比如，使用局域网wifi usb disk存储镜像在线安装系统。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340332/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



