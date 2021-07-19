在tinycolinux上编译pypy和hippyvm
=====

__本文关键字：在tinycolinux上编译pypy和hippyvm,pypy上的php,hippyvm on rpython, hippyvm vs phalanger__

在《发布wordpress on .net》时我们谈到clr上的php实现，即phalanger，在《pypy:一种新的DSL框架》中我们说到pypy才是真正的vmlangsys allinone，因为它走JIT，使来自原生c语言的扩展变得不再必要。在PYPY上就能实现效率和生态全包，这才是不拖泥带水最正统的VM编程语言体系，比CLR，JVM正统多了：就如同汇编之后进入os编程的时代C是作为高一阶语言生成机器码汇编的一样，在新时代VM和脚本时代的混合语言中py与c即是这样的关系，把这个自动化过程做进语言系统的pypy即是这样的大语言思维方案。

>在那里我们还提到，比起clr,jvm，它也具有多语言前端和统一后端，实际上这个统一后端是统一工具（这里并没有一个像CLR一样的统一后端），把rpy当工具set，把其它语言当前端，我们可以在rpy工具链上实现多种语言，且带来更多更好的新功效：比如在《pypy:一种新的DSL框架》末尾我们提到它可以促成py与js的混编，在后端使用PY生成浏览器中中的JS。

>实际上该如何理解py和rpy的关系？rpy是工具，也是语言(静态py子集)，它与py共同作用，py+rpy是作为元语言系统来生成其它语言系统的，py又是这个关系中rpy的metaprogramming lang（实际上就是rpy受py调用而已,相当于terralang中的lua+terra，只不过它们是非C的且兼容的PY语法版本。），因为这二者使用基本一样的语法。所以使发明新语言的过程变得简单，可以使用PY+RPY生成多种前端（虽然多种语言其实地位是平等的，但用于产生新语言时，还是用倾向于用PY，因为它是RPY上的主语言，类CLR上的主C#）。而用它们来生成PYPY时，就等同于说，PY生成了自己（假设我们用cpy+rpython生成pypy，这个pypy跟cpy是兼容的）。整个过程rpython只是工具，并不影响我们得到一个原生的pypy。即生成得到的pypy是最终jitted to c的，其实跟cpy是一样的c based python实现性能上一点不差还较Cpy快。一般说pypy就是pypy实现+rpy工具链。源码和生成结果都是这样。接下来会看到。

而pypy上也是有php实现的，作为例子，我们来介绍pypy的编译，顺便介绍其上多语言 - 一个PHP实现hippyvm。hippyvm也是PyHyp的一部分，PyHyp is a composition of PyPy and HippyVM., a single file can contain multiple fragments of PHP and Python code，当然我们本文主要讲编译，并不会过多涉及到混编的内容。

我的环境是tinycolinux+cpy2.7.14+gcc481+php561

准备工作
-----

由于编译过程会使用到大量内存，官方说大约2.5G内存时间上大约总是会用1.5个小时以上，我使用的是1G云主机，只能时间换空间了，先开启3G交换文件内存，但实测在使用交换文件1.5G左右，编译进程会很慢，形似卡住，实际上也卡住了。换成4G内存的云主机照样开启3G交换内存，才最终通过编译，/tmp下生成的临时文件倒是不大，毕竟，预处理多久都可以，但是会因为内存少而卡住，这个就不能接受了.

按如下在tinycolinux上开启交换内存：

```
sudo dd if=/dev/zero of=/swapfile bs=1024k count=3072 创建大小为3g交换文件
sudo mkswap /swapfile
```

临时开启:sudo swapon /swapfile 

或者做到/etc/fstab中:/swapfile    none   swap   defaults  0   0

除了bootstrap py,编译过程中会用到php-cli，我们分别用这样的参数来编译，记得下载对应缺失的4.x tcz pkgs然后重启生效：

cd Python-2.7.14 && sudo ./configure && sudo make && sudo make install

(以上需expat2,bzip2,libffi,ssl,curses这几个事先安好重启)

cd php-5.6.31 && sudo ./configure --enable-fpm --enable-zip --enable-mbstring --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-zlib --with-gd --with-curl --with-jpeg-dir=/usr/local CFLAGS=-D_FILE_OFFSET_BITS=64 CXXFLAGS=-D_FILE_OFFSET_BITS=64 --enable-opcache --with-openssl -with-openssl-dir=/usr/local/include/openssl && sudo make && sudo make install

(jpeg6在4.x tcz mirror中无对应tcz，需要自行下载jpeg-6b源码以--enable-static --enable-shared configure并编译出，因为hippy编译中会用到php,py的bin和lib，默认在/usr/local下，图方便所以不需加--prefix参数)

添加py支持：cpython:get-pip.py,pycparse,hippyvm src/requires.txt中的东西

然后准备hippy的源码,github/hippyvm/hippyvm，按readme.md检出https://bitbucket.org/pypy/pypy/，形成可用的源码结构，我这里是2018.2.15左右都是最新的源码。注意这里都选取默认branch，不要检出我们上面提到的PyHyp相关的brands，即https://github.com/hippyvm/hippyvm/pypy_bridge，按其readme.md,它对应的pypy在bitbuket的bitbucket.org/softdevteam/pypy-hippy-bridge/，它使用的是它修改了的pypy源码,这个修改的pypybridge也需要修改的bridge的hippyvm/pypy_bridge.

因为不支持prefix且默认是就地生成，所以把整个源码目录移到/usr/local/hippy，处理一下源码，把targetthispy.py移到hippy src根下,然后将hippy目录中的hippy也移到src root中。将goal/targetpypystandalone.py也移到src root下,这样就基本准备妥当了

编译
-----

其实未编译就能运行，称为untranslated，非jit版本。是cpython逻辑，就跟rpy一样，这个比普通的cpy还慢。直接python ./bin或pypyinteractive.py就可以了，而我们要得到的是-Ojit的版本

源码目录中那个rpython就是工具链，在源码中rpy虽然是源码形式，但一直也是可立即待用的工具。，你可以把rpy想象成一堆py工具，用cpy或pypy执行它，会产生C的本地代码（translated），这跟C项目通过makefile产生exe是一个道理只不过这是py的构建系统。且这里是产生编译器和语言套件。

而lib_py,lib_pypy，就是pypy生成后支持的额外平台模块,lib_py是纯py的，lib_pypy是pypy支持的独有模块

好了，先构建pypy。

cd /usr/local/hippy

sudo python ./rpython/bin/rpython --continuation -Ojit targetpypystandalone.py

漫长编译过程结束后（期间因为经常会出错，重新编译不会续编，所以上面 --continuation），最后结束，看到可以分为几个步骤，

annotate,rtype,pyjitpl,backendopt,stackcheckinsertion,database,source,compile,build_cffi

2核4G内存+3G交换内存下，除了pyjitpl和stackcheckinsertion用了约半小时，其它都是十分钟之内，耗时最大的是stackcheckinsertion，

编译好的pypy可以删除rpy，但是最好还是保留，因为根本就不大，接下来会看到。因为更能清希化：pypy就是pypy实现+rpy的事实。

如果不开启jit即不带-Ojit，那么编译好后的pypy实际上就是一个普通pypy解释器，就跟上面untranlated的cpy直接运行一样（非C，且未带jit）甚至更慢。至于rpy，你是在开头和结尾都不必由用户涉及的，只在编译pypy的过程中出现（作为工具链控制产生过程和目标pypy解释器选型），只对采用rpy来发明新语言的用户有意义。

然后用高速的pypy还构建hippy,这个pypy-c就是translated版本且with jit的pypy

sudo pypy-c ./rpython/bin/rpython --continuation -Ojit targetthispy.py

完工，同样是jit的新语言-php！

--------

当然目前这个hippyvm是很初级的，wordpress都运行不了，未来把OC移殖其上，当pypy源码中集成了php或其它语言前端，其实它也完全可当成语言的裁剪器如busybox


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106344673/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




