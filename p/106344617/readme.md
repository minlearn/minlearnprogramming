terracling前端metalangsys后端uniform backend免编程binding生成式语言系统设想
=====

__本文关键字： 用terra打造更科学的js,cpp,用lua+c分离式模拟JS。terra真正的终身语言，terra最接近编译原理的元语言,cling based terra：前后端都可免编程binding生成的元语言体系__

在前面《语言终极选型》《实践终极选型》系列中我们谈过"one for all",即一体化，终身语言的概念，联系到在《编程新手真言》第一部分我们一直在寻找某种lddlang，，比如在整书第二部分我们谈到过最熟悉的CPP，它本身就是一门多范式语言。甚至针对于那些要求更具动态性的类型系统，qt也通过扩展库和工具moc的方式组成了一门qtcpp langsys的小语言。在《JS完全》中我们那里我们谈到过js一门可用于web栈全栈开发的语言甚至进化到H5和mobile,desktop native，通常被称为某种一体化web,mobile,native语言的代表，而且它用函数模拟过程式和OO的方式也是某种“语法”一体化的表现，这此都是语言内部层次的极大化。

而后来我们跳出了单语言单生态的考虑着眼于一些综合语言系统，又有了新的发现，比如在《发布qtcling》时我们提到cling和rootsys，它是llvm based，整合了cpp,c scripting且免binding的一支，是真正实现全C系中一体化的，，在《发布monosys》中我们提到过java,net等统一后端语言，顾名思议它带有一体化语言后端的特征，还有一些利用translator compiler而非独立编译器实现的统一后端往往是针对具体语言的，，像vala这种，还有综合类型像zephir,rust这些，动静态结合带let等，他们都带有超越它们固有领域的极大化整合和改造倾向。

可是细细分析就会发现任何语言体系的极大化（通用化）其实正是它们企图在其内包含各种DSL的过程，在bcxszy part2中提到，发明各种DSL是软件模式之一，自古以来，DSL就是如上提出各种语言内机制或各种脚本语言、新语言/语言体系来完成的，即它们都是DSL技术的子集。

且它们统统都有局限。

比如，CPP是语言内的范型整合，且面向C系单生态的。而QT面向CPP也未免太单生态，其利用pme和type reflection扩展类型系统也隐喻着对它其它的扩展是secondary的事情。而js虽然在语法和问题域都有不俗的整合度，然而它终究是构筑于ECMA单标准和单语言实现上的，qtcling非免没有包括非C系，而直接rootsys也是单生态的，它binding库组成新cling语言体系的能力是巨大的，因为它是先库后binding出来的pyroot等，llvm也有免后端特点，然而cling/rootsys前端只有clang系，monosys它不是免虚拟机的，C#只能统一后端不能有真正的免binding前端生成器。C#有语言内编译服务然而缺少真正的语言内支持本语言开发生成器的特点。转换器往往固定而混合动静态类型语言往往扩展不到其它前端和后端的组合。

总之，他们共同的特征：离一门更合理的语言构造和使用方法的跨度始终跨越太大，或缺陷太明显了，这种“更合理，更整合”的设想就是接下来我希望得到的，我希望有一种 : 不致于破坏现有事实语言多生态的既有事实，又能巧妙地整合这些，还要能以传统发明语言的方式(而普通的像语言内提供类型修饰的机制终究有点捉襟见肘，比如py饰符)能在这个原始层次加以扩展的接口，且能在本语言内完成，形成一对多的，最好一主多宿（相对于主，宿可以动态拔插以扩展）的解决方案。

而以上所有这些语言，这些所有的特点，不能按常规方法，支持真正的元编程和代码自动生成。那么，用现有的方案改造/整合行不行？如果单语言的缺陷总是那么明显，那么或许至少二门语言组成的混合语言是另外一种出路（当然它也要以合理的方式支持尽可能多的扩展支持我上面讲到的合理，最小免修正整合)。

从1ddlang到anylang,从single lang到DSL mixed langsys
-----

归纳一下：一种更为颇为科学的设想要求 --- 我们需要一种真正纳入到支持用户DSL创造的一体化语言体系。。最科学的，我们要保留现有的各种不同运行时，再促成一个真正的可用的统一后端，如colinux as xaas的东西，这里是onelang as langsys。

它至少要是某种统一后端或前端的东西，用户可以以优雅自然的方式来产生新语言，新语言作为这个新语言体系的可拔插部件， 真正允许用户用这二门元语言(as host)整合自己需要的语言作为guest language as language comopent or lib plugin

比如我们的目标至少要是：能用这种语言开发任意zend php等的逻辑，使得一种语言，任意既有无修改后端。能粘起来工作，比如我可使用cling写php的wp程序。

目前最大的整合方案如monosys和llvm based langsys like cling/rootsys是最接近我想法的东西，然而它们往往足够强大没有太多围绕它们的项目，最后我找到了terra：

可以说,在terra下，llvm回归了底层虚拟机的原来意味。它是这些语言的统一后端。

它的3个类比物：用function发明DSL，类js用function创造OO体系，用codegen生成代码，类CPP的模板。vala等等

在我强化过后的terra设想中，利用cling作统一metalang替换lua，负责生成各种具体前端语言。可以使得，lua是host,terra是guest，guest可以扩展的方式meta programming变身多种语言或某语言的复合体。，存在一主一guest二门体系，主可用来metaprogramming，客用来兼容后端，就如colinux一样。下面详述：

terra:a multiple stage langsys that can micic js,cpp,etc..

terra的基本描述：

Terra is a low-level system programming language that is embedded in and meta-programmed by the Lua programming language: We use LLVM to compile Terra code since it can JIT-compile its intermediate representation directly to machine code. To implement backwards compatibility with C, we use Clang,a C front-end that is part of LLVM. Clang is used to compile the C code into LLVM and generate Terra function wrappers that will invoke the C code when called.

最基本的考究，就是lua作为转换器前端，将代码转成terra表示，然后运行terra，因为terra是llvm based的，而转换器是lua based的，所以前后端一个主转换一个主运行，兼有写法上的高效和运行时的效率，

理解路径1)：a dynamic language for controlling the LLVM

整个langys，它利用动态语言的头，本地语言的尾，组成一个混合前后端(初看它比较像c preprocesser+vala translator这样的东西)，其实像llvm这种带了jit又带了中间码，又带了native code gen的东西，可以做到混合前后端部件，这样可以免VM且达到本地码的效率，借且llvm，达到与cling与C模块abi linking的效果(Terra code is JIT compiled to machine code when the function is first called)。terra其实是另外一种cling+clang

理解路径2)：a dynamic language for controlling the LLVM -> using a dynamic language to control code generation of static one

multiple stage programming，它是metaprogramming中code generate中的技术。它在一些数值编程领域非常流行。其本质：

Multi-stage programming (MSP) is a variety of metaprogramming in which compilation is divided into a series of intermediate phases, allowing typesafe run-time code generation.Statically defined types are used to verify that dynamically constructed types are valid and do not violate the type system.

A multi-stage program is one that involves the generation, compilation, and execution of code, all inside the same process

The staged programming of Terra from Lua,,,注意是从terra到lua的staging，这二者的相互欠入性来说，分清二种语言，terra core和full terra langsys，一份具体的用该语言写的代码是terra-lua代码。

因为事实上lua跟C是完全不同的二种语言，它们的interportable终究只是他们的外在属性，内在它们是不可交流的，那么这二者是如何联接起来的呢？技术本质和过程到底如何？这二门语言有共同词法作用域所以就保证了这二门语言无缝交互性（interpreter），极力使得他们像一门语言（中的变量作用域处理部分），除此之外，其它二门语言不同的部分，依然是原本二门独立语言该有的(c/terra和lua有着极广泛的互融合性interopable)。基本上平时你用lua编程(lua)，涉及到control terra to codegen的那部分用c(terra)/lua

理解路径3)：a dynamic language for controlling the LLVM -> using a dynamic language to control code generation of static one -> a low-level system programming language embedded in and meta-programmed by Lua

统一后的terra langsys其实本质只是：它们在metaprogramming这个层次上是结合且统一的。

an important application for MSP is the implementation of domain-specific languages,languages can be implemented in a variety of ways,for examples,a language can be implemented by an interpreter or by a compiler.

we think that having DSLs share the same lexical environment during compilation will open up more opportunities for interoperability between different languages

那么，terra是如何利用lua+c作为元，来生成其它任意中高级语言支持的呢？这是因为lua的数据结构恰好支持重造一门语言所需的那些metaprogramming特性，比如一级类型有function支持，有table支持AST表示，等等，在前面说到js是一种直接可在AST上写程序的语言。

最好的举例是先说js再说CPP

js:

在以前介绍js的时候，我们就谈过functional language就是AST语言，因为它可以直接在语法树上写程序，现在terra，进一步把它清希化了，结合type reflection这一切做到了极限。它可以用函数推导产生各种过程式和OO，从lua模拟C/cpp

cpp:

其实，它也是某种预处理器的极大化，如针对CPP的。完全可以用lua本身来模拟生成更好更统一的预处理，它很像用C写编译器时，这个C是动态的而已。用本语言在本语言的一个实现内写扩展，且加载为库。当然在terra中是lua代码。

还比如，用来实现类CPP的类型系统。

Objectoriented frameworks usually offer a type introspection or reflection capability to discover information about types at run-time. Metaphor allows this reflection system to be incorporated into the language’s staging constructs, thus allowing the generation of code based on the structure of types – a common application for code generation in these environments. 

这也是为什么仅需c+lua，而不是需要是c/cpp+lua，因为CPP整个都可以是被扩展出来的。这比直接在llvm上构筑clang++好，因为我们可以用c+lua的terra来打造架构更科学的terra版cpp

terracling，架构更科学，前端改造为CLING based，后端保持llvm based的terra
-----

那么能不能将terra改造成cling based 呢？即用cling+c替换lua+terra，因为C是支持函数指针为一级类型的。这样做的好处是：直接用C系作metalang控制语言，生成扩展的cpp,py,php等等。

加了metaprogramming特性的cling+llvm，它的前后端都可以免额外编程工作自动生成。比如语言前端的parse等可以binding c dll生成，再对接到后端，库也可以C模块方式集进来，可以直接用zend php或是llvm上的php实现如roadsend php等等

意义：

cling作为脚本语言对生成C代码自动化生成过程非常好，且扩展出来的CPP同属C系，因此metaprogramming可以分散 CPP式将所有范式集中一门语言的特点(比如把c++ template弄成简单的一种语言特性Terra’s type reflection allows the creation of class-systems as libraries.)，这样可以避免QT将PME支持聚集到另外一个QTcpp中去。也可以将CPP预处理以更科学的架构导入，而且可以通过编程和程序内的方法引入，而不是预作为库服务如reflection，也不是作为基础件如编译前端等，而不是像CPP一样杂合到一门复合语言内。

可以直接binding已有程序语言实现，无论是llvm based或llvm non based都可以，只要以c dll存在即可。

还有，其实lua与openresty,gbc这些我前面提出的东西结合紧。整个lua+c揭示了几乎二门必学语言的事实，terra像极了linux的架构，可以类linux一样产生各种封装的变体/新语言系统。且易定制/易自然定制。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344617/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



