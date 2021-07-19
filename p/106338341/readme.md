云主机装黑果实践（8）：利用clover，离我们的目标更接近了
=====

__本文关键字：clover 黑底白苹果 无进度条,clover white screen,clover legcay bios video, clover force vesa mode, prevent Mojave switch video boot while booting,Resolution for General Protection Exception/ X64 Exception Type 0D Error__

在文章7中我们讲到了使用clover来代替chameleon，因为似乎进行到上一文chameleon已经遇到了瓶颈，1，其调试功能严重不足，遇到黑屏，我们只能大致猜想是CPU不兼容或显卡乱配问题，不能从更细粒度上去确定问题所在，指导接下来实践，2，变色龙只有一种模式，即bios，通过这种方式作fake efi来（填充满足需要efi引导相关结构的OS数据块）引导黑果，因此对于现实主机/云主机功能上受制于从legcay pc的层面去解决问题，功能单一，（比如驱动）靠爬贴能得到解决问题的机会少。对于这二个问题，clover都有解决，clover可以从bios,csm,纯uefi去解决问题，提供黑果功能，有更多的驱动乱配和具备更多解决驱动带来的引导问题的方向，其调试功能一流，直接集成为工具。本文详细描述在qemu标准云主机型号（文章6，7所述命令行和libvirt版本i440fx虚拟机方案）上利用clover5070 iso引导运行黑果mojave。

前文简单描述了利用boot7做cdboot的思路，这里详述，我们利用变色龙源码来生成clover的cdboot7，注意到变色龙的srccdboot下有$(SYMROOT)/cdboot，我们复制这一段为：

```
$(SYMROOT)/cdboot7:
	@echo "	[NASM] cdboot.s"	
	@$(NASM) cdboot.s -o $(SYMROOT)/cdboot7
	@dd if=$(SYMROOT)/boot7 of=$(SYMROOT)/cdboot7 conv=sync bs=2k seek=1 &> /dev/null

	@# Update cdboot with boot file size info
	@stat -f%z $(SYMROOT)/boot7 \
		| perl -ane "print pack('V',@F[0]);" \
		| dd of=$(SYMROOT)/cdboot7 bs=1 count=4 seek=2044 conv=notrunc &> /dev/null
```

all embedtheme optionrom: $(DIRS_NEEDED) $(SYMROOT)/cdboot后面加一条$(SYMROOT)/cdboot7,make dist前把clover的boot7放进去$(SYMROOT)，cd到src,cdboot下，sudo make all，就会发现symroot下有cdboot7了，使用这个cdboot,就加载了biosblk driver，在virtio blk下可以看到fat32中的EFI,cdboot6不行。

然后准备5070clover iso的config.plist,kexts和drivers下不动任何东西(5070 iso，kexts/other下有一个fakesmc,drivers/bios下有apfs,audiodxe,fsinject,ps2mousedxe,smchelper,usbmousedxe,xhcidxe)：

熟悉clover的调试信息与界面,虚拟机调试进系统
-----

Clover的原则是尽量少添加参数，不懂的或不能确定其意义的不要添加(与chameleon一样他们被集成设计成尽量不必添加任何非必要参数就能启动一份常见系统，除非真的必要)，我们一步一步来，写出这个config.plist，原则是保留尽可能多的debug。而且与前文写的变色龙chameleon.boot.plist尽量一一对应，这样好理解，最终我们写成这样：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>

	<key>RtVariables</key>
	<dict>
		<key>BooterConfig</key>
		<string>0x28</string>
		<key>CsrActiveConfig</key>
		<string>0x67</string>
	</dict>

	<key>GUI</key>
	<dict>
		<key>ScreenResolution</key>
		<string>1024x768</string>
		<key>TextOnly</key>
		<true/>
		<key>ConsoleMode</key>
		<string>Min</string>
	</dict>

	<key>Boot</key>
        <dict>
                <key>Arguments</key>
                <string>Kernel="kernel" "Graphics Mode"="1024x768" ﻿"Boot Graphics"=No "Text Mode"=Yes -v debug=0x100</string>
		<key>DefaultVolume</key>
		<string>OSXKVM</string>
                <key>Timeout</key>
                <integer>10</integer>

          	<key>Debug</key>
                <true/>
		<key>Log</key>
		<true/>
        </dict>

	<key>SystemParameters</key>
	<dict>
		<key>InjectKexts</key>
		<string>Detect</string>
		<key>NoCaches</key>
		<string>No</string>
	</dict>

	<key>KernelAndKextPatches</key>
	<dict>
		<key>KernelToPatch</key>
		<array>
			<dict>
			</dict>
		<array/>

		<key>Debug</key>
		<true/>
	</dict>

	<key>SMBIOS</key>
	<dict>
		<key>BiosReleaseDate</key>
		<string>12/22/2016</string>
		<key>BiosVendor</key>
		<string>Apple Inc.</string>
		<key>BiosVersion</key>
		<string>IM142.88Z.0118.B17.1612221936</string>
		<key>Board-ID</key>
		<string>Mac-27ADBB7B4CEE8E61</string>
		<key>Family</key>
		<string>iMac</string>
		<key>Manufacter</key>
		<string>Apple Inc.</string>
		<key>Manufacturer</key>
		<string>Apple Inc.</string>
		<key>ProductName</key>
		<string>iMac14,2</string>
		<key>SerialNumber</key>
		<string>C02KK5W9F8J2</string>
		<key>Version</key>
		<string>1.0</string>

		<key>Mobile</key>
		<false/>
	</dict>

</dict>
</plist>
```

解释一下：

开头，RtVariables是全局的开SIP，任何黑果引导的先决条件，为了好理解，所以把它写最前面，GUI和BOOT是菜单选单界面配置（当然，这里也有本质上的引导参数的喂给逻辑），引导后出现在你面前的就是这个界面，所以写在第二前面，GUI段面向clover作preboot用，BOOT段面向darwin接手后booting用，注意到我们加了尽可能多的debug支持的情况下，还尽量使用的是文字界面，和preboot/darwinboot一致的分辨率，这是为什么呢？

因为chamleon或clover引导时，要么用图形要么用文字，chameleon有纯文字界面而clover只有图形模式（文字界面是图形模拟的textonly,consolemode都是图形），mojave在启动时会有二个阶段不断换分辨率，这所有的过程，都涉及到分辨率带来的显卡驱动对启动过程的影响讨论，有很多花屏，黑屏，包括前面我们说到的云机上黑屏，都可能是显卡驱动导致的，第一要素就是分辨率（vesa显驱用vesa mode）：

vga std显卡vesa下mode 800x600时，clover preboot没有进度条，这就是上面GUI段手动设为1024x768以上的原因,猜可能就是800x600与匹配的vesa分辨率任何一种都不符，，逻辑上就不让过去了(要进入dmsg工具查看log中起作用的vesa mode才能发现原因可是进不去就没法使用dmsg)，为避免显卡问题在preboot时就halt掉我们的调试过程。这里应该尽量使用保守的模式（一般clover和变色龙够智能会帮你选好一般不用干预），进去mojave时，受到virtualgraphic.kext的支持，分辨率有更多选择，系统也会自己换分辨率。

所以原则上可以尽量手动保持preboot和darwinboot stage1,stage2二过程，这三个过程尽量使用一致的分辨率（如果有办法的话，比如修改源码也行，chameleon->boot2->graphics.c static int setVESAGraphicsMode中默认将vesa设为280，我将它改为267 https://en.wikipedia.org/wiki/VESA_BIOS_Extensions#Modes_defined_by_VESA，注意，vesa3支持刷新率定义但不要用，以免造成意外），这样的预先处理可以免去很多问题。所以BOOT段OS显驱发挥作用时再次喂给prelinkedkernel同样的显示模式。注意到我还喂了尽量少用graphics mode和使用text mode的参数都是为了规避可能的显卡问题(喂给kernel as flags的照样可以设置在chameleon的boot.plist中)。

此时，直到BOOT的所有配置只能完成vga std下的preboot，接下来的	<key>SystemParameters</key>段有大讲究，也是显示相关的，本来启动darwin后的stage1很顺利，开始切换分辨率到stage2时，花屏过不去了，后来加了虽然也会花屏，但一会就过去了，实际起作用的是<key>SystemParameters</key>段，我在里面加了injecexts和nocache，但实际起作用的是<key>SystemParameters</key><dict></dict>空体这句，不是里面的任何东西。而且好像它的位置也要放在<boot>下面一个平等位。比如它放在SMBIOS下面一个位就没用。而且这个空体，对vga cirrus下clover preboot也有用，不加这个空段或放到不适当位置时，vga cirrus在clover preboot时显示End random seed ++++ 白屏，而且一直没进展，加空段放到BOOT对等位下时，也是白屏，但一会就进入到stage1,stage2。直到这里为止，已完成clover的引导进入到了darwin的verbose，虽然花屏或白屏，但最终进入系统，这说明这二种显卡的vesa都生效了,cirrus进入后是800600,vga进入后是19201440，只是进入前都有瑕疵一个开始花屏，一个后面白屏(对比系统报告发现不同,clover下bios变成imac10.1，chameleon保持了按原样14.2,clover下smbios有好多是可以自动计算的，写法上可以忽略)

注意到boot段中的Log true似乎并没有生效。不如直接把dmsg.efi弄出来为tools直观,好像LogEveryBoot有用。kernel听说如果不加入此参数"Kernel=..."，将默认加载系统缓存（kernelcache）启动，作用等同于启动菜单的”without kernelcache“选项。但似乎实测证明也不正确。<key>SystemParameters</key>那二段(clover下没有UseKernelCache=yes，这二段组合使用模拟了类似的效果，但似乎SystemParameters二条怎么调clover都是用prelinkedkernel,换成kernel cache这个kernel flags倒是可以https://www.insanelymac.com/forum/topic/99891--/不过没必要用)，prelinkedkernel是自动-f生成或手动使用 kextcache 将 kernel 和 kexts from /System/Library/Caches/com.apple.kext.caches/Startup/kernelcache共同 link 成 PrelinkedKernel，由System/Library/CoreServices/boot.efi启动，boot.efi 是平台特定的 EFI Driver(application), 由cloverx64.efi或真实系统的EFI 固件加载, 主要作用是加载 Kernel, 并将控制权移交给 KernelEntry.boot段中的kernel flags都是喂给/Library/Preferences/SystemConfiguration/com.apple.Boot.plist再给kernel或prelinkedkernel的。但是不要直接修改它。会造成文件系统损坏，修改这个文件很容易破坏镜像。可用clover在内存中patching。

上述启动过程中的花屏白屏，可能需要专门的驱动和patch过程来解决（vs clover，chameleon在这方面没有优势，这就是我们选clover的理由，clover可有csmvideodxe,qemuvideodxe，不过云机不能用这些，因为csmvideo需要开csm，qemuvidedxe需要开uefi，clover在mbr机上并不可以纯uefi，因此也不能使用uefi drivers，比如CsmVideoDxe-64.efi:Clover图形界面的图像驱动，可以有更多的分辨率选择。（仅限于启动界面）。他基于UEFI BIOS的CSM模块，因此需要CSM可用。UEFI GOP VBIOS只适用于UEFI引导的系统的,即UEFI引导的Clover和Ozmosis,用传统BIOS引导系统的同学请绕行）。不过不对引导造成问题，我们就不管了。

搜索花屏白屏，在网上可以爬到很多贴，单一个黑屏就这么多可能和可能的解决方案。也是初学者学习黑果第一大拦路虎，这里有趣也最容易让人弃坑，黑苹果文化博大精深，到底都是方案，可是不一定适用你那套。

我们可以在GUI段通过custom和scan将dmsg.efi工具形成菜单。查看debug log就更快捷了。跟Uefishell一样，Uefishell,就是把启动脚本化，而且是在线脚本语言化，跟grub一样，接上shell，都可以。但可运行一些.efi程序,load一些驱动，cp,ls一些文件，注意与cloverx64分开，它不可用于引导OSX，cloverx64才是osx lister和OS X booter efi(boot6,7 is the booter itself)，osxclover提供了二个efi booter，其实都是同一个文件，就是bootx64和cloverx64，如果充分利用CLOVER，甚至可以打造装机上的“统一实机云主机firmware”和统一BOOT环境,让bootx64是总boot。也许我们以后要把各种OS（包括boot6,7）做成boot+pbr，做成各种os的loader文件，让所有fat32 boot下的所有boot都呈一个样归进EFI.


云主机上的clover通用保护错误和黑屏问题
-----

如果说实机上的白屏，花屏不影响最终启动，那么云主机上的黑屏就是一个实实在在的先决问题了。这之前，还有一个错误，就是cloverx64在云主机上执行会报错General Protection Exception/ X64 Exception Type 0D Error。这是clover层面的导致出错。但也有可能是显卡导致的。The reason is the incompatibility between UEFI graphic drivers & OS（or boot loader）.cloverx64只是相当于Os lister+它可以引导osx,Cloverx64一开始在preboot就用了显卡驱动，不像变色龙在preboot用的是text console。由于大量使用vesa，所以它访问graphics driver时会发生type 64 exception通用保护错误。而变色龙往往在完成preboot启动后，darwinboot接手涉及到大量vesa时才会黑屏，既然从clover中得知，黑屏可能不是cpu而是从qemu模拟的vga（vgabios inside seabios）到os显卡驱动导致的，本地就不黑，云主机就黑，clover也是切换到这种图形模式就无法输出了，那么这样这种某种cpu保护错误还与显卡问题同源。

那么，最终这是CPU兼容问题还是第一节显卡问题在云主机特殊环境上的新问题？如果我们在chameleon有限的debug下看不到，那么在clover下丰富的debug支持下能不能看到问题的蛛丝马迹呢？

不过，可以肯定的是，clover下解决问题尝试的方向会有很多，在源码定制上，它基于omvf实现，有很多开源驱动可以定制，摆弄，众多黑果开发者维护了很多驱动成果。可寻求解决问题的尝试路径会更多。

让我们试目以待。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338341/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>








