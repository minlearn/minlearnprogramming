发布wordpress for .net monosys,及monosys带来的更好的虚机+paas选型
=====

__关键字：wordpress dotnet,phalanger wordpress,phalanger on mono,mono xsp4 hang,mono xsp4 didnt response，asp.net空间当主机用，asp.net空间上安装新语言, web空间走socket/websocket，web空间部署c/s服务器程序，跨bs/cs。__

在上一篇《monosys as 1ddlang + 1ddcodebase》中，我们谈到phalanger是mono下对php5.4规范的实现，本文将接着那文的技术介绍如何将php和php应用融入monosys，及继续进一步探讨mono下绿化应用，免.net打包的问题：

php for monosys : phalanger
-----

为什么php总是一种不可或缺的语言呢？实际上它也的确得到了广泛的应用。

因为php是最接近C系语法的，如果没有python,且php一开始的定位是一门系统脚本语言的话，那么或许由c++转通用动态脚本最早的那批应该转的是php。而且它在发布级别的现代语言特性：考虑进并支持了组件，— 这点无一不与delphi一脉相承。。然而组件始终是单语言内二进制级的复用手段 — 它没有像.net组件一样的跨语言性。当然，不止php,其它native版本cruntime backends的langsys其缺陷其实还有更多。。

所以，php显然需要更进一步，比如将它加进一体化后端的mono中，使之真正成为多语言生态中的一员，可以免binding进行多语言融合开发。使之更为强大。这是生态各自为政的单语言无法比较得到的。

phalanger即是这样一种方案，并保持了与zend c php的高度兼容。甚至可以最小修地运行wordpress.

绿化的monodeveloper下编译phalanger wordpress，及部署到asp.net和xps4下
-----

首先是编译，选用的源码版本是github上Phalanger-920e736fff73350757cbbe41bba4a97d8196ff62，wp版本是WpDotNet-4.6.1_Alpha，通过上文中绿色monodevloper.bat启动的IDE即是默认以mono-3.12.0-gtksharp-2.12.26-win32-1为编译后端的IDE环境，这套源码组合在.net4和mono3.12下都能进行编译，在asp.net4+iis下和xsp4下都能运行，但是虽然.mono跟.net宣称高度兼容，（你还要是编译时运行时/sdk并非程序要发布到运行的目标.net/mono版本）

mono 2.8之后实现了完全的.net4.0，自 Mono 1.9 以来，ASP.Net 也能通过 Mono 的 fastcgi-mono-server2 在 FastCGI 下运行了，我发现很奇怪的一点: 它们各各在内部门的版本不兼容，比如.net4,.net4.5只是高度兼容，如果你的程序在.net4下编译，发布到.net4.5不一定能运行。同样地，mono内部高低版本也不甚兼容。但是同版本的它们在外部99%兼容，以下是使用IDE编译需要注意的地方：

0，仅需要生成三个文件，即phpnetcore,phpnetparsers,phpnetclasslibary,phpnetmysql(源码中的MySql.Data.dll默认的6.4.7.7不行要换成6.4.4.0否则运行数据库一直无法连接但用探针链接是可以的),wpdotnet.dll
 
1,工程文件中的改动：
去掉project options中use msbuild build engine及compiler ignore warnings非数字的字母(或整个去掉)
 
2,源码文件中的改动（处理运行时会出现的phalangerversion exception）：
SourceCoreScriptContext.CLR.cs:

```
_constants.Add(“PHALANGER”, “4”, false);
SourceCoreHttpHeaders.CLR.cs:
private static readonly string/*!*/PoweredByHeader = “phalanger” + ” ” + “4”;
```
我以为仅改动各dll的AssemblyFileVersion为4.0.0.0但是不行
 
3，运行时会提示找不到assembly载入异常等，配置文件web.config:

``` 
PhpNetCore, Version=0.0.0.0
<add assembly=”PhpNetClassLibrary, Version=0.0.0.0, Culture=neutral, PublicKeyToken=4af37afe3cde05fb” section=”bcl” />
```
其它不用改成0.0.0.0
 
当运行时为.net4时，且发布到asp.net下这没有什么可讲的了，结果也算在预期之内，也没有什么要注意的地方，基于生成的wordpress能正常运行，就是wordpress上传附件什么的你可能需要除mysql外再额外编译一个gd2扩展。

如果放在mono runtime下运行情况就复杂多了，让mono xps运行编译结果phalanger wordpress的过程和处理方法：

 
要放到mono中，把下面注释去掉
```
<!– <httpHandlers>
<add path=”*.php” verb=”*” type=”PHP.Core.RequestHandler, PhpNetCore, Version=0.0.0.0, Culture=neutral, PublicKeyToken=0a8e8c4c76728c71″ />
</httpHandlers> –>
```
下面的问题尤其严重：

没有现成的可用的xsp4和mono-fastcgi让你用，mono-3.12.0-gtksharp-2.12.26-win32-1中的xsp4是运行不起来的，打开localhost:8080一直不响应用，官方mono repos->windows installer中全是有问题的，会出现不响应的情况,google半天也没好方案。

说实话我也没能鼓起勇气从源码编译mono的xsp支持。我选择的是mono-3.0.2-gtksharp-2.12.11-win32-0，它能运行起编译后的wordpress，不过小问题还是有的：虽然wordpress即使能运行，但是各种小问题（比如卡卡的，后台能打开，前台不能，贴子永久链能打开，主页不能）都存在。换言之，这个wp如果不需要经常动它，跟zend php下的wp是基本一样的，但是需要二次开发的话，还是需要处理一些大或小的问题的。

.net空间 vs 传统单语言虚机空间或paas的优势
-----

一般的虚机服务商会推出二个平台，windows配置.net/iis,而linux配置php/apache等，但在虚机选型方面，其实我觉得与windows系列.net空间相比，linux系应该全系java,然后jphp,jpython，这才是二个平衡的对比。

可自定语言及扩展：

因为.net,jvm有统一后端特性可作http request handler。虚机也就有了用户扩展性，可以组件方式以放置方式安装新语言，不需要获取虚机所在root，自己为语言编译扩展。以支持新程序或新应用特性。

而其实有了jvm,clr这样的后端，传统直接架构在机器环境上的php程序，反而会因为经过预编译之后，变得更快，打包成组件之后，也不需要iconcube这样的加密工具。不直接调用原生DLL，也会少一点内存泄漏。
vs 全功能云主机相比的优势：

如果是.net4.5，支持websocket长链接程序或异步web程序，可以开一个端口，那么基本这个空间算是“小云主机”，“容器”，可以像集成语言一样，将socket/websocket网络支持部件和cs服务器部件也集进来呢，做成真正的跨bs/cs程序的空间。这样的好处多了去了：1，比如网站需答配一个长链接的chatroom可以不用80网页request/respone方式更易实现，2，为网站准备一个更通用的长链服务器程序构建的文件后端什么的，比如websocket版owncloud之于wordpress，3，，游戏服务器，，，， 10000: balala…

在管理上，虚机大环境可由ISP管理。用户完全不必租云主机。一方面避免了自己搭环境，也减少了维护，及费用。

好了，就说到这。

提供下载：至少数据库你自己安装一个wp4.6.1，自己生成。

wpdotnet.rar



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692516775/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



