monosys as 1ddlang语言选型+1ddcodebase实践选型绿色monodevelope集成常见多语言
=====

__本文关键字：.net上 都有什么语言，最后一个支持xp的mono,绿色版monodevelop,绿色xamrin studio,mingwsys vs monosys,gtk#绿色版，让monodevelop在mono下启动，以mono为运行时启动,green mono,绿色打包mono应用免.netfx发布__

接《1ddlang》->《编程语言选型简史》《编程实践选型简史》，这是继1ddlang之后第五种语言方案和实践方案。

.net最大的特色就是提出了clr,继承了从delphi开始鲜明的组件支持到.net一统语言CLR，使之基本上变成了“langone”: ———- 能将任何现行语言免binding纳入开发发布的语言生态系统，且视一切为组件，开发发布一体，源码即组件库，语言服务也是组件。.net支持多种常见语言，如果将它独立出来，很容易得到一种“langone”发布包，如题目所指的那样，可以作为1ddlang,1ddcodebase的一种明确的参考实现。可惜官方的.netfx发布包很紧不易另行定制发布。

而mono作为.net的变体，与.net生态不同的是，它最适合拿来定制和集成，且与.net高度兼容，且有monodevelop,xsp这样的完善工具生态支持，其多种语言如ironpy,ironruby实现都在mono/lib下。就像msyscuione/mingwsys/opt下的一堆语言一样。mingwsys中的全是本地语言如cpy,zend cphp。是一套没有显式化的“langsys” — 实则是分散的，而.net下的这些语言是统一的。

接下来谈如何绿色IDE开始讨论整合mono为独立“langone”的技术 — 我们将得到的结果称为monosys。再来谈具体语言，使之成为just another mingwsys。

绿化monodevelop,使之全程不依赖.net的方法
-----

monodevelop现在叫xamarin了。默认安装的时候需要.net,现在让它从mono运行时下启动，同时绿化xamarin ide。

我需要的是最底兼容.net4的，我选择了能广泛下载到的5.0.1.3，毕竟从5.0起，NuGet Support in Xamarin Studio 5.0（由addin变到了lib/mono），最新的xamarin studio都是依赖msbuild安装的。而这个不需要，是相对来说比较可用且易集成的版本。

再确定要找的mono版本，网上难找到.net与mono的版本对应关系了，这个也要最好最低兼容.net4.0的,我最初选择的是Mono 2.10.8（相当于NET with asp.net 4.0?），官网能下载的mono历史版本名字中gtk指明的是使用的gtk版本，你还得另外安装那个版本的gtk来支持xamarin的运行。为了省事不自己编译，我偏向直接下载，结果发现从Mono 2.10.8起大都以gtksharp2.12.11为基础（这就与上面的IDE选择矛盾了因为它至少要2.12.22），我只能找往下的版本，结果一路下来有好多不提供windows installer版本中的，我最终选择了mono-3.12.0-gtksharp-2.12.26-win32-1，它能满足2.12.22的最低要求。

归纳一下流程：先安装.net4，把mono,gtksharp,monodeveloper先安装一次,中途需要安装vc runtime 2013 12.0.30501，然后拷出文件夹，再卸载掉.net，用mono尝试启动它。

gtk-sharp 2.12.25 最新绿化方法（网上的过时）：

我是放到d:|monodev|GtkSharp|2.12中测试的，注意以上有||的地方千W不要少了一个|。要全部是||：

``` 
Windows Registry Editor Version 5.00
 
[HKEY_LOCAL_MACHINE|SOFTWARE|Xamarin]
[HKEY_LOCAL_MACHINE|SOFTWARE|Xamarin|GtkSharp]
[HKEY_LOCAL_MACHINE|SOFTWARE|Xamarin|GtkSharp|InstallFolder]
@=”D:||monodev||GtkSharp||2.12||”
[HKEY_LOCAL_MACHINE|SOFTWARE|Xamarin|GtkSharp|Version]
@=”2.12.25″
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|AssemblyFoldersEx]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|AssemblyFoldersEx|GtkSharp]
@=”D:||monodev||GtkSharp||2.12||lib||gtk-sharp-2.0″
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|AssemblyFoldersEx|MonoCairo]
@=”D:||monodev||GtkSharp||2.12||lib||Mono.Cairo”
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|AssemblyFoldersEx|MonoPosix]
@=”D:||monodev||GtkSharp||2.12||lib||Mono.Posix”
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|SKUs]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|SKUs|.NETFramework,Version=v4.0]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|SKUs|.NETFramework,Version=v4.0,Profile=Client]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|SKUs|Client]
[HKEY_LOCAL_MACHINE|SOFTWARE|Microsoft|.NETFramework|v4.0.30319|SKUs|Default]
还有：加个环境变量，GTK_BASEPATH = d:|monodev|GtkSharp|2.12|
```

mono绿色调用monodevelop方法：
-----

直接启动会弹出.net找不到，因为已被卸载，参照mono/bin下的ipy.bat等，将ide拷到mono/lib下，并作出如下.bat调用。

```
@echo off
“%~dp0mono.exe” %MONO_OPTIONS% “%~dp0..lib|mono|..|Xamarin Studio|bin|XamarinStudio.exe” %*
```
执行,成功！

我没有深入测试只是验证xamarin能否绿色作一个原型测试。当然不能排除这个绿色的原型还有更多未发现的BUG

一般mono应用绿色
-----

其实monodeveloper是大型的mono应用，一般的mono应用也可通过类似的方法在mono下直接运行。并额外得到精简。

让我们来说一下微软开发环境和.net的变迁：

据说.netfx开源跨平台变成.net core了，从.netfx大包发布模式到社区包管理/包贡献模式，IDE也变成了vs code，从厂商为政到用户为政，除了OS不开源，微软终于开源了它最珍贵的语言套件，这绝非为了拥抱移殖化必须开源，不如说开源其实是微不足道的，其最终正是为了实现.net真正的使命 —- 组件化语言不需要太复杂的语言级整包打包（因此.netcore），需要的是包管理海量的应用组件+用户贡献（因此nuget），而每个应用涉及到的包可能只是特定的几个包（因此不需要附带某个整个一次性发布包） —-见《实践选型简史》结尾应该谈到的demolet engine>langsys as engine但却没谈到的那些，这些在《demoasengine xxx》系列未尾中谈到过。

其实mono可以完成通过mkbundle或精简某个应用不需要的assembly部件，来达到.net core同样的效果（绿色发布.net应用而不需要附带宠大的.netfx托管运行时）。

对于php的支持
-----

上述绿化过程中仅假设要求.net4层次的green mono,也是为了迎合这个green mono将来要整合Phalanger 4的需求，它是php5.4规范。wordpress可以稍作修改在其上运行。

Phalanger完全可以做成跟ironpy,ironruby一样，变成mono/lib下的语言组件。

这是以后的话题了。

下载地址：

monosys.rar




-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692517549/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



