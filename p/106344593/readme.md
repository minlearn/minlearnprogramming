clingrootsys原理剖析(2):the pme
=====

__本文关键字：cern root,rint,root6 cling,clang cling__

动态语言中的动态类型语言
-----

一般会误以为动态语言就是解释语言。因为解释系统能动态执行代码也往往意味着其被归为动态语言。但实际上动态语言现在最常见的技术形式反而是一种称为“动态类型的动态语言”，它往往依赖前端而不是后端。这造成的结果是：静态语言系统和经典的编译->运行系统也能产生“动态语言”。
比如在编译器实现中，实际上类型系统可以提出元类型，封装有类型的基本信息，然后喂给后端的是元类型/对象产生的子类型/子对象树的形式就可以 – 一个较原来复杂一点的数据结构，然后其它过程保持不变喂给后端。运行期的类型信息照样在运行期可保留甚至动态演变。这难道不是动态语言吗？
（这种逻辑也可以工作在库级和工具链级，即语言系统实现的外部，比如pme，它的实现只要binding就可以了—而binding实际上是另一种编译器意义上的前端翻译，就行了，而执行时是现成的，比如qtmoc为qtcpp源码模式生成的字典，这就是为什么binding也能生成一种动态语言系统，后端执行时可以是静态的，但主要喂给它的是如PEM这样的业已包含类型系统–元类型系统，会将类型系统保持到运行期就可以了）
你可能会为编译过程的这些种种感到迷惑，但实际上这里面所有的技术，跟传统静态编译语言系统 – 你学到的最简单的编译原理实现，是一个外观的。而编译前端，解释前端，binding，这三个词都包含了转译。目标码可以是平台码也可以是中间码，供运行。所有这些，都不能改变所有用编译原理实现的语言系统共享同样的产品外观（都有该有的部分，只是呈现出了不同的形式）。回到系列文章第一篇的文头那些话，用这些通读所有复杂语言系统的定性你才能不致迷糊。

Pme为静态语言模拟了动态语言特征
-----

Pme, poperty,method,event，是对反射机制的一种实现，加了反射机制实际上在静态类型之上加了一门新的语言，和库级运行时，可在运行时查询到整个活动对象树，及每个类的场景图，成员属性。PME是组件的一种通用实现方式。
而且，这种兼具object io特性的pme机制，可为运行时通过外在的编辑器改变objectlvl的程序逻辑提供了可能。借助PME组件的并持久化将成员属性什么的持久到XML等载体。下一次需要时又可加载进来（仅限代码中的成员数据）。
而只有在cling/rootsys这种大环境中，pme与JIT合作，这种动态性才得到最佳发挥，DLL加载终于通过JIT，变成了语言系统的功能。而不再停留在作为操作系统的一种机制，而pme模块可以动态加载，这在开发上体现为，pme DLL体内的逻辑是固定的。可改变的程序逻辑是DLL外的那部分。那个定制脚本部分和你的APP逻辑部分，可以是JIT CPP源码（这里除了PME支持的代码中的成员数据外，整个代码都可持久，The interpreted and JITted C++ shares the same virtual memory space as the app itsel。）。有没有感觉有脚本的样子？直到这里，cling/rootsys开始有了同时能模拟了脚本语言式的解释效果和动态加载效果，可谓叹绝。

Cling/rootsys中的pme字典生成
-----

如果说cling call into raw dll靠的是符号，受JIT和操作系统DLL机制支持，而call into PME模块靠字典信息非符号，动态加载pme组件和发现组件里的OBJ树需要PME支持，因此需要自实现。这是为何呢
这实际上最重要的还是因为jit call into native libs只是使符合变得可见而已。而加载DLL中的资源，是普通的native langsys的功能，于是作为仅仅是执行引擎向OS的传手，llvm也可以而已。但其rootsys libs的pme是库级的，cling代码可以直接call into native libs，但不能call into rootsys libs，因为它们是有pme dicts as bindings的（不能直接通过加载的方式使其为cling可见必须通过对cling的封装变成rootcling才可以）。因此，cling除了jit，和pme，还需要一个手动或自动添加字典binding信息使pme module和普通raw c dll(那种业已解析为简单符号可直接加载的模块)变得一样。的方式，比如一个手动/自动DICT生成器。生成到raw cpp code传给LLVM后端。
带着这些观点，继续来看看cling/rootsys中的对应物，即其对pme模块的支持-aclic。
ACliC只是将pme模块形成加了pme字典的dll的工具。cling is faster building compiled code, but ACLiC can reuse it. Cling产生jit码是高速编译器产生的类解释器效果，而aclic可以在库级反射层面利用它。前面提过，将raw cpp改造成类似qtcpp的新语言系统，所有模块必须经过字典封装，这个过程也称binding。Rootsys即是这样的一门新语言系统。
在实现上，aliac是以patch cling的方式加上去到rootcling的。因此.L ++的方式产生so文件，只用于为pme模块产生dict 模块并链接好。

附：对于qtcling，有mocng，是基于clang的qt moc.exe重实现，这也可以作为cling的patch组件，类似aliac的方式加到qtcling,使之具备发现源码中有pme逻辑即自动生成dict模块的功能，to give it the ability to produce “automic dict generator for qt extending cpp syntaxs” 来完成对整个qt libs的从源码级的重新编译封装，最终完成整个qtcling语言系统的构建。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344593/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



