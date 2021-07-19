terra++ - 一种中心稳定，可扩展的devops可编程语言系统
=====

__本文关键字：devops可编程的语言系统。programmable language，可编程容器和可编程语言系统,c++ as terra++__

在前面《Terracling:前端metalangsys后端uniformbackend的免binding语言》，我们简单聚焦其语言性质讨论了terralang，主要说到其几个区别性本质：

1,它里面有三种语言c,lua/terra，为什么把后二者放一起？因为使用整个terralang，顶层上还是使用lua来作开发的，terra是配合写被lua调用的函数区块的（我们用terralang指代整个terra语言系统，terra指代三种语言中的一种），这种terra里有可jitted编译运行的c，c通过terra被lua用于meta programmed。新语言terra实际上是multistage中的中间层，即stage1->lua,stage2->terra,stage3->c，terralang能做到这一层主要是因为terra用了llvm+clang，terra用他们构建了terra实现，用他们jitted本地代码，无须binding。所以实际上是clang实现的lowlevel c family语言，且它能lua混编和元编，，主要你还是使用lua，这就像C混编汇编主要使用c只在某些地方需嵌入汇编一样。

2，由于上述机理，它能用lua+terra的方式模拟C++的好多模板语法和复杂语法如预处理，将这些用语言套语言的方式来实现，分散到各种DSL支持文件中terra++，语言用库来扩展的思想在这里得到真正的具现（而实际上C++之父的这个思想在现今的C++实现上越做越复杂），且解决问题的方法使用的是更集成更传统的编译原理方式。不单C++，它甚至能模拟出各种其它基于C的DSL，并将这些lang tech,class libs持久为文件，作为关键字使用。它比cern cling这种更有扩展性，后者只是专注C++，而追赶C++核心的多次变化的cling实际上加大了对C系语言的学习成本，而lua和C都很稳定且语言特性十分接近相通。-----terralang实际只是DSL工具，混编工具+这个工具+这个工具下的二种语言：lua+c,语言还是极其简单和稳定的lua,c核心-----平常并不需要写terra，语言的发明者才需要写terra，我们只使用lua,或c,在发布涉及到terra实现的东西的时候，我们要么在C中内嵌lua，要么在lua中直接调用terra，要么发布纯粹的terra .o,.lib文件，无须binding也不需要嵌入这个几十M的llvm+clang实现不像cling scripting一样只能发布llvmclang本身。。你可以用lua+C写无关terra的直接应用，也可以用lua+terra写可编程的语言扩展，始终围绕着C核心作扩展却用的另外一种语言lua来写应用。

PS：围绕着C核心作扩展却用的另外一种语言lua来写应用这句话揭示了terralang如果能用tinyc代替lua这种就最好了，选择lua是因为lua与c最接近，lua就是c的脚本的精确对应化，tinyc没有类terra的扩展，提供不了metaprogramming c的那些功能，所以改成terracling会更好。

好了，我们来说terralang的另外一个巨大优点，在前面，我们见识过多种复合语言系统，无论它们以什么形式出现，像typescript,一些带let关键字动静态语言结合的语言系统，《elm liveeditor》这些devops语言，一些生成器为核心的multistage语言。他们都少了一些至关重要的特性，工具全集成特性和可编程特性，。terralang刚好都可拿来补足这些。

工具视角看terralang为IDE all in one
-----

在我们讨论过的《elm liveeditor》这种语言中，我们想得到一种基础功能集成化（visual debugger,lang lib, auto build,ide all in one，甚至集成instant demo deploy and play）的东西，这其实就是devops适用语言的那些需求，那么terralang可以是这种语言么?我们说，它实际上是比live elm editor还要集成的工具和语言系统。

这是因为terra用代码形式来控制语言来发明和强化语言。terra允许以更灵活的方式控制语言本身，实现自我控制和强化，，这种可编程的特性允许它在部署过程中生成新语言或提供新的语言设施，那么这有什么好处呢。

如我们见过的语言系统通常都会带一个或复杂或强大或简单的IDE，提供可视为编辑和调试的功能，但这些外围实现始终是工具，terralang本身可被编程，它就可用语言本身作为ide（比如发明一门DSL实现IDE）,这些在terralang中直接有支持。VS elm显式IDE化为工具其发挥不了极佳的灵活性,terra这种非常统一和强大。又由于terra语言本身可被编程，在语言内调试在线调试这样的东西可以统一化成语言内编程手法实现，所以我们完全可用工具视角可以treate terralang as a ide。

而发布上，如上所述，由于我们发布的时候可以按lua或c的方式来任何一者standalone式发布，terralang as a langsys只需作为一个开发时的工具不必发布出去。负责这种功能的是运行时。一些虚拟机语言和面向对象语言更是需要发布巨大的运行时和类库，terra都可以分开发布他们或集成发布都可以，自由度更高。

而整个IDE加运行时的集成，这对强化terralang为devops又有帮助。

视terralang为可持续集成的CI工具,devops可编程的语言系统
-----

可编程的工具或语言体系一体，可语言内通过语言扩展，这些上面都说了，在强调devops的云时代，集成化，可编程的langsys有什么意义呢？，利于CI这怎么说呢？

我们知道docker是一种可编程容器，我们知道docker之所以支持devops，是因为它的编排文件和构建文件可以用shell脚本和yml这种来书写，是programmable的，为什么本地沙盒容器成为不了CI容器。因为它不可编程，不可代码在线构建，作为数据打包和作为程序生成始终是二个不同的过程。所以我们同时需要一种可编程的语言系统，可编程=DEVOPS。

terra即是这样的语言系统。因为它的每个DSL都可以是一个langtechs,一个类，一个问题抽象，一份练习codesnippter，这种可持续集成的性质，可以让任何规模大小的工程都得到持续集成。

-----------------------

在教学上,terra做到了二门稳定的最小语言为核心集，它的DSL特性还能允许你轻易能展开《通过terralang学C++》的课题。langtechs和domainproblems lib,practise codesnippter可以归类到细化的dsl中,是天然的可扩展语言系统的典型，用应用plugin的方式扩展语言。而现在的语言系统，没有一种能达到terra的这种效果(而很多其它用语言发明语言的方式始终停留在库级，或一些有限的关键字和语法级，如python语法糖,js函数直接在语法树上写程序，cpp的预处理和模板元编程特性等。。)这些都太中心化。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344624/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



