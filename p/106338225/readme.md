云主机装黑果实践（3）:得到云主机安装镜像
=====

__本文关键字：qemu kvm mojave enoch，，grub4dos 手动mbr安装到fat32，linux 挂载多区段镜像分区__

前面二文讲解了在kvm/qemu上尝试安装osx的工作，基本上，变色龙，四叶草，opencore的使用，就是在具体平台上尝试不同的patch参数(acpi,dsdt,etc)和大量第三方kexts，调试是最主要的工作，这三个loader支持都不相同，最弱的是变色龙，只是变色龙的mbr支持是最简单的和云机友好的。那么，作为一类特殊硬件，在云主机上主机启动遇到问题，该如何调试呢。又有哪些特殊呢？第一步是创建出云主机能用的镜像，上传测试。

前文我们使用的是PD开nested虚拟化实现了安装mbr patched 10.15。接前面的测试环境，我们继续来讲解。为了简化工作，我们要在deepin linux上事先创建这个镜像并处理，用raw格式而不是qcow2，得到的raw我们可以直接打包供InstallNET.sh用，我们这次选用的是10.14，因为10.15镜像有点大，而且我们尽量在linux下完成大部分工作 ——— 在osx recovery上分区很麻烦。

为了更方便调试引导区，我们要设置一个200m的引导区，并加入grub2和virtiope，然后chainloader变色龙iso。 —— 变色龙直接作为第一层引导好像在云主机硬盘上会有启动问题。

制作镜像结构和启动
-----

在《离线编辑skynas镜像》文中我们讲过挂载一个多区段的镜像，但是现在我们是从0开始的镜像,是空盘和空分区，不是一个导出的有结构的镜像，我们采用20G，10.14安装后约有11-12G，云硬盘最小也是20G：

qemu-img create -f raw osxkvm 20G
(或dd if=/dev/zero of=osxkvm bs=1024 count=20971520)

我们现在可以通过fdisk osxkvm, mkfs.vfat osxkvm直接操作这个镜像。使之有mbr和分区信息。但是注意：这是针对常规盘已有设备挂载点的（整盘已被挂载）那些操作，这里的镜像文件，虽然内部有分区信息（但没有被挂载），所以分区也得不到挂载设备点，不能经过格式化获得文件系统信息，也就不能实现挂到文件系统。

对于空白镜像，我们可以通过loop将整盘镜像虚拟成块，统一完成分区和格式化/挂载这样的工作：

losetup -fP --show osxkvm 
(P参数是带分区的镜像，show参数可以查看刚losetup到的盘，省掉一次losetup -a)

fdisk /dev/loop0进去分区,会自动转为mbr和msdos,n一个2048起的+200M分区，并a为活动，这就是我们的启动设定盘，其余空间作为一个分区，设置卷标BOOT,OSXKVM
(注意不要生成在PD的/media/psf/xxx，否则打不开, 始终要注意，镜像文件的不同步,这时你要重启deepin。或者将镜像移到deepin里面)

现在才可以格式化和挂载分区：

mkfs.vfat /dev/loop0p1 
mount /dev/loop0p1 /osxkvm1
（下回再挂载分区就是《离线编辑skynas镜像》文中的-o offsetnumber mount了）

好了，我们现在给它安装启动，我们不会直接在fat32里mbr装变色龙（原因：启动分区是调试必要的，云主机硬盘直接单分区hfs+启不动，4k硬盘装变色龙也有变数,变色龙装在fat32也有特殊：不能直接dd boot1f32，要把原pbr备份下来整个写入新的，再恢复部分老的，dd if=opbr of=/dev/rdisk1s1 bs=1 count=86 skip=3，dd if=opbr of=/dev/rdisk1s1 bs=1 count=90 skip=422 ）： 

所以为了省事我们用grub2引导wowpc.iso这些原来在windows下用的：

grub-install --boot-directory=/bootpart /dev/loop0
grub-mkconfig -o /osxkvm1/grub/grub.cfg
(这里/dev/loop0为虚拟块对应设备，/bootpart为fat32启动分区的挂载点,deepin的grub.cfg有延时设置不用update-grub)

为wowpc定制grub.cfg,加个menuentry "chameleon”，这里的wowpc.iso在接下来一节要准备好：

set root='(hd0,msdos1)’
linux16 /syslinux/memdisk iso raw
(记得在系统内apt-get install syslinux拷相关文件到boot：cp -av /usr/lib/syslinux/ /bootpart/syslinux)
initrd16 /wowpc.iso

把boot-macOS.sh去掉cdr发现这个raw image是可以grub2启动的,你还可以定制grub逻辑，加入tinycorelinuxpe。接下来就是制作镜像主体：那个osxkvm分区，和wowpc.iso引导文件了

Recovery里安装OS X得到镜像文件和引导
-----

跟据上文，准备mbr patched 10.14 install.cdr（提一下，替换A5,A6会出现apfs stop），全程记得用https://github.com/chris1111/Chameleon-macOS-Mojave-USB/releases/tag/V2 chris1111的安装包ChameleonEnochv2.4svnr2922-Post.pkg在EasyMBR-Installer1013.sh脚本完全后写入，不要用其它地方来的。

ps: 自定chris1111的配置：

FakeSMC-Extra-Options->FakeSMC-NULCPU->FakeSMC&HWMonitor:
HWsensors.V6.26.146，安装
FakeSMC,安装(Fakesmc是为把boot-macOS.sh中的device smc参数消减，所以接将kexts做进osx来解决，必备黑果驱动)
CPUSensors,GPUSensors,LPCSensors都取消
FakeSMC-Extra-Options->Extra-smbios:
Extr,安装
SMBIOS PC on Laptop->imac14,2，勾选
跳到最后Chameleon Enoch v2.4svn r2922->Chameleon Standard
勾选standard mode安装变色龙引导到虚拟U盘，，它默认以-Xmpc -v等参数启动。删掉主题default后启动会更清爽。

在osx下虚拟U盘复制以下引号里提到的文件到某一目录，准备制作wowpc.iso，并把它放到虚拟U盘Extra下

sudo hdiutil makehybrid -o wowpc.iso “boot,Extra,Extra/extesions等chris1111的变色龙引导配置文件所在目录”/ -iso -hfs -joliet -eltorito-boot “虚拟U盘standalone/i386/cdboot”/cdboot -no-emul-boot -hfs-volume-name "Chameleon" -joliet-volume-name "Chameleo" -iso-volume-name "Chameleo"

cdr和wowpc.iso准备好了，启动boot-macOS.sh进入recovery,如果启动有问题，加-f重建缓存启动，

在recovery里界面的磁盘工具中查看未分区的盘号，这里是disk1s2，发现不是很好看，二个区凑一起了，但是没关系，能用就行(要得到好看的，diskutil partitiondisk disk1 MBR fat32 “BOOT” 200M HFS+ “OSXKVM” R，在linux里按上面过程返工吧),

退出磁盘工具进入命令行，打入：sudo diskutil eraseVolume JHFS+ OSXKVM disk1s2

拷wowed.iso到boot分区根下：cp /Extra/wowpc.iso /Volumes/BOOT/

进入安装界面,开始安装OSX到上一步准备的镜像osxkvm，因为是在PD中是虚拟套虚拟，安装性能有点慢，安装接近完全，最后一分钟要等好久,把最后得到的osxkvm测试启动，最后关掉PD，把得到的raw image gzip -c xx>xx.gz打包，完工！。




-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338225/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




