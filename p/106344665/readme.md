一种新的DSL生成和通用语言框架：pypy
=====

__本文关键字：DSL框架和自动化生成工具,pypy as dsl framework and jit framework__

在《bcxszy》 part2中提到，发明各种DSL一直是软件工程模式之一，在那里，我们还一直在找寻某种1ddlang和1dddev方案 --- 更多更好的DSL和统一的语言系统并不矛盾，如《编程语言选型通史》《编程实践选型通史》所讲，问题的根源是不断出现新的问题域要求语言系统足够领域通用，最终要导向到语言选型问题，语言选型其实是一个涉及到编程所有的领域的活动(不光问题还要平台还有考虑人的入阶曲线且能将现有的codebase轻易迁移过来)，而理想的状态是提出一种语言或混合系统language for all，它可以集成一种强大简便的DSL方案，能胜任其它语言能做的事而不带有任何先天缺陷,保持同一生态不断层。关于DSL和这种lang for all的设想，有语言内(langtechs,lib,qtmoc工具链+pme)和语言外(方法论,toolset lvl，各种语言标准emscirpt)的一系列手段。

这些在我以前langsys系列文章中都不断涉及：

>在《发布odoo8》时我们谈到主从语言,lua+c,or py+cpp----这也是传统语言选型的经典标准---也是初级标准，注意到因为大凡脚本语言系统，为了兼顾效率和考虑进通用目的，都是binding c extensions--这也是为新语言快速建库的方法，不过当这类语言这样做的时候，它实际上也在承认它是靠补丁工作的，如果满足于同时使用二门语言，其实这是完全可以的，不过这像极了学会了使用C还要学会汇编一样，这样的转换始终带有历史遗痕和存在断层，仅支持从库级和语言技法级，扩展级去扩展DSL支持，这种语言通常用cffi这样的库支持，这样的语言代表是py,php,etc..

>而.net,java这样的语言系统，它提出了统一后端,语言服务也是运行时和库，可以作为API调用,有DSL支持，即使所有语言可以无缝interspect，且它提倡将原生扩展做进纯粹managed runtime，如c#重写forms而不依赖native forms， 这实际上是要统一clr上的生态使之脱离对本地的依赖，只是将一切放在虚拟机的运行时里，明显效率是个问题。

>在上述二种语言生态充分饱和之后，效率和大语言思维终于被提上日程，业界要做的，只能在上述这二个基础上改造，这首先是给常见的语言如php,py增加JIT，近年来，JIT发展迅速，为了将上述php,py语言发展为真正的通用语言奋斗（再在这上面扩展DSL，这就是上面说的langone+dsl化）。就是jit系统+混合语言系统：

>在前面《发布terracling》时，我们提到从CPP到混合语言系统选型出现的历史现象回顾，且做了一个小型的关于混合语言的综合。联系到更早在《发布qtcling》时我们谈到llvm的jit原理和它独立于传统编译器的事实，这里我们看到LLVM作为一个DSL和JIT工具框架，它的强大实用性，要理解它，可拿它与clr,jvm这样的东西类比，因为它们都支持多语言前端和统一后端且都有JIT，有强大的可比性，然而他们的区别却微小不易发现而至关重要，足以影响到他们归类在不同的流派：

>clr,jvm是虚拟机流派，llvm是运行时流派，llvm后端就是一个to native os的运行时没有VM。它没有为任何语言设置一个解释部件只是一路翻译，最后仅执行jit后的代码，这样的代码已是jitted to native的,这使得它的效率是很高的。一句话，llvm的统一后端和其运行时就是免虚拟机且JIT的没有虚拟机和解释部件，它允许从C系开始制造前端这是它与clr,jvm不一样的地方（后者如果要写C扩展是用虚拟机routing原生代码），它产生的新DSL和语言，可以与原生C语言系统的模块在IR级交互可直接调用这类模块无须binding，且由于jit是类解释系统的在线执行机制，因此可以支持产生qtcling as c++ script这样的语言。 而jvm,clr无非就是虚拟机+解释，而jvm,clr同样有jit，对于中间表示（字节码或AST）和执行结果，他们都提供了一个可写多语言前端为任一语言集成jit的框架，JIT和虚拟机都是黑盒（或者半JIT半解释，或者纯JIT），有没有黑盒这个是没有差异的，--------------- 产生差异的，恰恰是这个黑盒内部是如何运作的：llvm是分析字节码然后以jit方式快速编译且执行.新语言不需要VM运行只须带LLVM运行时，而clr,jvm的jit默认是解释系统加jit协同工作的，任何语言结果必须带虚拟机。llvm是all code defaultly routed to llvm,clr 是一切routed to vm first,then selectively to jit

>纯走JIT，完全可以使二门语言需要混合的部分走统一的工具链流程和开发发布。不必涉及到专门的binding过程。这就维护了统一生态不必断层。而统一后端加统一都走jit，可以使得多语言天然可交互。

>除了qtcling，在前面《发布terracling》时，并提到一种原型terracling as toolkit，qtcling和terralang都是典型的llvm based jit mixable langsys，而terracling更先进，因为它提出了用lua metaprogramming terra(c)产生新语言。使得选型二门中心语言，其它DSL都可以以库的方式被plugin进来，然而其方法主要还是用lua结合编译原理编程产生新的语言parser..

>最后，我们来归纳一下开发界对DSL逻辑和dsl语言发明的所有招数：有interlanguage interopt,binding interface tool,Preprocess,template,meta programming,partical evaluation,src2src translator（甚至到支持全部语言的haxe）,common runtime,anonation,js functional metaprogramming,regluar compiler和编译原理，还有jit+mixable mutiple langsys

如果LLVM是这么好的框架，那么不出所料，在LLVM上直接做PY，PHP的JIT应该会收到好的效果，然而，事实上llvm被尝试用于将很多传统语言如php,py装配新的jit，然而收到的实际效果却不好。google的unshadow和dropbox的pyston都反响不佳， 据说它对前端语言的要求最好是非动态类型的，这次我们碰到了pypy, 它不光是更好的LLVM，且它也面向多语言走JIT没断层，vs terracling它也有metaprogramming+编译原理出新语言系统的能力且以语言内机制自动完成jit部分，没错，它其实是另外一种更强大的langone+DSL框架，单PYPY是语言实现，整个PyPy语言系统就是一个编译器框架,完成可以拿来跟llvm+terracling结合效果相比，与llvm这种忠实地从0开始再造轮子的方法相比,pypy似乎更聪明一点，它重用轮子，它极力促成的结果是：使py真正变得通用化且集成DSL开发机制而能使产生的语言巧妙免除binding c的那些场景，因为它走的是更聪明的jit： 

pypy:更合理的metaprogramming和auto jitted backend,像terracling一样装配了一个语言产生器
-----

在制造DSL和混合语言的手段当中，有一种是语言转换器，就是src2src translator，pypy的原理:1）The RPython translator takes RPython code and converts it to a chosen lower-level language, most commonly C. 2）Unlike Python, RPython is a statically-typed language,,,,

>也即，pypy使用了rpython(rescricted)这门python的子集来自实现，但是要注意rpython其实离python十W八千里了，与其说它是python，其实不如说它是另外一门语言，它就是实现产生DSL的元语言。类似llvm的前端部分，terralang的lua metaprogramming部分。如支持clang实现的那部分。pypy就是用rpython实现的python语言的前端部分和解析部分，虽然rPython不是完整的Python，但用rPython写的这个Python实现却是可以解释完整的Python语言。，，这句话亮了，作为用户我们面向的，始终仅是最后一部分，即可以解释的完整的PY。

>这里的特点在哪里呢？它有三层，即使有这么多层，且全程用py或rpy实现，也丝毫不影响性能。注意这里的1）pypy代码，2）rpython表示，和3）jit运行代码。用户写的是pypy代码，不运行，rpython仅用于工具链表示，产生的c代码才进入到运行，而jit过后才实际运行。离线的始终是那些预置化过程和面向用户代码的部分。运行的始终是jit过后和优化过程的C代码部分和原生代码部分。

>然后其它的事就交给rpython强大的工具链，这是rpython第二部分，这就是我们说到的DSL产生工具框架和JIT产生器，类似LLVM统一后端。亮点是它是一个src2src 转换器，目前Pypy只实现了Python到C的编译，也就是说编译器的后端实现了直接转成了机器码。然后对这部分代码作JIT,它的JIT是jitted to c相当于免VM的native code了，逻辑是：PyPy 的編譯工具鏈可以靜態地對 RPython 代碼做類型推導。類型推導是編譯的步驟中相當重要的一步，做了類型推導之後，就可以靜態地把 RPython 的代碼直接翻譯成 C 代碼然後靜態編譯了。 用户不需要发明JIT，并不需要自己实现JIT编译器，工具链支持自动生成ＪＩＴ，只要按照PyPy框架的指引，用RPython实现一个带有足够annotation的解释器，就自动得到了高性能的带JIT编译器的实现。

>这里的特点又在哪里呢？不可忽视的地方在于， 按需执行的JIT - 对特定的函数做修饰，然后动态的把它们编译成机器码并切换到使用c扩展。这种做法的好处是，重要的事情说三遍，写解释器，得到JIT编译器。写解释器，得到JIT编译器。写解释器，得到JIT编译器。当有人想写一个新的编程语言的实现时，只要在PyPy框架下用RPython编写一个对应上面(2)的语言解释器，就可以借助作为meta-compiler的(3)的部分，得到一个能支持把(1)JIT编译到机器码的高性能实现。

>且它还实现了运行时优化器。

>PyPy's powerful abstractions make it the most flexible Python implementation. It has nearly 200 configuration options, which vary from selecting different garbage collector implementations to altering parameters of various translation optimizations. ----- 这有点类似kernel的busybox配置过程了。

>最后说它的缺点，由于pypy实际上不面向混合来自C语言的扩展，PyPy有很弱的C语言扩展性。它支持C语言扩展，但是比Python本身的速度还慢。但是要说这是PYPY的缺点分明是无理取闹嘛，直接将逻辑写在纯PYPY上，开启JIT就好了。 

pypy:更合理的断层,jitted2c使其跨系统和应用二栈，jitted2js使其跨任意全栈
-----

最后，pypy jitted to c的特点使得pypy可以跨系统和应用二栈，因为传统C系开发发布的那些领域就代表了整个系统开发栈。而其实rpython可以编译到js的，这使得py码代码迁移到web是一个巨大的帮助，可以将整个pypy编译为pypy.js放在浏览器中，如js有asm.js产品，可以将浏览器中的js+css+html通过模板编程控制手段化为py+css+html，这样py就web全栈了。

这是我以前在《发布jupyter》中提出的设想，结合它作为在线IDE，可以完全将所有前后端编码+逻辑调试放在浏览器端。

------------------

拟下文是《web开发发布的极大化：一套以浏览器和paas为中心技术的可视全栈开发调试工具，支持自动适配任何领域demo》，增补《编程实践选型通史》part2 web部分，那文说得不够透彻，预计深入讲解一下，可能自圆其说的往往都会有点玄，抱歉抱歉。。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344665/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



