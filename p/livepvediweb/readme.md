云windows云黑群晖云黑苹果云chromeos云pve，及实现在云主机网络安装云OS的增强方法(带镜像有演示)
=====

minstackos是一个基于debian,整合pve和electron开发栈as desk，集成了vscode/gogs/drone/cloudreve/magnetw的devops和日用系统，以配合我在《minlearnprogramming》一云多端云OS/统一编程栈的想法。见gitee.com/minlearn/minlearnprogramming。

演示
-----

![](/p/livepvediweb/demo.gif)

minstackos在线演示

> 客户端下载：http://d.shalol.com/clients  
> pve用户名root密码tdl，vsc密码tdl，gogs用户名root密码tdl，cloudreve用户名admin@cloudreve.org密码123456  

下载安装及用法：
-----

diweb.sh是一个生成和安装minstackos镜像的脚本，见gitee.com/minlearn/minstack，除此之外，它还支持安装云windows，云黑群，云黑果镜像。

(以下尽量在debian系linux云主机vnc界面下或本地虚拟机下完成,centos不推荐)

下载：

> sudo wget https://gitee.com/minlearn/minstack/raw/master/diweb.sh && sudo chmod +x ./diweb.sh

安装镜像：

> sudo ./diweb.sh -i minstackos | deepin20 | win10ltsc  | dsm61715284

上面系统5选1，都是原版镜像仅加了必要驱动和引导制作其它无修改，镜像都在互联 (minstackos镜像,<1G,deepin20镜像,>2G,win10ltsc>6G,dsm61715284镜像< 1G)，最好国内主机上运行

除此之外，你还可执行刻成U盘，给本地装机使用。先执行：sudo ./diweb.sh -i debianbase然后sudo ./diweb.sh -h 0|1 -i minstackos，生成基础镜像和minstackos镜像，比如，你希望得到基础镜像生成后可上传解压到你的服务器托管使用。

更多介绍：
-----

自助安装FAQ:

> 最低要求1h2g，推荐4h8g，小内存安装后请自行mount vda2加大页面文件到vda2  
> 安装后用户名tdl，密码无，启动beakerbrowser后可关闭进入命令行再次startx，点pve或vs进入相应程序。  
> 如何扩大数据区：默认20G，可以移动/minstackd至第一个分区再重分区加大此空间再把/minstackd移回来即可。  
> 如何清除数据：挂载vda2，清除minstackd/changes中的内容，重启即可。  
> 如何以硬盘方式启动：重启编辑grub条目，去掉perch，解包01-core.ldeb到硬盘上的某rootcopy文件夹内。
> 上述方法也可临时安装gdisk编辑磁盘大小，再重启开启perch，安装的gdisk对系统无污染。
> 只使用vsc/pve/gogs其中一个，可选关闭其它服务。  
> 更多。。。  

-----

本项目长期保存,联系作者协助定制minstackos包括不限于机型适配，应用集成等。

![](/p/livepvediweb/logo123zd15sz150.png)

