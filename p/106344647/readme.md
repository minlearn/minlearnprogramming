利用terralang实现terrapp(1):深刻理解其工作原理和方法论
=====

__本文关键字：发明自己的语言，可lua扩展的语言系统，用库发明语言vs用语言发明语言,是toolchain语言也是app生产语言，全生态语言__

整个xaas系列文章中，我们编译组装了自己的dbcolinux，现在我们要发明自己的语言，如果说linux生态允许我们很大自由地定制一个完善的OS，那么terralang类似地，就是一个给我们定制语言系统的工具,它的设计目的之一就是这个，而且经过了专门的去复杂化。我们可以在少量知识和实践储备的前提下，copy-paste式地发明自己的简单语言系统

我们要定制的语言系统是cpp ,在前面关于terralang的文章中《terracling》《terra++》，我们多次提到了这个目的，我们现在要利用terra最小核心实现一个真正可用的cpp —— 其实terra已经是个cpp了，但是它离真正可用的cpp还是有点差距，因为它只定义了最小语言扩展所需用到的核心(稍后谈到它的正交类型和语句设计)，所以严格来说它只是个cplus，而现在我们要在这个基础上造一个接近标准CPP的dsl for terralang，terralang.org的网站中，也有大量相关资料专门提到这个专门课题，这成为我们的所有just enough起点。

与lua接通,更像c，设计语言的语言
-----

在这之前，我们要了解terralang作为元语言工具的这种能力的深层原因，我们按前述文章的做法，称呼 terralang为整个terralangsys，而terra是单个语言：

首先，terra应该被称为terrac（Terra是c系的，terra is c extender towards Lua），但是它却更适合被称为terra/lua的，这是为什么呢，因为terralangsys中，terra被设计成与lua混编，那么terra究竟是发明语言的语言，还是用来被日常使用的语言。都可以，甚至terra当然也可以作为独立语言（A scripting-language with high-performance extensions.....An embedded JIT-compiler for building languages.....A stand-alone low-level language....），但是大部分情况下，它以terra/lua方式被使用。lua中欠入terra---terralang主要用来metaprogramming或写dsl。严格来说，terralang中只有terra,lua二种语言。

>terralang中有二种独立的语言，它们的runtime是并存的，也是可以离线分别使用的。Terra compiler is part of the Lua runtime，Terra executes in a separate environment: Terra code runs independently of the Lua runtime.在lua运行期对应terra的编译期，而terra的运行期是OS的,并不绑定整个terralangsys

>但是在语法上和使用上，terra/lua却是一体的。甚至是紧密联系的，就像一种语言——即terralang。

>terra是用来代替和改进C的，它有自己的类型系统（Terra’s types are similar to C’s），代码系统，和所有一门语言应该有的那些，本来如果C有terra需要的与lua交互的一切，那么C直接就是terra了，但是因为没有，所以对C的增强就演化成了terra，——我们把它称为cplus,在terra中日常编程是lua/terra混编，所以实际上可以视为lua/terra+cplus混编，另一方面，terra也是对lua的扩展，Terra expressions are an extension of the Lua language. The Terra C API extends Lua’s API with a set of Terra-specific functions,一句话，terra使lua像c,使c像lua

>那么，为什么要这么做呢？因为terralang被设计成将C与lua良好交互，inter-operate，它必须要及其简单，一开始就考虑进免binding交互，luajit有ffi可以以简单的方式与C交互，其实luajit ffi对于C的封装已经接近terra了，terra也是采用它的作法，既然Lua/c已经可以，为什么还要出一个terra/lua，这是因为其远远不够。比如它缺少元编程和multiple stage programming那些，所以terra增进和增加了这些，它有哪些方面使得它一面像C，一面又接通lua

>其实现原理和准则是什么呢？诚然多语言体系有极可能优秀的多的方案，但terralang只选取了适合自己的部分。

>它实现了一个预处理器，使得设计期和执行期分离，Separating compile-time and runtime-time environments，这二个期，它们交互的单元非常小 —— 一个terra function或type这使得语言系统高效清希，二个期都有完备的语言系统，却能承接，服务于一体化结果。这个过程是这样的：

>在设计期和编译期，The preprocessor parses the text, building an AST for each Terra function. It then replaces the Terra function text with a call to specialize the Terra function in the local environment. This constructor takes as arguments the parsed AST, as well as a Lua closure that captures the local lexical environment. 使得在语法上，terra/lua是一体的。写出来的.t程序将包含多语言源码，但在编译期shared lexical scope and different execution environments因为词法作用域上，双方的types彼此可见。随时相互转化。这种简化是刻意而为之的：基于lua ffi，它使来自terra的所有语言元素作为lua的值存在，简化了这二门语言在各个期的所有conversation，稍后会谈到。

>在运行期，it will call into an internal library that actually constructs and returns the specialized Terra function. The preprocessed code is then passed to the Lua interpreter to load. Terra code is compiled when a Terra function is typechecked the first time it is run. We use LLVM to compile Terra code since it can JIT-compile its intermediate representation directly to machine code. Clang is used to compile the C code into LLVM and generate Terra function wrappers that will invoke the C code when called.—— 最终在底层，combined terra/lua会是一堆lua和本地码混合的东西，没有terra的任何成份。compiler, generated code, and runtime of a DSL. 最终这三者都被打通形成一体语言。

>这样的努力带来的效果就是——比起其它多语言系统，作为多语言免binding交互的典范，Terra and Lua represent one possible design for a two-language system.而其它元语言，如py的meta object,采用的并非语言控制语言，而是将这一切集成在单语言内，这样不够灵活也不够强大-只有同时一门编译语言一门脚本语言才能自然而然地兼有多语言的优点，比如llvm可以将性能关键部分生成为machine code保证性能。还有.net的clr封装的unmanaged 模块等。无一不带天然缺陷。

下面详细说下其中都有些东西，刚好的可互操作的类型正交系统，元编程能力，和multiple stage->DSL是一条因果路径的。首先是其类型系统。

类型和语句的转化，正交化类型转化设计,type-directed staging approach，和共享词法域
-----

二门语言的交互，要看具体语言类型的是否typechecking属性，作用域，生命期，内存布局这些。

在luajit的ffi时代就有这种思想，这也是lua的设计初衷———lua被设计成与c正交,user defined struct可以是一段C,既然有table，为什么不直接用它表达对象呢，有函数，为什么不能用它表达其它呢，这就是正交。terra对于c类型设计也是正交的，——— 所以，它是一种刚好设计语言的语言。不多不少，刚好正交。比如它的类型与C的那些很对应，比如它还有pointer，struct—它还加入了更多的面向metaprogramming stage programming的方面。比如，built-in tables that make it easy to manage structures like ASTs and graphs, which are frequently used in DSL transformations. 

这种类型是用来作元编程的（至于要不要写DSL那是另外一回事），Terra’s type system is a form of meta-object protocol，与其直接把类型设计为面向app生产，terralang的类型被设计成天然在某个staged compilation流程中被使用，这就是ecotype，它允许程序员—— 这里主要是元编程者或DSL发明者，在类型被具化之前，干预类型的内存布局行为等操作，

首先来看二门语言和各个期的typechecking。我们知道terra是需要的，lua并不需要，为了保证语言的一体化。就需要协调 - between compiler, generated code, and runtime of a DSL关于类型可能带来的问题。In Terra (and reflected in Terra Core), we perform specialization eagerly (as soon as a Terra function or quotation is defined), while we perform typechecking and linking lazily (only when a function is called, or is referred to by another function being called). 这样就可以规避矛盾的发生。

所以，terra的编译期运行期分离，正交化设计的类型系统，都是为了简化，变态简化，只是为了最终使DSL的发明更简单一点。最后的重头戏来了，这就是共享词法域，它是服务于让terra写dsl变态简化的另外一个方面。

下面来说这个共享词法作用域，它最终使分离的东西在语法层形成一门叫terralang的东西。去除了在传统stage programming放置操作符（quotation,escape,etc..）的需求。这里需要结合terralang的生成代码机制和元编程机制讲，我们还没有谈到terralang的生成代码的机制，上面只是粗略提到，总而言之，从语法到最终语言形态上，二种共存的语言要处理的问题不可绕过（between compiler, generated code, and runtime of a DSL）作用域就是一方面，The shared lexical environment makes it possible to organize Terra functions in the Lua environment, and refer to them directly from Terra code without explicit escape expressions. 甚至去除了namespace的调用需要。To further reduce the need for escape expressions, we also treat lookups into nested Lua tables of the form x.id1.id2...idn (where id1...idn are valid entries in nested Lua tables) as if they were escaped. This syntactic sugar allows Terra code to refer to functions organized into Lua tables (e.g., std.malloc), removing the need for an explicit namespace mechanism in Terra. 

对于作用域，在staging的各个阶段和形态上并同时透明地跨越lua,terra，都保持了正交。这样本节开头提到的关于类型的矛盾，就都解决了。而且我们始终要记住：变态简化是设计一开始就maintain的原则。

下面来综合讲述metaprogramming到DSL的原理。

生成式元编程，和DSL发明
-----

其实元编程不一定DSL，我们可以仅使用terralang作meta programming。这也是大部分情况下我们使用terralang的情景。这节标题中的元编程可以放到前一标是，只是terralang的元编程更适合写DSL而已，与后者结合更为增益。

在前面说过，其实无论那种语言，写代码都是扩展语言写“DSL”，库也是语言的扩展。----- 要么面向问题要么面向扩展语言本身要么面向APP，只是terralang使得这种DSL具现化。

多种语言有多种元化代码生成的手段，CPP是基于模板的— 它就是一种编译期的类型特化过程，，还有跨语言元化的，这就是stage programming,在stage programming的范筹里，有一些生成操作符，上面宠统描述terralang的元编程技术中也有提到过。

为动态运行期生成代码的能力。这相当于cpp template programming的多语言版本。虽然terralang是真真实实地使用llvm，而不是同样图灵完备的基于template技术的CPP实现———。但这二者很类似，这2个过程可以类比。
 
前面的类型正交设计和共享词法域已经为terra做了大部分工作。ease这里的复杂度也许只是关于DSL本身的。我们知道，如果一旦涉及到编译原理，就很复杂，这里的复杂度几乎减无可减，因为涉及到数据结构，词法分析，递交等系统编程，但terralang也有自己的方案，它用了一种叫Pratt Parsers的方法,What makes this parsing technique interesting is that the syntax is defined in terms of tokens instead of grammar rules，再加上terralang也会专门性地提供一些造语言的API，极大简化了发明DSL的难度。这些发明出来的dsl作为terra的模块存在。并非欠入terra，而是欠入到lua。

这就是说，为terralang扩展DSL，做法上也很简单，完全去除了需要专门发明词法解析器等语言设施的需求。虽然它使用了编译原理，但是它使用的是更有限更简单的有限集，因为所有语言的区别只是一些对于terralang的入口table。

Terralang的DSL能力尤为珍贵，因为它允许任何人发明语言，terra扩展语言本来就设计成要开放成给任何人使用。


——————

Terra是为c系增加一门scripting的最佳实践。它把c改造为Lua like，也把lua/terra，改造为c like scripting，当然它的主要作用在于发明DSL，—— 以后，我们可以在这个c/terra上发展cpp，除了拥有一个全态全包的语言体系，甚至还能有一个高性能的库Terra’s type reflection allows the creation of class-systems as libraries。

Terra,，the programmable cp with Lua,,just like programmable nginx with Lua,有了terralang，从此什么语言都有了，terralang既是app生产语言，又是toolchain语言。还是shell语言，makefile语言等



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344647/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



