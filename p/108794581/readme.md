戒掉PC，免pc开发，cloud ide and debug设想
=====

__本文关键字：分布式IDE，cloudide，远程编码，远程调试,jupyter with visual debugger support__

编程界有关于语言的圣战，OS之争，也甚至有代码编辑器是选择cui text ide还是gui IDE的选择的讨论。这次我们讲的是云IDE，其实，我们一直讲的是devops和云可视IDE这类服务端直接支持的开发部署，从《ellie可视化》到《docker as snippter空间》，从《一种设想：在网盘里coding,debuging，运行linux rootfs作全面devops》到《分布式IDE：osx一个完美的开发集中》，这些都是从不同层面去说的，毕竟这样一个工作要解决好多问题，而为什么要把开发做上云的理由也很明了：devops在云端，服务器环境入口网络好。我们也一直在寻求某种“免PC的开发”：如果把发布部署做上云了，如在《去windows去PC，打造for程序员的碎片化programming pad硬件选型》中所讲一样，我们甚至不需要一台PC仅需要一个平板就可以在云端构建，或者一个专用programmingpad，，实现戒掉PC把一切放到远端，免除本地重复部署，比如借助devops可以在一次性的虚拟沙盒环境中编程部署，干净，永远守护，开箱待用，，----- 它是云OS扩展一级的东西，与lnmp,openfaas云开发云部署同级，如果说后者解决的是部署前者解决的就是开发调试，当然也有将二者结合的，我们甚至谈到一种天然绑定debug设施的os和appstack结构(press esc切换运行态/开发态)。

云 IDE是新概念吗？不不不，早在 2010 年就有成熟的产品了，其实类似baota panel的文件管理器当cloud ide也可以，但是毕竟它缺乏真正的开发所需的一条路径上的东西。云调试与云编程支持。这里的技术是把IDE分布化，基础原理依然是设置协议（语言服务，调试服务API后端化）和前后端分开，这样前端可以ported到任何设备，《用开发本地tcpip程序的思路开发webapp》讲到，Api分离，实际上是从业务逻辑中分离了gui栈，这是一种经典的抽象，也是很久以前桌面界的编程方案了。其实web不是没有过前后分离，只不过很晚才将前后分离做到云函数和devops式建立api服务池的程度。

目前有很多实现，但我们接确到的大都就是jupyter和vscode online这二个，jupyter面向文档嵌入代码领域而vs code online面向真正的传统IDE在线化，它们二者都有明显的前后协议层支持。针对语言编程和调试。它们二者都可以与docker打通，而docker实际上是一个贯通开发部署综合的devops，所以与devops又有了联系。

vscode/vscode online
-----

微软向云靠拢其营收已超过windows本身。github,azure,onedrive,vscode/vscodeonline都是例子，微软在 Build 2019 开发者5月份的大会上宣布了 Web 版本的 VS Code，即 Visual Studio Online Private Preview。在 2019 11 月 4 日发布公开预览版的 Visual Studio Online。

实现：

其实vscode本来就是一个分布式IDE和云化版本。(vscode本来一开始就是面向要被online的，它的插件设计规范中的Language Server Protocol，Debug Adapter Protocol就决定了,vscode的设计师一开始就是从分布式角度去做的,cdr的code-server就是这样的思路提出了web版的vscode，上面提到微软201911公布了 Visual Studio Code 1.40 版本，官方直接支持web搭建，只是没有code-server的完善比如它没有登入验证),前端上，从页面上直观地看，VS Online 就是一个 Web 版的 VS Code，vscode使用Electron(原来叫 Atom Shell),桌面vscode最初也是由atom而来，这使得前端很容易被ported到web，但这其实只是它的一个前端界面，而 VS Online 更强大的能力来自于vscode-server+remote development extensions，这里我们暂时把web或桌面那个editor界面称为前端，vscodeserver+remotedevelopment exs称为后端。

在vscode中，如果你使用了remote-devopment中的ssh组件（一般地，你用remote-ssh远程开发，用remote-container本地），会自动在远端下载vscode-server，这并不仅仅是打开远程文件这么简单（这仅属于工程组织文件托管范畴和代码托管），而是连IDE和插件服务都做在了远端了。------ 单纯的remote-developer是一个插件包，是用了同步/打开远程文件用的。在vscode的插件搜索栏中，搜remote插件出来的东西多得你理解不透。大部分是解决代码托管一类的，解决类似在wxsdk tool云函数同步到后端的功能(tencent-cloud-vscode-toolkit)，git可能是提交工具也可以是代码空间也可以是工程文件组织器。比如remote-github(注意到云函数和一些git平台，都有ide，只是git配ide白瞎了，因为没有调试的可能性。不过，虽然目前只是在测试阶段，微软已经实现了为github集成了基于vscodeonline的不离开页面的沉浸式webide,github人类的代码基因库结合全能可用的webide，这是极佳的整合和创新。),你要的那种网盘文件。codesnippter空间也有。类似google colab 网盘挂载。------ 真正发挥作用的是后端插件管理和其组成的那个IDE(ide是由大量plugins组成的嘛)服务，这正是vscode-server。只是架构上,，这个vscode-server，被没有被做成统一后端，vscode-server不能按版本单独部署，必须要用一个vscode-server-client来唤起部署，vscode-server按需部署必须至少绑定一个front。这二者不能随意组合,也没有配置项将IDE前端和vscode-server连接起来，所以，没用到远程插件服务的情况下，本地vscode开发并不需用到（比如，同为remote的remote-container并不需要用到vscodeserver?）。

所以，不满足桌面？不想使用浏览器？这也不是什么大问题，由于它的前后端分离，可以将vscode online接入后端云开发环境（或本地模拟出来的“远程”版本，不过这样失去了云存储和云开发的某些意义）把它变成vscode。同样可做到类本地vscode效果。。而vscode online是前后可分离,服务端可以是本地模拟的也可是真正远程的，更自由，允许桌面版连接服务端远程开发，不局限于web as front，当然如果可以你也可以定制出你自己的vscode online。如果你想任意组合这二者，实现"让web/桌面vscode统一remote后端"的功能：比如，你想在本地搭建一个GUI前端(仅把它当editor front)，却想调用后端的ide服务和插件，最终是为了编辑开发托管代码，或调用远程docker里面的应用呢，这当然也是可以做到的，1，自建codespace服务器(可以把它理解为桌面那个editor前端+vscodeserver)，在目标机器上安装 VS Code，搜索Visual Studio Codespaces (formerly Visual Studio Online)安装，并命令中注册，它的原理应该也是部署了一个vscodeserver。2，可以在远程直接安装github/cdr/code-server的那个web版，然后查看它带的codeserver版本，来决定本地桌面端可能会用上的版本对上，file->about->help可以看到它采用的code-oss版本和commitid。（2比1好，毕竟自带一个web前端是最基本的，桌面和移动端可以作为额外项添加）

这样你可远程同步代码 + 本地调试，远程托管代码+远程调试，也可本地代码+本地调试（vscode itself），也可以本地代码，远程调试。当然也有对接重量级docker和devops的。甚至上面谈到的“一次性的虚拟沙盒环境中编程部署”，代替程序员日常的维护多个虚拟机完成不同开发工作的需求(还记得程序员下班不关机这梗吗)。这就是remote Container。

> vscode/vscodeonline开源在github，叫Code - OSS,其源码倒是没有缺失和阉割也有完善的构建支持（https://github.com/Microsoft/vscode/wiki/How-to-Contribute#build-and-run），mit的开源许可也很友好，但是vscode二进制本身是this not-FLOSS license 的并带了telemetry/tracking，而且其remote-development却是不开源的，且规定二进制vscode-server不能被打包（见https://code.visualstudio.com/docs/remote/faq，Can VS Code Server be installed or used on its own?Why aren't the Remote Development extensions or their components open source?）。所以，类似code-server,Che,VSCodium这样的公司和产品就只能自己编译源码。集成docker devops运营（https://opensource.com/article/20/6/open-source-alternatives-vs-code）。https://zhuanlan.zhihu.com/p/98184765,code-server是vscode的魔改版本(基本上，code-server采用了vscodeonline里面的vscode-server组件+web前端，由于code-server必须要绑定一个前端，这造成code-server是前后一体放在远端只能web界面)。微软也有托管计划Visual Studio Codespaces。code-server也有。

体验:

这就要说到体验了，追求一种存储也在远程，而且最好一条龙开发部署都在远端的IDE。一切硬件无线和软件云化的云计算时代，讲究没有任何随身携带的东西，体验就跟使用云笔记一样（云上存储空间和执行空间全包，作为后端，前端只需要要一个GUI，前后端通过协议跨架构交互）因此在PC和手机上都有支持。云IDE这样的东西最讲究体验，体验是第一位的，比如它比本地版本不要落后太多。web端的肯定跟本地的前端，移动的前端会有区别，体验和技术上的

移动端有ios的Servediter（以前的vsapp），jupyter也有这类APP叫juno connect。macbook要arm化。也支持ipad开发，当然，功能可能稍微会有点受限。安卓端当然也有。这里的体验差别就更大了去了。

> 只是说实话我对ssh的稳定性很不放心。说实话它真不如一个网盘客户端（最好是一个类finder的集成在file explorer中）,不过市场中也有cloudsync的插件。web端的体验其实还可以，只要不刷新网页后台一直websocket连着。


jupyter
------

jupyter notebook最初来自于julia和科学计算，叫ipython,后来发展成了通用notebook工具。以前，我们曾介绍过它也是一个devops。因为它可以结合docker形成binder这样的平台和工具。jupyter只是个notebook，最初它用用来写文档中的代码块。后来被作为语言学习平台。不是面向生产和真正的IDE的。这类产品是cloudide，比如vscode/vscodeonline，VS Code 也有对它的支持：Jupyter Notebook Editor。

jupyter也是前后端分离开发的典型，它有一个协议系统，这种分离使得前端可以不限于webasfront，也可以是mobileapp。甚至小程序这种嵌入前端。比如第一步要将语言发展为repl(脚本语言天然有repl，而cpp这样的要借助一定手段或现有产品变通。如cling)，然后按jupyter的语言kernel协议写成插件接上jupyter。

它也有很多泛化产品：如接上docker的binder，如基于jupyter notebook的jupyter book(注意名字区别)，叫可执行的markdown（就是在md中嵌入ipynb片断，或者理解成在ipynb中写markdown），进一步可发展为利用云snippter笔记写blog,可导出整书pdf（我还没找到按toc能整书生成pdf的产品，那就成word了，gitbook算一个）。

还有如基于jupyter的可视调试：jupyterlab/ debugger。当然它也是分前后端和协议实现的。这就一步一步走向了真正的cloudide的段位了。

如果说上一代jupyter是面向文学编程和科学研究的，下一代是称为Jupyterlab的产品就有点ide的味道了，有Elyra这样的产品了。

-----

下一文探索发明lang.sh:把jupyter with debugger supporter和code-server整合安装在minstackos

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108794581/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


