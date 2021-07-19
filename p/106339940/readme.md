paasone的创新(2)：separated langsysdemo ecosystem及demo driven debug
=====

__本文关键字：visual demo driven debugging，engitor编程，更好用的qtquick__

接《demo as engine,post as app – passone:engitor+enginx之于通用软件开发/部署/运营/实践教育的创新意义》文，我们说到engitor可视作是enginx针对于语言系统分布化的一个特例，可以按enginx针对通用服务器程序分布化的方式打造成“分布式语言系统”（比如分离出ternoda和language kernel服务端，作成enginx下的子服务组件使之更合理化），我们还说，engitor是个天然的IDE编辑器(除了提供的qtconsole as language shell only还可以开发更多的编辑器服务as enginx组件，甚至强大到类似photoshop或游戏世界编辑器visual editor的那种 -not offline,but ingame editor)。编辑器也就是engitor一词”itor”主要的所指，其对于

1，运营级UGC的代码内容制作
2，语言/演示级隔离（可以将开发生态从偏重语言为中心的语言库/包级下放到以应用为中心的插件/扩展/api级）
3，编辑器带来的另外一点优势：那就是可视化调试。
对于这三者都是重要的，这一切都是不断整合的的结果(engitor整合了编辑器,enginx整合了服务器子组件IO)，然而是合理的整合：综合上正如前文所讲，”基于engitor的paas化”使开发下放，形成一栈结构，可以使所有的APP工作者维护一套套CODEBASE和工作在一套套DEMOBASE上。

前文主要讲的是1，本文重点说说2，3：

engitor之于规避脚本语言越来越大被当成系统语言的怪圈：分开语言生态与应用生态
-----

脚本语言中，讲使用范围最广的，除了java就是py了。。JS也在跨三界兴起(native,web,mobile)，然而它们都走了一条病态的生态之路。那就是应用引擎和语言引擎不分。拿js来说,,对于开发者来说，每一个 package 就是一个 “micro service”，是最小重用单元。大部分的 package 只有几百行代码，甚至有些只有几行代码。这样的重用粒度是在其他社区难以想象的。 ——————- 然而JS将这一切做到了包管理内和社区repos里作为语言库，但其实，类似py，JS npm这种什么问题域的东西都做成库的做法其实也不好。拿js cms开发来说，不需要存在demolvl的wordpress的等价物，因为它一切3rd plugins都作为mircoserive被贡献到了npm，作为语言生态而非demo生态存在。与语言包结合更为紧密些。。

而我们以前说过，问题域扩展会越来越多，一个应用栈系统支持，语言支持层应保持短小，demolvl engine应越来越大。而不是反过来，而脚本只应被宿主偶尔调用，不应形成自己宠大的生态。因为它不宜作为通用面向语言解决通用问题（因此需要配备这么大的包），而应欠入到具体问题，作为demolvl小包或热更层或engitor用户支持modable层。

 
脚本只宜被欠入被一个宿主拥有，围绕一个通用复杂度应用的混合语言选型中往往有二种语言，一系统语言一门脚本语言，不应出现二门宠大系统语言。脚本只应被宿主偶尔调用，不应形成自己宠大的生态。否则就是系统编程语言了，因为它不宜作为通用面向语言解决通用问题（因此需要配备这么大的包），而应欠入到具体问题，作为demolvl小包或热更层或engitor用户支持modable层。
 
在这方面，php比py,js的生态要好。php本身很小。没有一个包含XXX，XXX库的大全发行包，它的扩展也基本来自thin dll wrapped native dlls，lua看来也是标准意义上的脚本语言啊。。因为它不通用。也不须通用。也不宜通用。
作为一个初学者，一门好的工程语言，其实他的唯一门槛是学完了语言就可以开始编程（编码）—或许还要加一个调试支持（设计能力和抽象问题的能力只要不是太复杂大家都会有），语言的类库绝不是你学习一门语言必备的，你不必经过学类库(甚至包括标准库)完成编程学习库只是语言的addons。始终要记住：一门好的工程语言，它应假设初学者和非初学者在面对问题时全是只会语言语法和手头只有api可用的“api kits”。

engitor就提供了一个隔离层，它使任何语言的库分离在这个隔离层之下，向用户明确表示，上面的语言层很thin，而问题层的扩展可以无限fat，qtcling中，只有一种主语言那就是qtcpp，cling出来的部分是附属于具体应用的。qtcling可以任意免binding作系统语言和脚本语言切换的可能（因为它只有一种qtcpp语法），使一切逻辑不致于聚集到脚本层，仅热更和动态部分需要下放。

可视化editor能带来visual 调试
-----

只要有调试，我就能编程，根本无须太依赖语法与问题，调试在编程中的作用大约除了编码就是调试，大约在这里要对应前面那句再加一句:一门好的工程语言，它应假设初学者和非初学者在面对问题时会迅速找到调试工具和调试支持，并假设语言语法和调试是二个最基本的充要门槛。

可是显式的调试支持往往没有工程上的支持，以前是CPP之于汇编，不可视，就是黑乎乎的汇编，现在是js之于WEB PAGE，完成可视化类photoshop，直接整合进语言所属生态层的。非IDE工具。在敏捷编程中，test cases就是为了作test(yet another form of “debugging” in programming engeering domain)作的测试设局 — 它用的是编程的方法，而且都没有一个debug case之说。

engitor使dev进入demolvl时代，engitor整合任意应用为编辑器，由demolvl驱动，就使该应用debugging过程进入了一个真实的demo环境，在其中作debug完全是demo驱动的，就像gui之于visual debug一样，也像chrome的F12调试语义CSS一样，一句话，engitor编程，自带debug cases且无须编程。这才是engitor编程的最大意义。

1dddemobase and 1dddebugbase > 1ddcodebase：一个更强大的微实践选型
-----

我们在前文中说到,paasone为开发建立了一个视具体应用为开发发布生态的“1dddemobase”，但我们真的不应该为一门生来用作DSL的脚本建立一个类似js npm repos宠大的中间层“1ddcodebase”.增加了语言生态无谓的复杂度，这个1ddcodebase只宜是demolvl的codebase.

在前面的选型实践中，我总想维护一个“1ddcodebase”，就像QT那样，包含对语言改造支持，问题库，IDE，本地系统编程，脚本扩展整个生态的支持。（这个生态是我认为拿来支持编程教学和自学较为完备的。为什么必须要加一个native langsys？虽然web,mobile开发已完全不native相关，但因为我们需要涉及到平台相关部分。学习上这二代也有着紧密的承前启后关系不可割裂。），尤其是QTquick采用JS+利用web方案解决通用问题DEBUG无门槛的方式是极好的选型和教学范本（web编程和JS是调试设局最好的实践环境和语言学习环境，微服务和微实践——– 这一切都对应enginx和engitor做的工作） 。然而其中终究依赖了二门语言qt=qpp+js和生态，这就造成了割裂：离开了QT封装的那些：qtquick那些优势就不存在。

基于qtcling的langone可以用来解决这个问题。它就是更强大的qtquick，受enginx和engitor支持，langone结构和一栈web化结合将促成新的微实践大局：

1，它将更加显式化微服务和微实践。2，它更强化1dddemobase而不是codebase，和langsys/app之间的开发生态分区：engitor paasone和langone支持下，语言选型谈化了，app ecosystem demo选型也将同样重要。3，qtcling是一门可以将丰富的现有脚本语言binding进来作统一qtcpp编程的语言，可以复用已有成果。作混合编程。

综上：qtcling+enginx+engitor的组合将会是更好的实践教学选型for beginners。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339940/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



