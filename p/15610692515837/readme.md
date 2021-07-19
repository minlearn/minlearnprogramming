windows版gbc:基于enginx的组件服务器系统paas,可用于mixed web与websocket game
=====

__本文关键字：利用nginx实现paas,利用nginx实现组件化游戏引擎，(openresty)nginx+lua实现混合cs/bs一体化分布式架构__

在前面的文章中说到，enginx搭配任何领域协议引擎/逻辑引擎就能形成一个专门的服务器套装，enginx负责任何其它的事情。比如IO，安全，前后端其它组件的协配作为胶合剂而存在。拿传统游戏服务器来说，独立游戏（世界，地图,现实登录,转发网关，负载网关,etc..）处理服务器往往是将领域逻辑做成服务器的部分，enginx它本身没有游戏以上任一方面的服务器，但利用其可lua编程定制IO逻辑+胶合不同服务器的能力,可以实现和替换其中的一部分，比如，1实现不同的gamegate作消息转发,就实现了用enginx编程替换了其中的网关部分：

这样配合传统服务器就将其纳入到了一个统一的enginx生态。向高可定制服务器集群系统发展，（enginx即是服务器的框架的框架）：

一个现代APP无非由界面，存储，网络与交互，领域逻辑等stacks组成，enginx可以负责包括网络交互与安全在内的一系列事情，openresty+lua可定制的能力使得定制服务器集群变得高可用，一体化。使任何分布式集群形成appstack化。特别适用于定制web架构及其其它tcp集群架构。是服务器的服务器。
再比如，2,搭配msg middleware实现api和领域协议处理。甚至可以将领域逻辑引擎enginx生态化不需要外来服务器实现（基于lua的领域引擎不会比原生本地的服务器性能下降多少）。甚至向组件服务器系统发展：

比如，进一步，配合协议处理，enginx能使任何分布式长链接应用共享与WEB一样的语义化协议（不需要定制协议处理细节）：

比如，具体到网络交互细节部分（协议处理）的一种实现法，可以做成更一体化的方案，比如类web的协议封装，比如websocket，其实二端通讯，无论是基于多高级的应用层高级协议如HTTP，WEBSOCKET都要加上自己领域的那一层，这些是语义化的东西，PB即可以做。如果是简单基本websocket的游戏服务端框架的话。那么只需要提供网络支持即可（或者再加上一个协议文档化的东西比如pbc,portobuffer）。这样基本上就是一个简单的组件化语义游戏服务端框架了。

更甚至,配合语言系统，enginx甚至能使之成为一个容器性质(且以语言后端为基础的，下面会说到)的APP环境：

比如，当这种语言是一种脚本语言时，配合解释器开一个worker线程执行一段脚本就达了这个目的。（这就是不折不扣的paas+langsys backend baas了）。这体现了enginx，能直接接上语言，以语言后端真正成为领域逻辑服务器的特点且以容器的方式进行。这就是“组件+脚本组件+容器”了

好了,VS传统服务器，GBC即是以上谈到的组件服务器的一种实现：

gbc的特点 VS kbe：传统服务器集群与组件服务器系统
-----

这个对比几乎是专门的服务器集群（传统服务器）vs逻辑清希的脚务器脚本化组件（组件服务器）的区别了。

它有一台beanstalked和pbc组成的领域协议处理系统。niginx只负责io和中转部分。我认为这是除了语言后端的逻辑处理，其网络协议处理方面是作为组件服务器化的另一大特点，其以语言为容器制造worker的特点。每个脚本都是一个app，一个应用的特点，更是其同时可用于游戏服务器和一般化HTTP WEB服务器的二大努力。

可以看出，组件服务器的逻辑更清晰，突出语言后端，CS/BS全包架构，定制逻辑引擎方面的能力更强大。与单语言环境的PAAS相比可以同时接上多语言促成多语言环境下的PAAS。

gbc改造成windows版本
-----

gbc默认只在unix系发布运行，流程逻辑基本上是py virtualenv利用supervisor开启nginx,redis,beanstalk+2个app的守护过程：由于作为主体的openresty与其它组件都在windows上有实现，除在win下supervisor不能移殖外其它都可移殖所以可以轻易将其移殖到windows上。全程只多了那个supervisor，只要把这个去除（换成普通的windows支持的调用方式即可）,gbc本身的framework和package都并不用动。

改动的部分：主要是配置部分和启动部分（有四个文件start_server，shell_func.sh,shell_func.lua,start_work.lua需要涉及到和简化掉，前二基本可直接删除我把它做成了以下一个简化浓缩的bat如下），后二个文件需大改（涉及到很多路径修改的部分看下载）:

```
luajit %CD%\update_config.lua
cd %APPSTACK_ROOT%\openresty\
RunHiddenConsole nginx2
cd %APPS_ROOT%\gbcdata\
RunHiddenConsole beanstalkd -l 127.0.0.1 -p 11300 -b %APPS_ROOT%\gbcdata\db
RunHiddenConsole redis-server2 %APPSTACK_ROOT%\redis\redis.conf
cd %GBC_ROOT%\
 
REM 这里的路径要做成workerbootstrape中按approotpath为key取configs的形式:  即其中 local appConfig = self._configs[appRootPath]这句
start luajit start_worker.lua %APPSTACK_ROOT%\gbc %APPS_ROOT:\=/%/gbcdata/apps/welcome
start luajit start_worker.lua %APPSTACK_ROOT%\gbc %APPS_ROOT:\=/%/gbcdata/apps/tests
```

以下是效果和运行图：

本地下载：

gbc.rar


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692515837/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



