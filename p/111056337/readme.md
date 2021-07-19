why elmlang:最简最安全的full ola stack的终身webappdev语言选型
=====

__本文关键字：react，stdlib inside lang，前端开发的几种技术方向和潮流，elm editor vs vscode+elmplugin,haskell vs elmlang，
打造类tcent cloudbase的碎片化cloudapp 云原生devdeploy环境和私有小程序平台,把vscode当成dsl trigger editor生成器可配核心，一种可调调试难度与核心裁剪的专业语言与ide。使用elm lang as vistual trigger lang:Time Traveling Debugger,适合webdev的函数式语言或函数式命令式混合语言,elmlang vs py,vs js__

我们知道，站在最高层来看，开发是一种综合考虑“语言，问题，人和设计”的工程行为，而软件作为产出，是一种抽象app栈架构，一般appstack包含有网络，IO，持久，用户界面这些子栈堆叠而成(最后是业务逻辑)，这其中ui是代表，它代表一种appmodel，如desktop appmodel使用本地图形渲染出来的gui,web appmodel使用page ui。

------ 在开发行为中，语言处在中心方案域位置（问题，设计是方案，人，语言是实现域），，而在开发中方案域和目标域的重要特征是它们都会发生变化(软件整个银弹讨论都是关于维护软件生命期变动的讨论)，语言也处在最易变动的中心位置，本身作为变化体串联起这种变化：比如，一般情况下，我们都是用通用语言+库扩展来解决问题的，局限于使用语言本身提供的映射手段来解决新出现的问题，但当语言本身的抽象手段不够用时，我们往往将其DSL化：提出一种新语言或对现存语言综合增强改造，这种语言多变适配开发可以任意方向发展，往往一个价值观，一个硬件特性，对方案和实现领的任何巨细修补行为，都可以成为主导发明一种新语言特性的理由，，不信去看网上的一份《Every Language Fixes Something》and only something，

------ 包括语言在内，我们可以把处理“语言技法问题，抽象问题，映射人的设计需求”这些综合问题在内的开发体做成中间件appframework，它属于解决问题的“一次大而全的设计和实现”，一般封装直到建立appstack为止（给开发者最大关注业务逻辑的时间而不是浪费在appstack本身上），比如qt就是一种框架（qt为cpp增加了moc和event,signal，和gui框架，已经把app变成了qtcpp，因为给cpp增加库级已经不能不能满足qt的需求，所以它干脆改造工具链加入moc处理，对应上面提到的“这是一种对问题人语言的综合处理反映到语言层的处理”，一般框架具体到产生一个APP结构兼实现业逻辑的功能。，qt的库包含了对桌面APP的实现，一起组成了整个qt框架），而高级的框架它超越语言和应用stack，甚至考虑到对OS改造。全程提供full ola（os,langsys,appstack） stack增强或改造，如plan9。（后者叫架构更合适）。

这其中，appstack,appmodel,applang,appframework都是有机联系的，定义APP的UI，即appstack中的ui部分尤为重要，如果说appstack决定一种APP类型，往往也决定开发APP选用的这门语言的选型和框架架构结构，拿上面的qtcpp桌面开发来说，面向对象+消息往往是QTcpp这种语言的ui方案,desktop ui天然就是一个widget组件树存在互动和管理这些互动的组件，这类OO语言如java能很好handle，，事情到了web前端领域，作为最能体现编程技术潮流的前沿地方,框架和语言的变化在这里变动更剧烈，也存在着关于UI开发延伸到语言的变化的类似进程：html页面不是desktop gui那套，而是一种js api视角的dom，js有部分函数特性函数是一级对象，functional特性的js本身能很好处理这种dom，OO反而不必（正如在函数语言中一等函数类型可以轻易表达很多设计模式问题一样，传统的OO命令式语言需要借助接口这类复杂的东西来达成），最初的人们满足于简单的js+html，然后，需求更新了：后来人们想得到更强大的js前端页面，提出了jquery.js扩展操作dom，实现了在一次bs交互中动态更新某节点的功能，这仅仅是往js语言扩展了一个库。到后来，他们想得到universal desktop and mobile/webapp(非universal platform app,非苹果发布了m1芯片和新的arm macbook和big sur 16，真正将ios做到了intel上的APP。)，让web页面做到跟桌面GUI一样强大，以及single page app这类东西的时候，实际上，对页面的动态性和性能优先已经到了一个很高的地步，各种xx.js扩展和不断的ecma标准化已经很难框定这类东西了，人们发现原来流行纯函数式语言时代的某些语言和机制可能才是处理这种逻辑(fullwebstackdev)的最佳方式（js跟纯函数式语言还是有点区别），比如函数语言的FRP可以适配更多 dom ui方面的新需求：因而越来越多的框架应运而生，angular，react ，vue 三大开源框架，他们最大限度解决了前端 页面加载速度 及更新速度问题（稍后会谈到使用virtualdom和diff算法）。当然，也方便书写和开发。于是，当前端可以以统一的frp react方式被解决时，这种手段也被包含在后端形成mvc，形成full webstack级的框架方案。

> 纯函数语言与ml系函数语言：
> 我们知道，冯氏机代码的眼中只有指令和数据，这二者是平等的东西，存在时空可置换原则，冯氏机中的编程语言也有二派。一种是命令式，一种是声明式，函数语言一般属声明式语言，前者主要针对代码中的数据作数据方向的创新抽象，并整合到语言，后者主要是函数式主言这类针对代码结构进行语言发明创新方向的成就
> 同样的道理发生在软件抽象的任何地方正如脚本和数据一个是时间一个是空间，你可以用脚本编码数据本身，也可以把脚本作为数据文件供调用，，（还比如devops把部署自动化了，也是命令脚本化了，资源文件化了，这样就为开发上云准备了条件（因为做成了可被调用的“代码”形式在云上的异地容器运行完成，我们终端只需获得结果），再加上云ide我们就彻底不需要在本地维持一个开发环境。）
> 纯函数语言，偏函数，高阶函数是函数语言的语言级区别于命令式的概念，纯函数的特点是：无状态，无副作用，无关时序，幂等（无论调用多少次，结果相同），这些数学上的东西可以实践成函数语言，但完美表达数学概念的语言机制实际上作为实用工具来使用会显得很原始，这种语言稳健，精美的类型系统能自由安全处理，推导数据类型的正确性和安全性（不过这样也有drawback，函数调用深度debug比较麻烦打印出中间结果比较费劲）。写的时候能调通的代码基本没什么bug，自带并发你完全不用考虑那些诸如死锁，线程池等等的复杂概念，天然适合分布式和容错系统，但学习成本和适用范围比命令式语言要高很多（像Monads和Functors这样的东西很难理解，特别是没有数学背景），还有比如它的代码中不带数据和变量（传数也不便，monads用管道类似的操作来传数），没有循环，只能用作为超集的递归代替。而我们现在使用的命令式则走的是另一个相反的反面，比较亲和大众但是副作用和学习曲线也相当巨大，并发需要涉及到加解锁。
> 命令式这就是我们现在正在流行的C系派生出来的大支，而函数式语言曾经在学术界非常流行，近几年也有一些流行但都偏小众。如一些年前曾经流行的erlang也被选为最不推荐学习的语言之一
> 因此，现在的函数式语言，大部分是混合命令式和函数式，在纯函数语言中加入了一些命令式的成份来稀释它使它变得实用，同时保留函数式语言的大部分优点。保留了函数式针对代码结构而不是数据化方向创新的成果，使得函数语言中的函数变成传统过程式（子过程函数）的合理延伸，，比如haskell和Ocaml就是这样一种ml思想(基于ml 1975年metalanguage的派生)而elmlang就是haskell的子集。

在前面《elmlang：一种编码和可视化调试支持内置的语言系统》和《在群晖docker上装elmlang可视调试编码器ellie》中，我们重点讲到elmlang作为函数语言用于webappdev的一些FRP原理和基础及elmlang的visual debuggable特性，本文则是从elmlang用于fullwebstackdev语言选型方面的细节和合理性角度讲的,即以下几个方面介绍elmlang：

1）。elmlang是一种基于ml系的haskell派生的语言。本身是用正确的语言干正确的事的典范，elmlang在语言期就给webapp保证的很多webapp的专用开发特性，如pure views, referential transparency, immutable data and controlled side effects，所以在扩展级和用户级可以做很少工作就可以写出webapp。注：除了这些，elmlang还提供运行期无错和time可回溯的debug（这些都来自于母体haskell的函数语言特性）。

2）。elmlang是封装了从applang到appframework一条龙的东西。elmlang的核心和标准库包括了开发webapp的必要成份，elmlang现在整体库的体量达到了1.5k。一切都是all at hand，但elm与py这类batteryincluded不同，因为elm不定位于通用语言，所以它是webdev的终身语言。注：elmlang app本身却没有提出一个appstack的web框架之类的东西，它推荐使用elixer+phoinex。ellie正是使用的这一架构。

下面详细解析：

elmlang:用正确的语言干正确的事。webdev = elmlang as functional programming language，vs python,vs rust
------

我们这里在讨论作为方案的语言对应于解决问题的问题域的手段，要知道，语言能完成的事情都一样，因为它们都是图灵完备的，但成本和抽象手段都不一样，人们显然更追求用最简单的语言能提供的最直接的映射手段去表达一个问题的解决：

这种简便性体现之一是代码可以写得更少更直观更符合自然语言：比如py的成功，它用字母代替一些逻辑符合，它用很多语法糖generator造出很多方便的写法。这样下来，一个.py文件显得很清爽，甚至一个没有写过程序的新手，都能在他的意念里推导出这些代码的用意（前提是它熟悉一些C过程式的东西），这是提高语言流行度一个最得分的选项（保证更少的代码是一个重要方面，或者说，python可以用其它语言的经验来“推导”出其灵活的用法。这种灵活正是为了能在python中少写代码，使代码显得简单。主要集在中语句和写法上的创新上。python是一门从核心创新的语言（集中在过程式和单条语句上）。而不是像其它语言一样从外部和范型级别增加新东西扩充功能去创新。）当然，其实py本身是很复杂度的，python的简单是上帝视角下比较了多门语言的不便之后，发现了py其实是一门灵活胶水特性之后的认知开始的（弃它的缺陷不顾，如全局gil，无jit慢）新手学py，还是一样难的（甚至py越是灵活就越显蒙逼）。而且它作为胶水是batteryfullincluded不像lua只配了一个语言核心。。Guido开发Python的初衷：开发者需要一种高级脚本语言，在易用性和功能性之间取得平衡、在处理复杂逻辑时没有Unix shell的限制；能够像C语言那样，全面调用计算机的功能接口，又可以像shell那样，可以轻松的编程。但是python对于shell的包容实际上没有perl好。曾经一度perl在胶水，shell,web方面的表现都很出色，perl -pe ''可以代替sed awk,grep等一系列用到re的地方，而且它的进程管理也能代代替ps等（至于shell的pipeline，perl语言中的pipeline是默认的行为，是以源码级函数或子过程为单位。当然它不是以组件级执行进程获得输入输出为单位，后者perl也有（当然没有shell的那种方便,这句话其实也说明perl的代码中好多符号，其实是优点而不是缺点））,perl是web开发原语内置的语言，因为它的自语言和文本处理和正则很强大这些刚好是web domain的逻辑。但当过程和命令式远远还有深意可挖时，perl追赶OO和webframework让它迷失了方向，让php和python分别蚕食了web和胶水，shell领域。2015 年发布了 Perl6，perl6相对perl5是一门全新的语言，，“Perl 6” 已经被现在称为 Raku 的产品所采用。现在，perl7准备在perl5的基础上原路继续发展。

elmlang它使用了haskell的子集。一开始就定位于简化代码而设计，跟py的目的有异曲同工之妙，elmlang的docs中，有很多函数语言haskell的成份和章节。还有天然的Debuggable属性：它维持了一个Predictable State Container for JS Apps
这使得it easy to trace when, where, why, and how your application's state changed.促成"time-travel debugging", and even send complete error reports to a server.

除此之外它是安全的，它与rust语言安全类型与并发的异曲同工之处：比如error处理和pattern matching(实际上，异常和其他数据结构一样是同质的，不是有特殊语法规则和能力的元素。因此也可以复用其他用于组合和处理数据结构的方法论和工具。elmlang运行期免错，是因为它有函数类型的maybetype，A Maybe type may or may not contain a value, we must handle the case where the value is Nothing)。当然这种脱离语言本质区别所属比较是没有意义的，但它们在维持安全的外观上是很相似的。

最后，当然还是那句话：对于没有任何基础的初学者来说，任何语言都一样绝对难，只是相对不难。elmlang和py就是相对不难的。

elmlang是封装了从applang到appframework一条龙的生态级东西。vs js
------

新语言直接提供了新问题所需要的元素，集成了更简便的表达问题域的方式，即通用语言核心就集成了专用问题域方案而尽量不借用扩展库来完成没有多少其它乱七八糟的东西。“通用语言干啥啥都行，干啥啥干不好”，而DSL或许才是出路（如上，这种DSL不是扩展一下通用语言的库就行了，而是重新设计整个生态或再度框架化一次最优方案）

拿js来说，js的出现和选型本身就是一种DSL化，它是异步IO语言，是web前端的良好选型，浏览器中的js只绑定dom，不是通用语言，后端的nodejs算是通用语言。这二种运行环境中,较其它语言js其实最本质的特点是它的IO。Ryan在发明nodejs时，他评估了很多种高级语言，发现很多语言虽然同时提供了同步IO和异步IO，但是开发人员一旦用了同步IO，他们就再也懒得写异步IO了，所以，最终，Ryan瞄向了JavaScript。因为JavaScript是单线程执行，根本不能进行同步IO操作，所以，JavaScript的这一“缺陷”导致了它只能使用异步IO。（注：不要把异步IO与并发弄混，在单核语言的前提下和限制下，实现并发。用尽单核语言的潜能。JavaScript是单线程执行的，不存在后台语言那种并发执行。一些编程语言/环境只有一个线程。这时如果需要并发，协程是唯一的选择。）nodejs和浏览器中的js加一起，使得js变成了通用语言，elm进一步以js为低阶语言建立stack，它可以mvc（一体webappstack）结构生成客户端可用的js。任何语言，具备能生成js这点的，都可以称为浏览器（上层）语言。"相当于"让浏览器解析这种语言。而且elm内置webfront模块控制html，它定义或模块支持有，相关css和div控制的逻辑。

对于elmlang，如果说js是一种focusing io的子集做进语言核心，那么elm进一步focusing on frp and react patten，elm对前端开发的支持和融入到语言本身是原生植入到语言的。elm架构和生态在js扩展端的对应物是redux全家桶。 redux参照了elm架构，而不是反过来。二者共享很多相同的成份和实现。

> Elm is reactive. Everything in Elm flows through signals. A signal in Elm carries messages over time. For example clicking on a button would send a message over a signal.
You can think of signals to be similar to events in JavaScript, but unlike events, signals are first class citizens in Elm that can be passed around, transformed, filtered and combined.
> 在0.17版本之后(目前2020未是0.19.1) Elm已经用subscription取代了signal

下面简单梳理下React 的工作原理及虚拟dom 和 diff算法

> 与 jq等不同，React 会创建一个虚拟 DOM(virtual DOM)。虚拟dom 说白了 就是真实dom树的一个描述，用js的对象去表示真实dom ，当一个组件中的状态改变时，React 首先会通过 "diffing" 算法来标记虚拟 DOM 中的改变，第二步是调节(reconciliation)，会用 diff 的结果来更新 DOM。所有显示模型数据的 Views 会接收到该事件的通知，继而视图重新渲染。 你无需查找DOM来搜索指定id的元素去手动更新HTML。 — 当模型改变了，视图便会自动变化。------ 这又是一个解放程序员双手让他们关注业务逻辑的框架带来的功劳
> 将Virtual DOM（虚拟Dom）树转换成Actual DOM（真实Dom）树的最少操作的过程，叫作调和。diff算法是调和的具体实现，将O(n^3)复杂度 转化为 O(n)复杂度。diff算法原则：分层同级比较，不跨层比较；相同的组件生成的DOM结构类似；组内的同级节点通过唯一的id进行区分（key）
> 整个框架级别，通过Models进行key-value绑定及custom事件处理,通过Collections提供一套丰富的API用于枚举功能,通过Views来进行事件处理及与现有的Application通过RESTful JSON接口进行交互.
> 纯函数的柯理化可以直接在语言处理和编译期就可以合并多个纯函数（因为输出确定）降低算法复杂度和程序时间。


------

无论如何，elmlang可作为你的终身语言如果你是一个webdever。再也不要去追逐一门通用语言并寄希望于扩展它的库能达到终身适用其它未知未来领域了，因为这好似不可能。
all inside,all at hand : elmlang这种语言因为面向一种业务领域。很容易积累起知识群，经验群等社区。面向search engine也是一种，虽然它要搜索一次。也算某种面向搜索引擎编程。面向XX也罢，因为一切all at hand,searchable，左右逢源，无论什么问题都有参考能独立解决，总是最好的学习，，

未来我们进一步缩小这个webapp的范围将elmlang打造成webapp trigger editor and cloud app editor环境和workshop.



