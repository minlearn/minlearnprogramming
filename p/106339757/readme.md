利用citrix xenapp and xendesktop建立你的云桌面
=====

__本文关键字：云办公。真正可用的个人云桌面,云下载云播放。GPO dns，xendeskop storeweb无法完成您的请求__

其实云电脑桌面，虚拟桌面vdi,瘦客户端这样的东西出现很久了，只是它们从来没有像今天的疫情时代一样让人对他们更为关注。一般来说谈到云电脑人们都会想到最初的vps，正如谈到云桌面这个人们会想到远程桌面这个工具层面的东西一样，但其实云桌面与办公，可以是内涵和外延很广的东西，比如虚拟化这些东西结合起来，造就了一些专门做vdi的公司，如vmware,pd,citrix，（citrix很早就提出了BYOD，带着你的设备办公的理念。它的xenapp and xendesktop产品就是vdi产品，当然citrix不止这个产品，它还有虚拟化全套，和workspace云工作台这样的东西-类似钉钉,slack）同类产品是vmware的view。。越来越多的云服务商开通了它们的云桌面，云手机产品。，，还有很多。

只是云电脑用于生产力和异地办公，甚至发展普及到个人使用代替实体电脑的程度，需要解决很多问题。相信谁都尝试过用一般的云主机和一般的远程桌面办公，用云电脑，性能和速度，成本，都是大问题。gd5446显卡的云机在远程桌面下打开有flash的网页，鼠标寸步难移，对于多媒体应用，有点接近云游戏的实现难度了，可是我们知道，现在这个时代，5G才刚刚开始，云游戏还没有被普及。

citrix，是如何尽可能在现有条件下克服问题的，又达到什么样的成果？citrix中的app,xdesktop，使用一种ica的协议。这种协议可以工作在恶劣的带宽环境中，极大利用带宽，而且可以实时多媒体（类似云游戏的那种）,可以打视频电话等，体验接近普通本地办公电脑，简直可以类比虚拟机界中的pd(pd的远程桌面也做的不错可以发布remote app)，云存界的icloud+icloudriver(其算法十分稳健体验好。比如删除中同步，单用户范围内秒传)。

反正我是怀疑这效果，所以我看了下阿里云的云桌面产品，阿里的云桌面称为图形工作台，按量测试，先开通集群，然后开通实例，只在杭州h区i区有机器，系统是windows 10 2019srv based on ltsc，最低2h2g的机器，看看显卡驱动,是qemu -vga std虚拟出来的机器，当然也有用vgpu显卡的。硬盘也是virtue scsi，非blk（我的装黑果文章中，用这种显卡会更兼容，装黑果失败了所以我来折腾citrix windows了），集群和实例都收费，集群更贵阿0.71元一个小时。先看效果。带宽给足的情况下，效果居然还可以，播放高清视频和本地差不了多少，也没有出现开flash网页鼠标飘飘的情况！可做小型云下载云播放系统。

这部分费用产生在哪？，citrix是一个分布式结构，讲究扩展和性能，部署在多机环境，基本大量使用windows的基础设施来构建自身（比如用windows的身份机制，用windows的iis，用windows的数据库，需要配合windows域控，iis,selserver一起工作，很吃内存），一般使用多台windows server来构建服务器集群（也有linux版的Citrix），服务器有二大件：desktop delivery controller（DDC，其中又分几个小件，其中有Delivery controller业务核心,Licensing Server认证服务器,studio,storefront这二配置工具,citrix director组件。）和virtual delivery agent（VDA），virtual delivery agent就是安装在按量机中的程序，服务器的主要部分delivery controller是安装在阿里云别的服务器上的 ，感情阿里云是在这里建立了各种delivery controller服务器，与citrix授权一起产生租费的（citrix收费的方式是订阅制），然后通过某台阿里云称为gws的域服务器（暂且这么命名吧）作认证透出服务，我们可在网卡dns处找到它172.16.0.2xx，

现在我们来考虑自建所有的服务器自测效果，如果自己建的也能达到差不多的效果，证明citrix是高度可用的,citrix的授权有30天的免费期可以拿来测试。我重新开了2台港区按量2H4G装的win2019 datacenter with ui，港区网络有时好有时坏，方便长时间看综合效果，，

1台2C4G测试机上安装通用服务器（域控，mssqlserver）和ddc业务服务器并配置，相当于gws
-----

这部分服务器程序在新装机器上，包括os在内约占2G多启动内存,随着运行会占更大内存。因此需要一个4G左右的机器。且Delivery controller是（业务核心）组件，生产环境按接入的用户数，基本上4G为起步。如果内存不够且是ssd机，手动加大虚存尝试。我开的4G。

先处理下服务器，，计算机名修改为ddc，表示这里安装的是通用windows servers程序和ddc服务器。方便辨认，计算机名也有讲究，licensing server用它跟citrix注册，因为这台测试机器要作为许可服务器在内的服务器，许可的主机会使用主机名向ctirix注册。本机如果有域先脱域(系统属性，计算机名修改处，从当前域转为WORKSPACE工作组，这样就脱域了)，注意，如果ping localhost得到结果是你的windows用ipv6 ::1表示域名，那么添加REG_DWORD值16进制，HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters\DisabledComponents，设置成20，重启这样ipv4就好了。dns为自动获取。

把官方按量机中的用户目录/下载中的二个文件citrix_virtual_apps_and Desktops_7_1912.iso和citrixcqi.msi复制过来（citrix专属驱动就复制不过来了）。其内就是主要的服务端安装文件了(也有客户端receiver for Windows and osx，在客户端,我们要使用到的就是其workspace app中的->receiver中的->viewer,注意到receiver 和delivery 是一对相反词)。打开iso，从iso复制出来安装文件夹到桌面，安装期间几次要重启计算机再继续，如果直接使用iso安装会失去挂载。

———
PS：如果你这台机器性能不够，其实你可以在这里仅安装dc，而把licensing server,studio,和storefront。甚至域控，都放在域内另外一台机安装，或者把许可服务作为通用服务放到域内另外一台机中去，为了达到动态组装，扩展，性能，这也是生产环境中推荐的方式,ddc不能在一台域控上安装只能在一台域成员上安装，除非你先装DDC再装域控，因为安装程序的逻辑是：当它发现本机有域控，就不能安装Delivery controller 和virtual delivery agent。至于mssql，它是可以跟域控一起装的，我们这里也是这样做的，虽然install sqlserver on a domain controller is not recommended。
我们的测试是域控,mssql,ddc全装在这台机。
———

执行autoinstall，安装DDC，组件统统选上除了Director（在安装程序的逻辑中，以上都可以部分安装，directer可选安装,），选上sqlserver，安装程序建议5g，要开的端口会提示你许可证服务器7279,27000,80828083,Dc:80,89,443,Storefront:80 443，统统在服务端防火墙处放行，最后发送珍断信息不要勾选，因为你是在本地用户下安装，非域下，安装完窗口末尾有一行提示”本机需要域才能配置DDC”。ddc安装完后，如果需要新添组件，可以通过programfiles/XendesktopServersetup/xendesktopserversetup.exe修改，需要保留安装程序文件夹。

现在安装域控，windows域控相当于osx server的部署描述符文件，服务器管理中添加角色和功能，,基于角色或基于功能的安装，选择ad目录域服务,接下来功能就不用选了，勾“如果需要，重新启动服务器。”部署后黄色叹号处需要配置，将此服务器提升为域控制器，添加新林，域名填ctx.srv。表示，这是一个citrix用处的srv的集群，这是域的内网根域名（整体很容易理解：这是一个命名为ddc，以ctx域林为根，自身为控域的域成员，使用的内网dns逻辑是xxx.xxx.srv，避免使用com那些结尾的域名，如果有人注册了这个com，就会登录到它的服务器，所以这里还是不用com的好），域名系统服务器全局编录，林功能级别保持为server 2016，netbios域名默认CTX。无法创建dns委派，略过，会自动重启，本地adminstrator会变成域用户主机会变成ddc.ctx.srv，下次登录不到本地administrator,只能用域逻辑登录。因为是域控，不可能再作为本地机器登录了。ctx/administrator自动也是本地管理员组。设置domain policy，让域内主机自动配置。比如如果你想让域内主机都不用按ctlaltdel登录，需要在计算机管理，组策略中找到域，default domain policy,计算机配置，策略，windows设置，安全设置，本地策略，安全选项，交互式登录，由没有定义改成已启用。

————
Ps:我们手动设置下mssqlserver，以免配置时等待过长。让sqlserver工作在域上使用ctx/administrator logon，默认安装方式下给你的logon帐号是nt认证built in network service,网络是tcpip enable, tcp all使用动态端口。这种方式下等待过长。

事实上mssql不推荐安装在域控上，如果你要接入另外的域机器上的mssql，需要那台机器上的实例选择了“混合模式”验证身份登陆，这样才可以顺利的进行接下来的远程连接。。Services account用CTX/administrator,密码填上。auto startup type。安装完后，开始菜单打开SQL Server configure manager, network configure, 找到实例for sqlexpress的protocols,tcpip enable,给于所有ip，enable,yes,端口1443，特别注意ipall也要1433,同样在这个窗口，去SQL Server services那里右击restart一下，windows防火墙里放行。打开防火墙→高级设置→入站规则→新建规则→选择C:\Program Files\Microsoft SQL Server\MSSQL14.MSSQLSERVER\MSSQL\Binn\sqlserver.exe，如果你的防火墙关的，不用去理这步。
————

现在来配置，使ddc和vdc运行起来。现在开始在studio中配置（如果登录的不是域用户无法继续），打开studio，交付应用程序和桌面，创建站点的交付控制器开始，，使用30天免费订阅(也可以在这里选择授权文件安装，也可在localhost8082处进行，这时，主机名就会影响往citrix注册)，选择空配置，并输入站点名称default，studio,storefront等要用到sqlserver信息，如果是连接外部域主机，三个位置都填”mssql域主机名,1433“，提示没有权限，使用同域的ctx\administrator继续,连接，正在生成数据库架构。正在创建storefront群集,完成。如果你使用内存有限的机器，配置服务阶段会卡很久。

第一个站点建好了。可以在内网直接用浏览器加html5查看，但无法连接远程桌面，因为站点中没有远程桌面或app定义，在storeservice可以看到http://ddc.ctx.srv/Citrix/StoreWeb/,,这称为receiver for web站点。进去但是里面是没有任何程序的。配置成公网访问，会在iis管理器default site处添加一个https443段在http段下，进去storeweb一样是空的(实际上添加一个citrix gateway)。我们来安装vda(同样会产生一个有定义的Citrix gateway).

再添加一个domain用户，为接下来加入域的vda计算机用，active directory用户和计算机处，找到ctx.srv,computers，添加minlearn@ctx.srv，不能改密码永不过期，默认就是domain users组。

第二台2C4G测试机上安装配置vda,桌面服务器，用客户端连接，运行
-----

这次安装DC本身，如果你是同一个镜像恢复过来的系统，要去windows/system32/sysprep/下执行sysprep.exe。进入系统全新体验oobe确定重新生成一个sid.这部分仅是一个代理占内存不大。但却是桌面应用所在的地方。当然性能也是越高越好，仅上网使用浏览器的话2G吧。

————
ps:Ddc不推荐与vda所在服务器整合，我们的测试也采用分开机制，Delivery controller中的Delivery controller组件是桌面分发的上层服务器，VDA是桌面分发的下层服务器，这个也算服务器，只要receiver才是最终客户端。生产环境中这二者往往装在不同机器上。citrix本身就是面向licensing to 500+ 机器起步的，况且不好直接拿服务端让人家远程进来吧？
况且如果你把ddc,vda安装到一台机去了。登录StoreFront后，会显示“无法完成您的请求”
————

dns设为前面win主机的内网ip，准备加入域ctx.srv。系统属性计算名中，先给这台机器主机名叫vda,表示这是vda所在服务器，将这个机器由组切换加入域，重启用第一个服务器上的ctx\minlearn用户登录，，整个机器域名会变成vda.ctx.srv，因为这不是域控服务器本地administrator会保留，minlearn用户在本地权限是受限制的,在域用户下，访问设置会出现没有权限或找不到，无法访问本地设备等情形，这要针对找到那个资源或设置条目，执行，通过开始菜单中的windows管理工具找，不要直接开始，设置。改dns，要控制面板，网络和internet，网络连接，解决这种问题需要在域控处一一设置权限。但是我们在本地提升它为管理员也可以，注销切换到本地administrator,在administrator下，本地计算机用户组管理，把ctx/minlearn升为本地administrator组，要先保证dns为域ip哦。

把安装文件拷过来，一样的方式安装VDA,receiver也要安装，因为我们呆会要本地测试一下，启用与服务器的中转连接，组件全不选，功能选一定选上实时音频，防火墙规则都是自动80,1494,2598,8008,1494udp,2598udp,16500-16509udp，，期间会提示远程桌面模式尚未配置（跟其它服务器一样，vda也包装了windows的会话主机当服务器使用），还有119天过期，略过，过程中需要注册到DDC，填ddc.ctx.srv，安装，发送珍断信息不要勾选。后期可以通过programfiles/Xendesktopvdasetup/xendesktopvdasetup.exe修改vda向ddc注册的参数。驱动管理器中发现新装了很多citrix相关的虚拟设备和驱动，主要是io设备。

————
PS：DDC与VDA也有一个高可用模式，允许recevier直接与ddc通讯：
虚拟桌面的代理VDA默认是与DDC之间每5分钟通信一次的啦，所以如果DDC都挂了情况下，VDA和DDC之间的通信就会出现问题。
如果 XenDesktop 站点中的所有 Delivery Controller 均出现故障，可以将 Virtual Delivery Agent (VDA)配置为在高可用性模式下运行，以便用户可以继续访问和使用他们的桌面。 在高可用性模式下，VDA 将接受来自用户的直接 ICA 连接，而不是由控制器代理的连接。这样就可以做到在DDC都挂了情况下依然继续使用虚拟桌面喔。就这是VDA的高可用模式。
不过我没有测试高可用模式。
————

继续返回第一台机配置。Studio中创建计算机目录，多会话操作系统，未进行计算机管理的计算机，其它服务或技术，添加计算机，计算机目录是安装了vda的计算机的目录,这里填vda.ctx.srv，添加一个交付组，帐号使用ctx\minlearn。

再次访问https://你的公网ip/Citrix/StoreWeb/发现有内容了，点击会引导你下载windows activex控件，或workspace客户端(其实我们只是使用workspace中的view来远程桌面，真正的workspace是一个工作流聚合程序)。我们的目标是使用客户端连接而不仅是浏览器。打开workspace，添加服务器地址，账户为域内minlearn。

成功！效果嘛，暂时看还是可以的。。

———

我后来用2台1h2g的轻量代替上面的测试方案，1台代替方案中的1-dc，studio,storefront(也即，仅保留ddc中的许可),1台代替方案中的2+studio,storefront(把除许可外的其它ddc部件移到这)。把非核心和核心服务器分开，做一个个人最低费用的且可用的vdi 集群（二台1h2g港轻量68元/月，三年是接近2000,其实我理想中的终端价格应该是>500-800元/一年，因为一般手机电脑3年就坏了，3年投入1000-2000是我的心理价位。云主机虽然不是终端还有云带宽，但价格下完全可以拿来类比）可是第二台机应该2h4g才好用。

Studio用来连接ddc,配置完后可以删掉？需要改动时再次连接就是。

也尝试过上面扩展中的2,3做成按需，和尝试过把按需ecs打造成关机0收费无盘网启系统节省费用，但最终证明这个是做不到的。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339757/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





