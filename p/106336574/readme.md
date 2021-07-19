打造类手机刷机的win10 recovery镜像
=====

__本文关键字：打造小于4G的win10精简镜像,打造类手机刷机的win10 recovery镜像，打造统一bootloader分区 as pc recovery，非romos__

众所周知，windows系统不支持mount，其系统盘下往往有windows,user(documents and settings),programfiles这三大文件，在使用过程中，安装程序和用户都有可能往这三个文件夹中写东西使得系统盘变大，其中user和programfiles文件夹最容易变动，这使得使用windows，总免不了要经常重装系统。虽然windows安装完成后，有给desktop,document,video,photo,download文件夹换位置的功能，但是并不能达到使较稳定的os文件夹和极易变动的prog+docs这二大件分开的目的。而如果存在有一种类linux的mount，就可以随意将这二个文件夹指向到其它位置和大点的盘符，系统盘本身不易变动不再需要频繁去装且可单独备份/清理，（因为分离了前者，后二者整个删掉也可以做到相当于清理系统）。

在手机平台上，刷手机recovery/刷ROM/恢复手机/双清/应用缓存清理，这些技术很流行，手机系统的rom往往专门针对某个型号，给手机刷机的第一步搞的recovery，如果此步失败或断电中断，手机极容变砖。只是PC不容易这样，其实手机刷机和PC装机有很大共同之处，只是PC的设计就是给重装各种OS开放的，PC上的系统比较通用，即使windows也有几种版本。因此PC上有bios和uefi这样的方案。bios是写死在硬件上的，OS往往自带bootloader作为第二层，像windows的bootmgr,ntldr,linux系的grub,然后才是OSkernel+APP，那么uefi就是PC专门直接开放给软件层省去BIOS的方案，UEFI约定各OS将他们的BOOTLOADER做在硬盘第一分区的recovery，uefi的第一分区esp，就相当于手机上的recovery，复杂的bootloader可以是预安装环境，如windows pe，如群晖的webassit,如apple mac的在线重装界面。

那么看出来了吗，我们这里有好几个分离，1，将OS文件夹与DOCS+PROGS文件夹分离。这使得达到极大免重装（易双清）数据mount到其它盘，双清就相当于清理数据盘。2，将recovery和os分离，这样系统坏了并不会导致不能重装，（且易刷机，配合recovery里面的软件极易拿来刷机，这才是重点），易双清和易刷机这样的方案都有了，才像手机。

好了，下面我们来给PC打造这样一个uefi分区as recovery rom的功能，并。我们最后将产生一个<5G的多区段img，这个img包含了recovery,os（系统只读且固态，但非ramos，维持在4G）,data template,本着能双清就不要刷机的类手机原则，所以这个img一旦弄好最大用途只是拿来给别的同型号机器刷机。

材料：laomaotao winpe，我们主要依赖里面的diskgen做镜像和恢复而不是ghost，因为后者有缺陷稍后会谈到，还有CPRA_X64FRE_ZH-CN_ZZZ+++.iso这个win10镜像。

它展开有4.58G比4G大，我们需要用一些手段把它放到4G空间中去。准备工作：用U盘做一个laomaotao启动U盘，把这个ISO拷进去。好了，使用u盘上的laomaotao启动PC（我在一台只有一张16G的SSD盘的笔记本上做）：

１，制造system img和recovery img
-----

打开diskgen dev v4.9.6.564 free,把16G硬盘分成4个区，依次是400mb esp,100 msr,然后是4G的windows分区(ntfs)，后面剩下10G，先分200M出来承载data template(ntfs)，剩下先不用。（在镜像做好后，将镜像恢复到一台具体PC的数据区时，这个PC的数据区可以用所有剩下的部分，用diskgen的动态扩展分区就可以了），那我为什么不用上呢？注意：我这是在做镜像，而不是在使用镜像，是基于方便测试目的。

使用winntsetup安装这个iso到4G空间上，启动分区设为400m esp（如果不选这里安装后重启了，后期你只能用启动分区修复工具修复，它会将C盘下的efi文件模板拷到esp分区并做好bcd修改），等ISO安装完后就会将win10的bootloader拷进去。重点来了，上面提到ISO展开是实际占用4.58G的，所在在winntsetup上我们务必在右上角使用compact os方案，选第一个xpress4k方案，安装完后大约是3g的样子（为了使分区更小，你还可以找到dism++，点开空间回收，把硬链接选上，还可以腾出200-300M的空间）

ISO安装完成，拔掉U盘，重启系统，等它第一次运行，用户名和密码就用admin,admin。系统第一次进去完成，把必要的驱动装上（为了不让意料外的结果发生，最好断网离线安装驱动，事先把驱动解开放U盘，否则系统有时会下载更新包），插上U盘，进入winpe，还可以使用dism++精简一次多余的驱动，然后你再找到系统盘，删掉c\system32\driverstor里的FileRepository。又节省了几百M。其实此时C盘下的efi文件模板也可以删掉(如果你是winntsetup直接选的那么400m esp)，增加驱动+几次精简过后，windows区还是维持3G左右的样子，这正是我们要的结果。------- 但却不是ghost这种软件想看到的结果，如果你查看windows分区属性，会发现compactos之后的windows实际上是4-5G的，按正常方法把这个分区ghost备份下来，再还原到一个4G大的空间时，会提示空间不足无法还原。我试了diskgen的分区备份/恢复到镜像pmf文件也是一样

所以我们得把整个分区img拍下来。

我们在diskgen中新建一个硬盘镜像，保存到U盘命名为win10formypc.img，镜像大小400+100+4096+200=4696M，然后分别对应左边算式的位置建立ESP，msr,os,datatemplate区并格式化。现在我们右击这个虚拟硬盘的4096系统区，点克隆分区，把做好的16G硬盘上的4G系统区复制过来，按文件复制方式，发现是成功的。把它恢复到原来16G中的4G系统区，拔掉U盘，系统也是可以启动的。

这样我们就完成了制造系统分区（这个镜像依然只是拿来测试的，我只是跟你说明这种方法可以，这个系统还需要强化，把windows和docs+progs分区）。

再次U盘进入laomaotao,打开diskgen，右击老毛桃所在的400mU盘分区，文件复制把里面的boot文件夹（含10pe64.wim和boot.sdi）复制到我们16G硬盘上的400m esp分区，400M只剩50M不到了。打开bootice，选择esp里的bcd，编辑添加一条指向到boot/10pe64.wim和boot/boot.sdi的wim启动项，命名为lmt recovery，点一次保存当前配置，再点一次保存全局配置。拔掉U盘，开机测试是可以启动winpe的。

插上U盘重进winpe（依然使用U盘上的winpe），，利用给4G系统区拍镜像的方法，把recovery区拍进u盘上的win10formypc.img

2，强化system img和建立data template img
-----


好了，现在我们研究分离windows和docs+progs的方法。利用windows中的mklink。

插上U盘重进winpe重进winpe，一直注意到data template区是以D盘符显示的。先改注册表，下面的修改中，D即是把C中的文件夹转移到data template的注册修改，打开注册表编辑器，定位到16G硬盘系统区中的C:\Windows\System32\config\software，加载配置单元。将其挂载到任意根下，我挂的是HKEY_USER，因为这里条目少，命名为111，把下面的内容存为reg，导入，即会修改111中的对应内容。请自行研究注册表文件中对应修改的意义：


```
Windows Registry Editor Version 5.00

[HKEY_USERS\111\Wow6432Node\Microsoft\Windows\CurrentVersion]
"CommonFilesDir"="D:\\Program Files (x86)\\Common Files"
"CommonFilesDir (x86)"="D:\\Program Files (x86)\\Common Files"
"CommonW6432Dir"="D:\\Program Files\\Common Files"
"ProgramFilesDir"="D:\\Program Files (x86)"
"ProgramFilesDir (x86)"="D:\\Program Files (x86)"
"ProgramW6432Dir"="D:\\Program Files"

[HKEY_USERS\111\Microsoft\Windows\CurrentVersion]
"CommonFilesDir"="D:\\Program Files\\Common Files"
"CommonFilesDir (x86)"="D:\\Program Files (x86)\\Common Files"
"CommonW6432Dir"="D:\\Program Files\\Common Files"
"ProgramFilesDir"="D:\\Program Files"
"ProgramFilesDir (x86)"="D:\\Program Files (x86)"
"ProgramW6432Dir"="D:\\Program Files"

[HKEY_USERS\111\Microsoft\Windows\CurrentVersion\Explorer\ShellFolders]
"Common Start Menu"="D:\\ProgramData\\Microsoft\\Windows\\StartMenu"
"Common Programs"="D:\\ProgramData\\Microsoft\\Windows\\StartMenu\\Programs"
"Common Administrative Tools"="D:\\ProgramData\\Microsoft\\Windows\\StartMenu\\Programs\\Administrative Tools"
"Common Startup"="D:\\ProgramData\\Microsoft\\Windows\\StartMenu\\Programs\\Startup"
"OEM Links"="D:\\ProgramData\\OEMLinks"
"Common Templates"="D:\\ProgramData\\Microsoft\\Windows\\Templates"
"Common AppData"="D:\\ProgramData"

;其中这二条有一些16进制programdata，可以手动改下，注意鉴别，发现是对应C:\\文件夹的就改成D:\\对应文件夹，注意\\和\
;[HKEY_USERS\111\Microsoft\WindowsNT\CurrentVersion\ProfileList]:ProgramData
;[HKEY_USERS\111\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders]
```

卸载配置单元即保存了修改结果，然后打开pe中的cmd，复制粘贴执行下列命令：

```
xcopy "C:\Program Files" "D:\Program Files\" /E /H /K /X /Y /C
xcopy "C:\Program Files (x86)" "D:\Program Files (x86)\" /E /H /K /X /Y /C
rmdir /s /q "C:\Program Files"
rmdir /s /q "C:\Program Files (x86)"
mklink /J "C:\Program Files" "D:\Program Files"
mklink /J "C:\Program Files (x86)" "D:\Program Files (x86)"
xcopy C:\ProgramData D:\ProgramData\ /E /H /K /X /Y /B /C
rmdir /s /q C:\ProgramData
mklink /J C:\ProgramData D:\ProgramData
```

如果无误的话，它实际上完成的是文件硬链接操作，上面是关于program files,programdata和program files(x86)的。下面是关于user的。

```
Windows Registry Editor Version 5.00

[HKEY_USERS\111\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders]
"Common Desktop"="D:\\Users\\Public\\Desktop"
"Common Documents"="D:\\Users\\Public\\Documents"
"CommonMusic"="D:\\Users\\Public\\Music"
"CommonPictures"="D:\\Users\\Public\\Pictures"
"CommonVideo"="D:\\Users\\Public\\Videos"

;其中这二条有一些16进制programdata，可以手动改下，注意鉴别，发现是对应C:\Users的就改成D:\Users,注意\\和\,S-1-5-21-3843801140-3458922274-3296897442-500是你的admin用户
;[HKEY_USERS\111\Microsoft\WindowsNT\CurrentVersion\ProfileList] 下的 Default、ProfilesDirectory、Public
;[HKEY_USERS\111\Microsoft\WindowsNT\CurrentVersion\ProfileList\S-1-5-21-3843801140-3458922274-3296897442-500]下的 ProfileImagePath
;下面这条特殊处理，在pe中看不到，只有在退出PE进入硬盘系统后才能看到。C:\Windows\System32\config\default
;[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ShellFolders] 下的值看到数据中有 C:\Users的都改过来。
```
卸载配置单元，然后是执行硬链和转移操作：

```
xcopy C:\Users D:\Users\ /E /H /K /X /Y /B /C
rmdir /s /q "D:\Users\Default User"
mklink /J "D:\Users\Default User" D:\Users\Default
cacls "D:\Users\Default User" /S:"D:PAI(D;;CC;;;WD)(A;;0x1200a9;;;WD)(A;;FA;;;SY)(A;;FA;;;BA)"
rmdir /s /q C:\Users
mklink /J C:\Users D:\User
```

注意到d:\user\default需要重新赋权。生成的d盘仅200m不到。所以为它预留的200m是足够的。

都无误后，按1中给系统拍镜像的方法，格式化win10formypc.img中的系统区，把硬盘中的新系统区和data template区拍进win10formypc.img

重启进入系统，完成！！这个<5G的镜像打包后大约2-3G，以后在新机或本机还原时就进winpe，打开dg还原/重做镜像，其中datatemplate你需要扩展到你本机硬盘上剩下所有空间，这就是刷机，如果用这样的IMG做出来的系统垃圾多了，就稍微用清理工具清一下C盘，或整个删D盘。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336574/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




