云主机装黑果实践（2）：在deepin kvm下测试mbr方式安装的黑果10.15最新版
=====


__本文关键字：QEMU 免kernel firmware运行黑osx，把enoch做进iso normal boot,建立10.15 enoch懒人安装包__

在前面云主机装黑果系列文章中，我们知道了变色龙Chameleon,四叶草clover作为黑果技术的本质，其原理就是重写了白苹果的引导。但它们不是一般的1）boot to os loader，而且也是2）firmware to os(fake devices, inject kexts，patch to kernels,配置tricks) ，即firmware as loader，黑果的技术基础就是依靠它们将kexts和配置写入安装包或系统运行时的内存代替官方,操作起来复杂度不低，因为它们都是为不同硬件写的，所以大量第三方kexts的注入选择，硬件平台适配和osx版本对应是一个考究，集成大量kexts可能使一台PC上的黑果”负重“（采用了过多的固件驱动），所以这类固件也不要在白果上用，可能损坏硬件，最有可能损坏的就是视网膜屏，绝非危言耸听。

它们都同时支持bios(mbr,mbr+gpt)和uefi（gpt）,可以启动windows,linux但又都对osx专门做了工作,都有相关工具，如gui wizard tools。但一般来说，clover更灵活更强大,支持更新。《实践（1）》文前面我们是实现了装在本地kvmguest上，使用了Chameleon。好了，我们继续讨论用它让黑果装在云上，阿里云云主机一般是qemu standard i440fx pii(x3)老intel机型，Xeon E5-2682 v4 以上cpu，bios机(如seabios8c24b4c)没有uefi，只能从mbr上启动，而且驱动部分是virtio。所幸硬件上这完全可以运行黑果最新版，经过一些破解，软件上也完全可以，我们要做的工作就是，将前文章第二部分启动脚本里的enoch做的固件和引导(包括支持某osx版本安装运行所有做的全部破解工作)的工作集成到osx内部/guest vm内部：

首先是那个显而易见的大问题：bios云主机是不能以外部喂给kernel的方式提供firmware的，那么enoch能不能像普通boot如grub那样，集成firmware到os镜像内呢，这是可以的，因为变色龙是一种firmware as bootloader，我们在grub2的命仅行下也能浏览文件，这里的loader本该是os的功能但居然可以共存，loader就是第一层os，linux是第二层os，可以前后启动，变色化也可以是一种grub2 loader，被安装在硬盘上且充当osx的传手，一前一后启动，—— 这就是说，mbr和bios机，也支持将一部分firmware放在硬盘上与os一起（就如同efi文件夹与os一样）。

不管如何，让我们实践吧，这次我们直接测试10.15。因为这次我发现PD居然有嵌套虚拟化，这次我们在OSX+PD虚拟机DEEPIN测试黑果,不用跟上文一样重启来重启去了。

1）让qemu免kernel在kvm guest主机上运行黑果
-----

工作依然是建立新的可启动iso开始。(最终我们要得到一份能InstallNET.sh用在云主机的安装后的磁盘image dd)，但是我们需要在kvm/qemu上测试一下，所以还是按常规流程做成iso。因为10.15比前文osx-kvm脚本中的createinstalliso.sh提到的10,11,12,13更高，较10.11，12，事情有了不同：10.14开始，osx强制运行在uefi和apfs上，做了这方面的硬性选择，我们需要破解其为mbr和安装在普通hfs+上。能找到的开始支持10.15的引导是enoch2921 https://www.insanelymac.com/forum/files/file/71-enoch/。—— Crazybirdy有一个仓库和脚本专门https://www.insanelymac.com/forum/files/file/985-catalina-mbr-hfs-firmware-check-patch/解决这二大问题，搜索Catalina MBR HFS Firmware Check Patch 10.15.x可到。下载，里面有一个MBR-Manual-Method15-20191209.dmg，解包打包成MBR-Manual-Method15-20191209.zip本地存储。还有一个enoch2922有几个配置文件，也下载。

研究里面的EasyMBR-Installer1015和说明文件，提到破解了让安装程序不作mbr和convert to apfs方面的检查，重点是脚本中的打包iso逻辑（Work on 10.15.txt What's the difference between createinstallmedia method, MBR-Manual-Method, and MBR-Automatic-Method?），使用上拖入一个大于10G的安装U盘分区和一个install 10.15.app就可以，内部逻辑上，它其实跟前文osx-kvm创造iso的逻辑差不多：都是提取安装包install xxx.app中的basesystem.dmg（这是安装程序，recovery），对kernel进行补丁，kexts注入，Extra写入，将install xxx.app本体写入（因为对kernel有patch和kexts注入，对标准createinstallmedia逻辑流程有侵入，所以并不同于供vmware+unlocker虚拟机用的，用写入到虚拟镜像盘或U盘的方式，然后createinstallmedia形成一个新的懒人cdr的方式。），最终生成一个cdr。

但是，1）这个脚本都设想enchos是外置的，引导本身并没有写进去，因为脚本作者都是设想这类脚本生成的镜像是供在osx下使用和u盘安装使用的，最后，变色龙引导以post install方式被安装，—— 这也就是为什么enchos文件crazybirdy也是分开放的，也是Enoch-rev.2922.pkg这类发行打包方式使然，，只有osx能option识别这类方式制作的“可启动”U盘。只有先运行enoch作为固件或loader才能识别其中写入的安装（本身并不包括enoch）。2）问题2，整个cdr也不是一个标准的可启动的iso（放到windows下或qemu -boot d -cdrom './install.iso.cdr’ -m 512M 是不能启动的），因为本质上，这个cdr是dmg转换过来的,在osx和qemu+ kernel enoch机眼中，是可启动硬盘，而不是可启动iso

好了，我们现在要把EasyMBR-Installer1015里的脚本改造成生成镜像供qemu免kernel参数用上mbr patched了的10.15cdr，而不仅是U盘和实体机，最好是windows下也能用的bootable iso。。未来还要考虑云主机的接受方式。

2)变色龙mbr启动iso制作，在osx上制作变色龙mbr启动iso。将kernel集成到iso内部。
-----


在2.4前变色龙是wowpc.iso发布的。但是r2922发布的是一个在osx上运行的“事后”package(追随clover系列风格)，离线安装到本地需要破解的黑果镜像或U盘。wowpc.iso里的boot加kexts加plist,后者gui方式需要手动选择大量项目（其实也不多）注意这并不是Chameleon wizard工具。

先创建一个虚拟U盘，供脚本使用。JHFS+即MacOS扩展（日志式），第一行创建的-fs是临时的，因为马上要在第3行重分区，第二行挂载，第三行分区为mbr盘中的一个hhfs+区要求10G大小保险，整个镜像11G，注意，去磁盘工具中查看dev node，我这里是disk3：

hdiutil create -o “install" -size 11g -layout SPUD -fs HFS+J
hdiutil attach "install.dmg" -noverify -mountpoint /Volumes/install_build
diskutil partitionDisk disk3 MBR JHFS+ Recovery 10G

未来这个recovery区即是脚本执行时要用的虚拟U盘。打开它。

然后安装r2922引导，用Pacifist打开Enoch-rev.2922.pkg提取其中的Core.pkg中的所有enoch引导文件到一个临时文件夹，cd到这个文件夹的/usr/standalone/i386下，1)首先Install boot0 to the MBR（我们这里是为硬盘生成引导，如果是要作成windows下的iso：用hdiutil makehybrid -o mywowpc.iso 从wowpc中或Enoch-rev.2922.pkg/Core.pkg解压出来的所有文件加你的Extra配置组成的所在文件夹/ -iso -hfs -joliet -eltorito-boot 从wowpc中或Enoch-rev.2922.pkg/Core.pkg解压出来的所有文件加你的Extra配置组成的所在文件夹/usr/standalone/i386/cdboot -no-emul-boot -hfs-volume-name "Chameleon" -joliet-volume-name "Chameleo" -iso-volume-name "Chameleo”），2)然后Install boot1h to the partition’s bootsector到disk3s1(boot1h为for hfs+,boot1f32为for fat32,boot1fx为for exfat，如果用这个虚拟U盘生成的cdr启动时提示Boot0 done,boot1 error，那就是boot1文件选错了)，3)最后Install boot to the partition’s root directory:

```
sudo fdisk -f boot0 -u -y /dev/rdisk3
sudo dd if=boot1h of=/dev/rdisk3s1
sudo cp boot /Volumes/Recovery
```

你还要把crazybirdy里面推荐的Extra配置文件复制到打开的虚拟U盘(这个U盘是可写挂载的)。

最后，打开EasyMBR-Installer1015编辑（这个盘跟脚本里的U盘实体属性有一点是不等的，比如它不能动态erase，故）：
# diskutil eraseVolume JHFS+ Disk1mbrJHFS ${devDisk1mbrInstaller} 从这里开始改diskutil rename "${devDisk1mbrInstaller}" "Disk1mbrJHFS"echo "rename virtual disk to disk1mbrJHFS done"

运行脚本后，选择我们准备的install 10.15.app和虚拟U盘拖入，运行到成功结束(中间会有lzvn需要放行。但脚本也不会等待，直接复制失败，所以放行后再次运行脚本从头建立虚拟U盘生成即可。)，它进行了mbr10.15黑果上前述脚本处理引导破解和firmware注入的工作（A1-A4），如果需要，你需要在镜像制作完成后额外手动完成以下动作还要二步：

A5. (Needn't with Clover, Need only if you use Chameleon with -f to boot Disk1mbrInstaller, use Pacifist v3.2.14+ to access Core.pkg.)
    Make directory of /Volumes/Disk1mbrInstaller/System/Library/Kernels first.
    Copy InstallESD.dmg/Packages/Core.pkg/System/Library/Kernels/kernel to /Volumes/Disk1mbrInstaller/System/Library/Kernels/kernel

A6. (Needn't with Clover, Need only if you use Chameleon with -f to boot Disk1mbrInstaller, use Pacifist v3.2.14+ to access Core.pkg.)
    Replace InstallESD.dmg/Packages/Core.pkg/System/Library/Extensions to /Volumes/Disk1mbrInstaller/System/Library/Extensions


这样，这个虚拟U盘就成为mbr patched,（注意它并不是脚本运行后要安装到的“系统目标盘”），等脚本完成，将其转成iso，${mbrdiskname}是脚本中的名字，实际名字会是Volumes/10153EasyMBR19D76之类：

```
hdiutil detach "/Volumes/${mbrdiskname}"
hdiutil convert “install.dmg" -format UDTO -o “install“
```

现在可以执行上文中的Boot-macOS.sh了，进入PD中的qemu-kvm，，这次，可以免kernel参数了，修改脚本删掉，内含enoch启动，它会自动找到分区中的installmedia：

以后，你完全可以re Detach re convert来重新修改dmg和生成新iso来解决启动中出现的问题或深度定制你的黑果。


———

其实黑果上登icloud也不会被封，如果你实在不放心，买个100元不到的iphone4，硬件上也登着。恩恩



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338198/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




 
