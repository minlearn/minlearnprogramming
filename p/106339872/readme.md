engitor:基于jupyter,一个一体化的语言,IDE及通用分布式架构环境
=====

__关键字:工具层devops__

很难为jupyter这样的一个东西定性，它最初只是一个增强的python repl环境，后来变成了CS架构并支持了多语言，S为语言kernel，C为notebook,console,qtconsole这样的东西，可以分开部署使用。

>> IPython 3.x was the last monolithic release of IPython, containing the notebook server, qtconsole, etc. As of IPython 4.0, the language-agnostic parts of the project: the notebook format, message protocol, qtconsole, notebook web application, etc. have moved to new projects under the nameJupyter. IPython itself is focused on interactive Python, part of which is providing a Python kernel for Jupyter.
如果想快速尝新，下载windows下Anaconda的py发行版，第一个用ipy4是Anaconda2系列的Anaconda2 2.4.0版本.
我们当然关注的是jupyter system与传统CS程序相比的那些不同点：

首先，它不是应用，而是侧重语言系统。要说它是应用，它也只是“编程教育利器”，“一个多语言在线IDE”，是语言系统方面的应用（so,也是CS应用）

其次，它至少有以下特点,先来说表层的，那些直观可见的东西：

jupyter是一个分布式IDE
-----

1，以语言为后端，客户端接受服务端的执行结果，直接输出执行结果。以页面上的cell为单位。
2，CS二端组成了一个分布式的DEMO SHOW系统。

总之就是IPython,他的一个很大优点就是可以把代码写码过程、运行结果展示合在一起，并持久保存在一个notebook中，并由jupyter支撑这个过完成程。

再来说点深刻一点的：

jupyter可能是一个自带开发发布的分布式devops计算环境
-----

它增强了语言IDE，它是分布式交互开发环境（做成了CS和WEB嘛，大凡与WEB沾边的，应用架构上已属分布式）。

它改变了开发协作方式，人们发布ipynb,就可以共享源文件和执行结果，而不需要下载到自己的机器上利用本地语言系统运行一次。如果这个结果可以直接形成应用(分cell的code block块可以像语言源文件和语言内模块一样组成软件)，这足于给编程界带来一股强劲的创新了。发挥直男不由分说的特点来说简单就2点不用怨我：

第一，它改变了软件协作的方式，使ugc,ugc=user generated content，这里c就是coding或codes,它使W人组件开发做到了线上并直接存管结果。

PS：这什么意思呢？
 
如果github是人们递交静态源码仓库的地方，开发者是以offline的方式参与开发。那么如果有jupyter hub，那么它就是组合正在运行的软件组成更大软件的地方。这句话中隐含了组件这个词，组件是现代语言都有的大头，实际上简单来说就是，demo就是组件,可放置工作的dropin的复用件，能将运行中的程序部件作直接聚合积木搭建的东西，都是“组件”。如果这些组件可在网上直接整合，运行结果也托管。那么它立马可以产生一个“动态github”。如果你的app够小，一个ipynb就够。
这样，用户可在线上直接编程搭APP。因为开发用的语言系统和运行用的环境都在线上，结果也只需要呈现在网上。用户只需要复用ipynb贡献codes这些，作为ugc中的c即可。这对需要用户贡献用代码完成逻辑的社区应用系统或游戏应用大用，它使厂商直接接上第三方扩展者。可以极大快速丰富一个应用生态。

第二，它的可调试特性，使W人组件开发的无门槛性降得最低。因为它是个DEMO effect instant show system.

综合起来，它只是将IDE发展分布式，且其架构和产品定位上也可以作成“动态github”之类的东西而已，能理解到这层已经很不错了。

附下载地址了事(软件取名engitor有engitor=”engine tool editor“的意思因为受jupyter支持的语言系统应该到了toolkit直接搭应用的程度了，是编辑方式生成程序的内容生成工具和演示系统，软件已整合对msyscuione/langsys/qtcling的支持，下载后解压到D盘msyscuione下)：

下载地址见原文


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339872/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



