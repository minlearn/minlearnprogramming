把群晖mineportalbox当mydockerbox和snippter空间用
=====

__本文关键字：把群晖当真正产品级的mydockerbox用,利用docker hook dir as volume在群晖上制造snippter空间__

虽然在前面的《群晖docker上装ellie》和更前面一些文章中我诟病过docker aufs的诸多弊端，但不可否认的是，docker这种能虚拟user mode OS能虚拟application的机制能成为开发和部署的利器，比如结合git server webhook能构建出dockerhub这样的自动构建/发布平台（除了开发，部署，甚至于dbcolinux中我用openvz模拟其用于“装机”），windows 10以后也把windows2016中的docker技术应用到了桌面级-从此桌面沙盒程序将一切程序数据和程序文档数据隔离在container内，动态装卸，能在APP级允许程序实现类手机应用的“缓存清理”---当然windows docker也可用于服务性程序。bitnami打包的服务器程序已经基本全部有dockerized版本。。。越来越多的迹象表明，docker这种虚拟化机制还是十分流行和有用的。

其实群晖套件spk就是沙盒原理，群晖在技术上可以仅实现为docker only box，比如它的photostation等都可以实现为docker，变成纯dockerbox用。变成类上面win10+docker支持的hostos+guest all docker化架构，只是它要兼容没有docker支持的机型，所以不便全盘docker化。

好了，现在接《群晖docker上装ellie》，继续讲解未完成的东西，接《使用群晖作mineportalbox（2）：合理且不折腾地把webstation打造成snippter空间》,本文也会讲如何在群晖上把docker打造成又一个snippter空间:而且这个空间也是可以统一像home photostation,home cloudstation一样置入home中的。

在群晖上装ellie docker-compose.yml
-----

群晖6.2的docker是不支持直接安装docker-compose.yml的，只有把minlearn/ellie-corrected和postgresql9.5的镜像先在群晖中下载下来安装，然后先用postgresql镜像开一个容器postgresql1，主机端口不必是5432，但容器端口要是5432，密码环境变量设置一下，volume可以新建一条挂载到主机的home/docker/postgresqldata中，对应docker的/var/lib/postgresql/data，定为rw。这个目录会屏蔽docker中的/var/lib/postgresql/data，因为下一次你使用不同的主机目录把它volume出来到新的主机目录，出来的数据是全新的postgresql给你的模板数据，所以，任何主机目录中的volume对应体，在重新链接或转移时，应该备份一次。

然后，用ellie-corrected镜像开一个容器，按上文加上4000:4000和自定义环境变量SERVER_HOST=your ip。link到新开的postgresql1容器，别名定义database。ellie会启动，等待一会ellie便会连上postgresql，mix ecto-create,migrate,phx.server。等启动完成，创建快捷方式到群晖桌面为群晖的公网IP:4000。点击就可以访问了。
ellie-corrected的volume
在上文讲到，ellie docker-compose.yml中为ellie-corrected设定的那条volume会不能生效。问题是：你把容器的/app映射到主机上的/apphost，启动ellie后会在日志中显示/app/run.sh执行permission denied，这是因为，1：通过volume postgresql,volume ellie这样的方式mount到主机目录的目录是容器实例运行时，自动屏蔽掉容器端的那个目录的，而使用主机目录的一切的。包括权限。2，像postgresql的volume是不包含任何代码和可执行体的数据路径，所以没有执行权限问题。而ellie中有一个/app/run.sh要执行，要知道，docker内部的权限体和host上执行这些代码的执行体是不一样的。启动时ellie自然会有权限问题。

>其实群晖docker默认给docker volume划分的共享文件夹是根目录下的那个docker，在这里建立的docker volume依然会有权限问题。而群晖也没有为docker建立一个统一docker用户，因为各个docker内部的权限都是不一样的---分属各种各样的docker内部os template定义的用户。那么，有没有一种方法，将它转为主机上统一用户，使得docker出来的任何需要权限的目录在主机上都可以通行呢？

>有一个workarounds可以解决，统一把/docker或home/docker中，volume出来的主机目录设为EVERYONE读写权限即可。这样会有安全隐患，但群晖一般都是一台独用的，admin,guest都是被禁用的。所以能保证你当前用户不被破解，安全是可以基本得到保证的。

那么，最终如何实现在主机端访问到ellie /app，像postgresql volume一样使用ellie volume "/apphostdir /app"的方式呢？

修正ellie所用的/app volume
-----

VOLUME这个参数可以透出容器内部目录，当然不用VOLUME参数，在了解容器ROOTFS中存在/app的前提下（这需要用户看过ellie的dockerfile），直接volume出来/app到/corrspondinghostappdir也是一样的，使用VOLUME更能清晰化这种过程和目录(另一种bind mount)。

就如同WORKDIR定义了接下来在dockerfile中的当前目录环境一样，VOLUME也规约了一段dockerfile中上下文中，那些影响各层aufs构建打包过程中，向最终volume定义透露出来的最终aufs叠加层。一个volume指令后面的指令(主要是run影响aufs叠加）会对它上面的volume不产生作用。即，仅对调用volume指令前的那些aufs操作产生的结果有意义。

所以这个 VOLUME /app放在chmod+x /app/run.sh后也会是没有意义的。你可以尝试移动/app/run.sh使之放在docker rootfs的其它地方，自己尝试吧。

---------

好了，通过上述的讲解，你可以通过docker把一切程序用到的data和app数据归纳到home/docker/xxx下。将群晖打造成一个真正产品级的mydockerbox,和各种volume制造成的snippter空间。

关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340055/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



