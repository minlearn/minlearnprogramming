共享在阿里云ecs上安装自定义iso的方法
=====

__本文关键字：阿里云 自定义iso，阿里云 自定义镜像__

应用场景：

首先，最基本的目的：你想在云主机上安装自定义iso,比如一份精简或优化过了的镜像/高版本的系统镜像，而不是运营商提供给你的那些，或你想在在云主机上安装/ghost还原镜像变得跟本地一样方便而不用总是依赖于后台备份。

还比如，你想在阿里云海外linux主机上安装windows，但又不想花一月多出来的那20多元，这就要求winpe具备从linux完全转换到windows磁盘格式和系统的功能，再在这基础上安装自定义ISO。借助winpe virtio内置的grub4dos这能轻松办到。下面详述：

第一步，将winpe virtio放到你已有系统中。
-----

下载winpe virtio，如果你的原来系统和磁盘格式就是WINDOWS/ntfs系列，
1.上传peboot.rar,0pe.iso,winxpsp3.iso到你的云主机，直接将下载到的peboot.rar/boot解压到C盘；(默认系统盘为C盘讨论，这里只讨论安装winxpsp3.iso-实际它是61精简版本的win2k3，其它iso类推)

2，将c:/boot/windows/*.*全部复制到C盘根目录。根据4win.txt调整boot.ini内容，timeout=5要大些，比如调成50，这样vnc显示延迟能方便点到。
(如果提示文件覆盖，请自行根据情况处理。一般如果你的系统是准备弃用的，全是就可以了)
把0pe.iso和，winxpsp3.iso放到c:/boot/imgs/下。

如果是linux下，按与windows相同的方式和结构解压peboot.rar到/boot/下，和复制二个镜像文件到/boot/imgs/下
因为linux通常内含grub2,所以/boot/linux/4lin.txt指出要修改的地方，会与windows下boot.ini不同。

然后就是开机。VNC进入。


第二步 VNC开机进入winpe，处理
-----

### 如果你的原系统是windows

VNC选单，选选grub4dos->grub.exe，如果你的运营商支持通过tigervnc这种vnc进入的，比如west263，那么进入下一个单后可以直接用peboot中的安装菜单安装winxpsp3.iso。这里完成第一步，第二步，基本可以直到windows安装完毕。

但如果你是网页VNC，比如阿里云ecs，那一般如果按上一种方法第一步从iso复制完文件后，安装程序会将boot.ini改为3秒左右，下一次自动重启后等待时间过短，你一般点不到grub4dos第二步的选单出现(系统就循环自动进入第一步了，执行不了第二步，安装失败)。

这就需要进入WINPE。手动复制文件，不须用到peboot的第一，第二步。
（其实只要能进入winpe，在winpe下能看到镜像文件，之后的思路基本就很确定了，什么？还看不出来，那好，我们继续）

将winxpsp3.iso解压，然后执行i386.bat安装到C，回到文头所提，如果是其它版本的iso可以利用winpe下的ntsetup统一完成复制文件。重要的步骤来了：
将C盘已复制的系统文件中的boot.ini timeout改为50，这样重启后它就会自动找到第二步安装需要的目录了。整个安装顺利完成。

### 如果原系统是linux

如果你的是LINUX,别担心，依赖WINPE virtio版，依然可以顺利安装WINDOWS镜像。利用好wwwroot中的linux工具即可。

1.依然是vnc开机，选grub4dos->grub.exe,进入winpe后打开inetpub\wwwroot下的showdriver.exe，确认显示C盘(linux下的/)，再打开bootice，将C盘mbr弄为ntldr。pbr也是。

ps:为什么这步需要首先完成呢？因为鼠标在virtio下可能一会变得没用（原因未明），而bootice不支持快捷操作，(所幸除bootice外wwwroot其它工具都支持键盘，下面的步骤如果鼠标没用就键盘操作吧。。好吧，挺有点小麻烦。。)，所以要趁着鼠标可用的情况下先完成这步。
ps:将mbr/pbr弄为ntldr后，以下第3步复制文件的时候有20%的概率会卡死，那么整个系统就启动不了了，可能需要重来。-_-。

2.TmpRamStorage/ramdisk.exe虚拟出一个256m或512m的内存盘(我默认将设置文件改成了512m，你也可以改动)，这里即将作为临时区存放第三步中复制自C盘boot下的整个文件夹（大约2，300M）
ps:说到这，要求你的云主机内存至少512m，这应该是最低配了吧。

3,rdrext23.exe，从linux分区/，复制出整个boot到第二步创建的临时盘。其中有一些grub2的大文件，可以不复制过来。

4,然后，格式化C盘，将临时盘中的boot文件弄到C盘，再按开头第一步准备文件的那些步骤弄好winpe。这样，就完全完成将linux变成ntfs和windows安装盘了。。接下来的问题，完全就是上面说过的了。

最后，设置网卡和静态路由(仅aliyun ecs)
-----

最后，对于aliyun ecs,安装好的windows可能在正确配置了IP信息后不能上网。这是阿里云设有双网卡导致的特殊情况。

难点来了。如何设置静态nat 路由：

```
route delete 0.0.0.0
route add -p 0.0.0.0 mask 0.0.0.0 47.88.3.247(你的IP)
route delete 10.0.0.0
route add -p 10.0.0.0 mask 255.0.0.0 10.117.239.243（你的内网网卡网关）
route delete 100.64.0.0
route add -p 100.64.0.0 mask 255.192.0.0 10.117.239.243
route delete 172.16.0.0
route add -p 172.16.0.0 mask 255.240.0.0 10.117.239.243
route delete 10.117.232.0
route add -p 10.117.232.0 mask 255.255.248.0 0.0.0.0
route delete 47.88.0.0
route add -p 47.88.0.0 mask 255.255.252.0 0.0.0.0
```

反正我成功了,bingo!!


这步也属一个挑战了，其实你可以先备份你原来的设置，然后一条条通过route add填到这里即可。填这里的时候，始终要提醒自己的是：临时和你改为永久路由的，都要有效化（即显示在命令行route表里面）。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336557/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




