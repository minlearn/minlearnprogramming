在tinycolinux上安装chrome
=====

__本文关键字：chrome as desktop shell,uniform web os for admin and user__

 一个APP总是由UI，中间件，业务逻辑组件，但唯有UI足以划分一个appstack，因为UI是一个APP必须的部分，即使是console也有TUI，现今我们看到的UI主要有二种，随OS发布的原生GUI，和随着webapp发展出来的WEBPAGE GUI，但实际上若好好归纳一下，VNC也是一种远程控制专用GUI。 硬件加速GL,DX也是一种UI，它是游戏APP的GUI，概言之，用图形或非图形技术实现的交互，只要它混合其它栈元素组成开发发布单元，它其实就可以是一种UI(你可以看到语言库和大型IDE中项目模板往往就是按appstack和UI类型组织的)，只不过技术实现上，因为WEB的UI往往是一种HTML渲染引擎的东西，所以它其实属于基于原生UI的高级UI，但是，无论如何，一种OS使用某种高级UI并以此建立起全部的APP生态是可能的，如果有这样一种OS，那么就法上它可以称为该UI的OS。

chromeos，webos就是这种东西，它展现的是webpage使用的appmodel完成的是web appstack面向的是webapp，用户可以单纯一个chrome就可以完成整个应用（当然webgame比起硬件加速的native cg game是二个东西），管理员可以用chrome完成维护任务，开发者可以就browser开发网页程序。chromeos就是一个linux系统核心+webkit UI组成的全部可用生态（desktop SHELL，AUI,工具，APP..），如果不存在还需要在这种OS玩大型3D游戏这种需求(况且现在已有webgl,websocket,html5这样的方案)，它其实是一种足够好用且可扩展到任何原原生UI和原生appstack占据的那些业务领域的东西。

其实,linux宏内核设计本来就是面向多样化被发布。它甚至可以per app os。chrome as os desktop all and AUI其实是合理的，它可以答配文尾提到的mineportal demos打造oc专用增强os。

好了，现在让我们在tinycolinux上安装GUI环境，以此原生UI为基础，实际上我们的最终目的不是这个，我们是要安装chrome，把它打造成类chrome os的东西，最终将tinycolinux发展成面向webui和webapp的专用OS。

在tinycolinux上安装x环境
-----

根据http://wiki.tinycorelinux.net/wiki:adding_a_desktop_to_microcore有xvesa和xorg可选，我们安装的是full blown的xorg而不是tinycore.iso中自带的精简的vesa,因为chrome需要xorg，这次我们选择从3.x的tcz repos中下载而不是4.x的。

依次下载解压Xlibs.tcz,Xprogs.tcz,pixman.tcz,fontconfig.tcz,Xorg-7.5-bin.tcz,Xorg-7.5-lib.tcz，Xorg-fonts.tcz,Xorg-7.5.tcz,8个文件解压完先重启一次，不要马上执行startx(startx在Xprogs.tcz中),重启后在home/tc下执行startx，提示发现不了/etc/sysconfig/Xserver，手动准备/etc/sysconfig/Xserver文件，内容就是一行Xorg，保存，重新startx发现已经能够进入桌面（且以后每次重启登TC用户都会进入这个桌面），只是没有窗口管理器和右键菜单。

以上是xorg的configless配置，所有的配置都是用户配置，生成在home/tc，每次重启进入TC都进入桌面，是因为第一次/home/tc下startx已生成了配置文件，重启发现都会自动进入原先那个桌面(xserver文件那行xorg和configless的效果)，，加了新东西后测试或重来可删home/tc所有文件，重新在/home/tc下startx会生成新的一系列配置文件夹。

现在在基础桌面环境里安装flwm和wbar.tcz(mac style docker?),重启依然没有窗口管理和右键菜单，这是因为一直没有启动flwm，看来startx并没有在home/tc配置文件中将启动flwm逻辑加入其中。在tinycorelinux bootcode中加desktop=flwm，重启，现在有桌面和右键菜单了。

安装chrome
-----

我下载的是3.x的32.6 M大小，版本为14.0.835.186的chromium-browser.tcz,在完成安装了x界面后，剩下的基本就是安装chrome和依赖tczs了。依次下载并安装下列18个tczs:

（由于以后每次tc登录都自动进入了桌面，你可以外部开个putty执行以下命令或sudo reboot,也可在桌面右键-terminal）

```
atk.tcz,cairo.tcz,gtk2.tcz,gdk-pixbuf2.tcz,pango.tcz
dbus.tcz
dbus-glib.tcz
libasound.tcz
nss.tcz
libevent.tcz
libcups.tcz
libgcrypt.tcz
libgpg-error.tcz
nspr.tcz
hicolor-icon-theme.tcz
shared-mime-info.tcz
chromium-browser.tcz
chromium-browser-locale.tcz
```

（在此过程中，进入桌面右键-terminal，/usr/local/bin/执行./chroum-browser测试所需tczs.）

全部安装完后重启一次，右键桌面APPS-chrouim，进入chrome，发现弹出对话框是乱码，点最右下角的那个乱码按钮，进入chrome，发现标题栏和地址栏是乱码，就算是在地址栏输入英文，也是乱码。这应该是chrome标题栏和地址栏，工具栏这些地方使用的字体是系统中没有的。非系统编码中缺少网页字体显乱码方块（系统此时是en,chrome也用的en,en-us?在/usr/local/chromium-browser-addons/locales中发现无en但有en-us项，改名也无用，调整系统etc/sysconfig/language也无用）

发现调chrome设置（乱码菜单中那个扳手图标进入）中跟字体，编码有关的选项都不行。这应该是这个prebuilt chrome版本的bug.

不过此时的chrome已能浏览网站，https的浏览不了。应该要源码重新编译。留到以后测试。

-------

此处的为tinycolinux装GUI技术可以运用在将tinycolinux打造成virtiope这样的地方。恩恩

本文也是为《web开发发布的极大化：一套以浏览器和paas为中心技术的可视全栈开发调试工具，支持自动适配任何领域demo》一文作铺垫，这文中的demos设想如果全部完成，那就是bcxszy pt2 mineportal demos总成了，mineportal是一套demos集选型,xaas部分为diskbios，完成mineportal的平台选型,langys部分为engitor，完成mineportal的开发发布选型,appstack,apps部分为deepinoc，完成mineportal的源码选型,这三大demos最终为了使得基于大web的oc装箱可用，在线开发，集成一切必须的学习支持。学习者可以通过研究它的实现获得PHP开发的知识，且积累自己的codebase.

关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692515909/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



