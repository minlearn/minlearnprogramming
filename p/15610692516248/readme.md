msyscuione:基于msys的一体化CUI开发生产环境,支持qt,llvm,ros集成常见web appstack
=====

CUI又称TUI，作为一个开发者和云主机这种服务性环境的使用者，无论有没有意识到，它都是装机时我们大多数情况下第一要装的。linux往往天然集成语言环境和包管理（语言级或系统桌面级），这使得云主机linux装机量往往占首位。相反在windows下没有这样一套东西，因为windows往往作为终端windows应用往往面向要求图形界面的普通用户。

那么为什么需要这样一套环境呢？

1，cui环境是历史上程序开发和应用(部署、安装)原始形式，cui是程序上产出后的raw form,与GUI相对，GUI是高级封装形式。比如编译器这种东西历史上就是CUI后有IDE的。用法上约定俗成。仅需tui就够了；第二，服务性的程序往往也只需要而且产出时提供的就是其CUI的形式。不需要套一层GUI。也不需要像终端程序那样依赖复杂而频繁的GUI配置。复杂性程序本身也不需要透露太多用户界面用于配置。只喂指定参数即够。因此适合服务器环境。第三，有些需要batch配置的程序必定需要CUI，GUI反而不合适。

故，这三点其实可以看成是服务器开发和应用部署和客户终端的开发部署差别要求。

2，CUI是最接近被调用的。遵从生产部署的先后顺序列，比如一些API DLL本身能运行的话就是天然CUI的—dll即demo，开发即发布。程序的开发和生产往往是共享部件的近年来的java,.net大语言系统深刻地体现了这点因为它的语言环境有时可以作为可选系统组件（比如netfx系列），。运行环境与开发环境中的runtime往往天然一体，在脚本语言中，发布runtime往往意味着发布整个脚本语言环境。

ps:runtime=run time support,分开run和time并加了support才是重点，即runtime其实不是语言后端，那些supportting libs可能反而正是重点：提供对该语言开发的应用在run time的一切支持，包括前后端。
4，一句话，CUI是程序的原始形式。维护这样一个环境是必要的-它是继os core之后在PC软件上出现的第二大存在，这往往出现在windows和linux易用性之争上。或CUI，GUI之争中。
再来看这个msyscuione:

其实对windows上的cui的整合工作一直存在比如msys2,比如cmder，而msyscuione倾向于模拟了linux下的开发生产合一环境，全开源（未来可能与ros结合做成开箱即用的全开源高可用整体），并极力做到一个整块生态，即全部基于mingw,未来希望整块就小精。并尊重了多语言多开发的现实，将它们合理组织在langsys,appstack目录下只透露simple facades给用户（就像我的1ddlangsys=qtcling,1ddpractise codebase一样）。

大家知道一个生态有什么好处吗，我们现在接确到的每个应用的每个DLL都可能是大块的（比如chrome v8,qt dll），导入复杂的对象环境到内存。模块同一，你看windows的DLL其实全是由DLL组成的，它的每个DLL都是关于kernel.dll,user32.dll等的生态，这种小精性有如瑞士军刀自成一体所以快。不必一启动时拉大量第三方DLL，迅速占满系统资源。现在的APP普遍比较大因为web时代我们复用轮子的开发越来越典型了，一个APP都可以做得系统一样大，就是这个道理。

msyscuiinone被组织进了msys的文件结构的另一个的好处，是以后可以做sandbox，免注册表挂载。绿色激活某一组件到活动系统。就像云端（yuanduan.cn）一样，你可以理解为docker的fuse，或shadow filesystem

msyscui没有包管理，没有语言级容器。msyscuione将这一切留给现有语言或msyscuione可能不断增加的新语言支持，因为包管理往往与语言绑定是它们的机制，记住：程序的不折腾原则是在正确的层面干正确的事情。这是指抽象，而运营，可以选择一个应用切面渗透作已有整合，像微信小程序那样，一个应用强大了完全可以通过业务渗透+软件抽象整合，软件之道莫不如此。

————

msyscuione开发环境主要部件：

1,集成msys1.01
2,集成perl-5.24.0-mingw32 (比如为了支持qt等的shadow build)
3,采用i686-4.8.3-release-posix-dwarf-rt_v3-rev2(集成python,python2.7builtin)
4，集成qtcling
5，。。。
msyscuione支持编译的源码体系有qt和llvm/cling等支持ros免rosbe。


生产环境方面，支持常见开箱即用的那些webstacks,其实每种组件都能定义一种appstack，git加web也能组成gitstack,openvpn跟其它组合也能定义access server之类的东西，nginx也有openresty这样的增强变体，但webstack往往指wamp,wnmp这些简单环境，比如当今最常见的那些由一种动态语言加数据库加其它东西混合而成的东西它们没有层次,msyscui为他们定义了一种良好的语言/stack分开的层次。

msyscuione 应用stack环境主要部件：monogodb,mysql,nginx,git,apache,openvpn,ssh

———–

其它，msyscuione最小仅要求w2k3/winxp：

修正了mingw32的如下文件头，开闭其SECURE API支持,在win2k3/winxp上不会出现“找不到msvcrt.dll中函数入口”的错误

```
i686-w64-mingw32\include\_mingw.h
/* #define MINGW_HAS_SECURE_API 1 */
使用junction.exe替换了ln,使得一些需要创建软链接的编译脚本可在win2k3/winxp上通过。
junction.exe to replace ln.exe
```

未来还将支持更多..

下载地址见源站文章链接。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692516248/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




