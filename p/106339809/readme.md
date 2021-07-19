在colinux上装openvpn access server
=====

__本文关键字：colinux modprobe tun,在colinux上编译colinux编译,openvpn access server cant open /dev/net/tun__

在前面的文章中我们分别介绍过colinux和openvpn(它是一个足于被称为可搭建access point server的东西)，在《发布mineportal2》中我们将mailinabox做到了colinux，并称为mineportalbox，这里我们准备把openvpn access server也做进mineportalbox，使之成为一个能构建集私密个人网络访问点，邮件消息，文件存储，静态网站托管空间的一体便易环境。

首先要解决的问题是重新编译colinux，为什么呢？因为官方发布的colinux0.7.9.exe中，安装到硬盘中的vmlinux-modules.tar.gz展开到colinux->lib->modules下没有modules.dep.bin，这造成在colinux中不能执行modprobe目录，而这会导致安装完openvpnas后出现"cant open /dev/net/tun"错误（通过943访问web管理界面会发现openvpn服务没启动，点启动会出现这个错误）

1,在colinux中编译colinux
-----

我选择的是colinux2.6.33.7+ubuntu14.04 32位镜像（带gcc4.8）环境，下载源码后一般需要执行./configure,make download,make cross,make kernel这几步。（当然你可以make all不过那样不便于这里说明需要修改的地方），为了清晰起见，我按上面几步说明源码中要修改的地方，以使编译能顺利通过：

去掉configure中对depmod的检查或注释掉：# check_depmod "depmod"

bin/make-common.sh中，MINGW_URL修改为jaist的下载地址：MINGW_URL=http://jaist.dl.sourceforge.net/sourceforge/mingw

bin/make-cross.sh中，configure_binutils()，--disable-nls 后加--disable-nls --disable-werror

patch文件夹中，加2个补丁(在附件中下载，它们是ptrace-2.6.33.diff,vdso-2.6.33.diff,)，然后写在series-2.6.33.7中带入源码应用这二个更改

我在bin/make-cross.sh中还去掉了编译完binutils和gcc4.1.2将其临时文件夹删除的二条clean语句

好了，开始编译，如果你的colinux不是ubuntu14.04，也许上面的更改会有一些小的不同(比如还需要apt-get install zip textinfo flex etc..)。直到make kernel得到vmlinux文件和vmlinux-modules.tar.gz，删除现有colinux中的lib/modules/2.6.33.7-co-0.7.9目录，退出当前colinux，用新的2个文件覆盖colinux文件夹中旧文件，重新启动colinux会自动将新的vmlinux-modules.tar.gz注入lib/modules，在其中你会发现已有modules.dep.bin，执行modprobe tun，（tun是linux2.6x自带的，也是openvpnas需要的），进一步执行lsmod | grep tun，输出一行带tun的条目，说明tun运行成功。

2，安装openvpn
-----

wget下载openvpn access server二进制包，dpkg -i 包名进行安装，passwd openvpn设定一个密码，因为我使用的阿里云专有网络IP主机+colinux slirp网络与host相连，会给出https://10.0.2.15:493/admin/这样的管理地址（事实上，整个openvpn这个时候是属于内部体系的，我们在这步先保证openvpn本身服务能运行，最终外部能不能连上一条逻辑上所有的东西以后再谈）。事先将493和1194转发/透露出来，访问https://主机外网ip:493/admin/管理，进去发现openvpn服务是OFF状态，启动依然显示“cant open /dev/net/tun”，可是在第一步的最后我们不是已经modprobe tun成功了吗？为什么Openvpn 加载了 tun 模块，仍然提示Cannot open TUN/TAP dev /dev/net/tun呢？

放心，这里只是一个小问题，只是因为ubuntu没有自动创建设备点文件而已，手动在colinux中执行：

```
mkdir /dev/net
mknod /dev/net/tun c 10 200
```

即可，现在试试开启openvpn服务，成功显示 ON.

3，远程DNS解析使得VPN可用
-----

这个时候，通过https://公网IP:943/下载到的openvpn-connect-2.1.3.111_signed.exe是可以登录进VPN的，但是不能打开网站。这是由于openvpnas的安装脚本仅识别简单公网方式，不能识别复杂的aliyun专有网络和colinux slirp私有IP环境下内部地址的情况，使其最终变得可访问，这应是另外一个类似“将openvpn access server装在virtualbox或家庭内部nat设备”之类的问题，下面我们来尝试解决：

首先，进入https://公网IP:943/admin/管理界面，把server_network_settings页hostname由10.0.2.x之类的地址改为你的公网IP

然后进入vpn_settings页，由于默认安装方式下，三个主要问题（Should VPN clients have access to private subnets (non-public networks on the server side)？ Should client Internet traffic be routed through the VPN? Should clients be allowed to access network services on the VPN gateway IP address?）都是脚本选择yes的，这造成受DNS污染和劫持的客户端DNS解析都是不能工作的，我们需要接下来在DNS Settings单选区选择第三页指定DNS，主要DNS设为8.8.8.8，次要设为114.114.114.114，保存，update running server!!

重连客户端打开google等，发现可以打开了！

--------------

进行在这里，所有的工作只是保证openvpn服务正常运行，基本作为一个理论上可工作的VPN运行，而实际上，反VPN技术很强大，DNS只要从外入内就会有污染，这应该是更高级的VPN课题了，所以这里不讲，下回题目为《利用VPN打造游戏gamelobby client -- turn lan game into internet》。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339809/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



