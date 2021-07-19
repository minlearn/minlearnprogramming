elmlang：一种编码和可视化调试支持内置的语言系统
=====

__本文关键字：编码和可视化调试支持内置的语言系统，以浏览器技术化的IDE和WEB APP为中心的可视化程序调试语言系统,让编程和调试装配到浏览器,为每个APP装配一个开发时高级可视debugger支持__

不可否认的是，即使编程语言的技法再“抽象”，库再领域完善，工具再完备（况且工具也不完备，我们稍后会谈到），它们还是没做到尽量对每个人都像。且能适配任何应用，编程依然是专业人士的事。

更高层的“艺术化编程手段”是一种出路，在《bcxszy》part 2中，我们归纳了从工程和艺术层面使编程高级化的手段，比如提出更多语言，即语言DSL化脚本化（针对语言技法的改进或增强也是一种DSL化,pme,etc..），提出更多领域语义（针对领域和库，及适配人的建模过程），提出更多模式（超越语言但能被任何语言都实现的patterns,frameworks)来对接人和人对领域的统一理解，及提出更多工具（可视化编程,rad）这些手段，这些手段间又是相互联系的，比如rad可能跟pme有关以后者为技术实现基础。又都往往需要集中这些，使之能体现到一种高级综合语言系统实现中---因为我们总是最终依赖一门语言为中心的各种选型，开发总是与具体语言和它的生态绑定，因为没有人再倾向于发明轮子。

然而直到现在，编程还是尤为专业化。编程复杂度，专业程序依然如此之高源于一个基本的事实：这是因为业界注重于解决问题为先，怎么复杂怎么来，似乎走了一种过度抽象的道路，治标不治本来的历史遗留复杂度，甚至于上面提到的方方面面：

首先拿语言技法来讲，

>降低编程复杂度需要涉及到众多因素，统一到以语言为中心这首先是为语言提供来自其它语言的优点--这就是DSL化和标准化，是把多种语言集成或打散的过程，（它要解决的矛盾是各种语言本身的技法生态带来的不够用或聚集过于复杂化，比如语言技法是针对写法的抽象，需要分散与集成写法）。

>例子一是多典范语言设计，但它其实是很无力的设计-因为业务逻辑膨胀的速度远超语言系统对接起来要求提供的那些，比如硬件的原子操作都能形化为语言的关键字这只能使得语言似乎太聚集和膨胀，各种语言标准只能将语言实现堆得越来越复杂。将一切堆到库级，用库来设计，也避免不了语言技法级本来就存在的问题，这是因为库属于那个语言的生态，跳出这个生态除非在其它语言中有等价实现才有可能，这依然是分裂主义，我们需要共用一个生态的多种语言。

>然后它们又提出了.net,java共后端多前端这样的东西，然而这不是真正的langinone，除了用了vm有效率问题与与nativedev断层之外。不是说.netfx的多前端不可以分散化各种langtech，而是 --- 它们本来就支离破碎，OO这个东西其实也有问题（它虽然免去了要求人们去理解底层的方式但是仅是复用层面如此---面向被使用者，但它是一种过程式范式的附加而不是替换，这就造出了新东西，要求人们同时理解过程和OO，而OOP三重机制比较繁复），各种DP advanced oop techs，framework只是越来越多，做的决不是减法。它建立在专业的底层上，却要求更专业的人来理解和应用它。那么，有没有一种统一的范式，可以类过程式又能可选地实现为OO呢（后面我们谈到函数式）

>类似多语言系统的观点在我以前的文章中随处可见，针对它我们也提出过混合语言系统设想。

>而工具上，语言的高级化和底层不变又形成了矛盾，因为debug的时候我们从来都是通过在某个编辑器和IDE中，追踪底层的执行frame的，所有现在能看到语言编译或解释实现都是这个套路的，而coding过程中DEBUG过程比重几乎甚至是超过coding的专注于讨论这个其实比讨论OO更有意义（后面我们谈到语言内置高级化的可视调试，且适配到per app）。

>一句话，我们并没有处理好底层的简化工作--对应于我们需要的现实的映射，依然还在一方面极大地依赖计算机的方式来处理建模的事情一方面对抗没有一套统一方案真正可用的困难，这造成了与人的断层。编程是不是一种跟跳舞一样容易的事情，完全取决于编程者专业度需要专业人员处理各种细节。一句话，即使有语言作代理对接领域，即使这个代理不断亲切化，也有人难于越过这个学习门槛。

抽象永远是正确的，但关键是如何去统一和抽象，对于过度设计该尽力避免，决不应该乱统一和抽象，即业界总是造出新东西，而不知道造出可替代的东西。我们需要从多个方面去重新考虑语言和根本上与此为中心的开发设计，提出较《bcxszy》pt2中艺术化编程手段更先进的手段，甚至使编程变得不像编程反而正是唯一出路。elm-lang正是这样的语言系统设计。

下面结合elm-lang来一一说明，每条都对应elm的一个特性和其对于传统过度设计的修正性设计：

首先来看elm-lang是一种什么东西:

elm-lang A delightful language for reliable webapps.Generate JavaScript with great performance and no runtime exceptions.

it use FRP,Functional reactive programming (FRP) is a programming paradigm for reactive programming (asynchronous dataflow programming) using the building blocks of functional programming (e.g. map, reduce, filter).

compiled to js的DSL设计与jsintro特性
-----

在《发布terralang》《发布pypy》这些文章中我们不断提到高级混合语言系统，和可裁剪的语言系统如linux kernel般可裁剪的思想，它们主要是从DSL化，和语言内技法这些方面去抽象语言---往更简单更统一更强大的语言系统方向抽象。

而elm-lang正是这种translier为技术的混合语言系统，它用函数式静态系统haskell生成标准的动态语言js，类pypy静态生成py这种动态语言（haskell,js vs rpy-pypy），只是前者少了JIT而且也不是作为DSL工具链框架存在，排除这些它们是相同的。

rpy最典型的用途是jitted一套语言工具的前端，但实际上，它可以jitted out任意的一段rpy写出来的codesnippter或项目,因为它是通用jitter，，所以在作与js的交互时，我们仅需要compiled codesnippter to js，不需要把整个pypy.js带过去(必须事先让js engine binding到pypy.dll)。而elm-lang也可compiled to js这就与web生态接轨了，统一了WEB全栈开发。elm-lang+它的各种库就是以webapp开发为中心的，因为它具有jsintero因此可用于在服务端生成eml后缀的服务端程序就如同php内嵌js一样,jupyter之于nb一样，所以elm就是一个服务端编程语言---生成html+js页面，而这对客户端也是一样的：在IDE选择上，jsintero特性也为它寄宿于chrome+nodejs提供了条件。

treate oo as paradism pattern but not explicit langtech
-----

elm-lang被设计成用于替代js+各种库如react,redux全家桶，将web开发各种范式由JS+库的生态尽力整合到一门语言elm的langtech上。这主要是因为它的frp特性。

先不说FPR，单就函数式语言本身来说，函数式极其类似C过程式，这也就是为什么JS代码看起来很亲切的原因，是一种能兼容兼顾过程机器抽象和OO人类抽象的机制。且做到了像C一样能成就copywrite式编程for newbies.高层能直接对接webappstack的gui-view,db-model,io-msg，低层能像C过程一样copywrite又通过函数式天然保有OO的功效。而oo is evil 的地方在于它高于函数的class封装显式化了，而这是不需要的， 满布class的一份源码不能成就copywrite式编程 --- 而这是新手入阶的基本范式，而functional的js可以保留这种能力同样可以非常自然和容易地进行OO，不需要涉及到OO和OOP对传统过程式的侵入。elm对接copywrite式的demos修改式编程非常好因为它类似C过程式。

况且，它还有它的可视化的调试器inside支持，这又是一个极为重要的加成：

uniform coding and debuging support inside langsys design：为每个APP装配一个开发时高级可视debugger支持
-----

为每个APP装配一个开发时高级debugger支持，elm-lang从工具的debug层面探求使webapp开发变得变成极简的艺术手段：

debug.elm-lang.org有一个online debug它为每一个APP写一段debug逻辑，这得益于elm-lang的frp+pme（pme来源OO，所谓的响应式编程很多是基于event bus或observer的，但在frp的elm中用了更强大更原生的函数范式界的机制来代替）,这就是所谓的debugging support inside languagesystem，类unittest，就是那种边写代码边额外写测试用测试驱动编程的过程，这里是用DEBUG辅助编码无错。

elm-lang采用的function reactive programming完全得益于语言本身的FPR属性，,,,这个编程范式能带来可交互特性。fpr编程全称为interactive fpr programming，这个名字对支持elm的coding,debuging一体化，即语言内可视调试是本质上的帮助。debugger，且是visual的---当然这是不同于traditional step,step into,step over,stack frame view的低级debug，，它是面向具体app的高级debug，具体来说是hot swap支持，它类似传统QT gui creator的pme。

可视化和与IDE DEBUGER的结合则是PME这种东西的功劳。谈到PME,这是VB和QT这些GUI工具的事，因为它们都是交互相关的(注意交互二字)，所以在这里是共用的，交互DEBUG。 这在debug.elm-lang.org中的《How Elm makes this possible:Purity,Immutability,Functional Reactive Programming》中有述。

这其实也是类war3 we的东西。如果说war3 we采用的是面向war3 game ,gui editor+jass自动生成，那么之于elm-lang per app debug可以面向任何app，它就是debug ide+elm生成的js。

uniform webiz client and ide app ecosystem:让编程和调试装配到浏览器
-----

在《编程实践选型:part编程的最高境界是什么》中我们谈到WEB的极大化和浏览器对webapp开发和浏览的双重支持，但是那里我们并没有说完，WEB是一个在领域逻辑上把开发发布做到极简化的工程样例。W3C主导下的WEB，各种标准而不是工具，使得WEB处于设计泥坑不断提出设计和反设计，比如抛弃了如XHTML这样的东西。所以有时标准不如一套简洁有利工具的支持。

由于elm-lang的第1，3特性，它使得基于elm-lang开发一个webize ide成为现实。这使得WEBAPP的开发真正武装到了牙齿。

远程调试器使得一个编辑器不用跟本地语言绑定，所以使得一个简单的text编辑器也能成为语言开发工具。甚至于一个浏览器加一个插件的方式，如php xdebug+chrome插件。

与elm-lang关联的另一个项目-lighttable(nfw)就是这样做的。它可以实现在一个IDE中同时调试js和php，比如让chrome os中的chrome+nodejs核心同时成为WEB浏览客户端，和真正的多语言可视调试IDE --- 这统统都是web技术。这样WEB开发发布就真正极大化了。

---------------

这篇文章是我《web开发发布的极大化：一套以浏览器和paas为中心技术的可视全栈开发调试工具，支持自动适配任何领域demo》系列中的一篇，这文也是对，下文也许是在tinycolinux编译chrome一文后再源码编译出这样一个ide或《把owncloud当git空间管理项目codesnippters》，因为elm-lang类jupyter使得codesnippter is post，即用编程的方法直接服务于写博客，可谓真正的uniform code and debug，且oc可以当成一个极好的codesnippter project workspace存储基地，好了，结文。

关注我。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344688/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



