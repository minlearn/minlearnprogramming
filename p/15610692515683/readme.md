hostguest nativelangsys及uniform cui cross compile system
=====

__本文关键字：windows host targetting at linux,Compile for linux on windows using mingw64,Cross-compiling on Windows for Linux__

在前面《发布msyscuione》中我们谈到cui对于开发机系统装机的重要性 ---- 它基本上就是提供nativedev系统最基础开发和运行时的支持套件，基本是完成一个OS发行版的二大必要部件。在《一个设想：colinux:一体化xaas for osenv and langsys》《免租用云主机将mineportal2做成nas》我们又谈到对于开发和运用来说运用host os和secondary os的那些场景和需求。

那么，对于同时存在二套OS的编译需求，该如何考虑为其选取一套跨host/guest的语言系统呢？比如，就像host os,guest os有32,64的运行时藩离一样（colinux 32/64），编译器也要克服这些。所以就有在二个OS间处理统一编译的需求，这就是cross compile和统一cui套件的需求。。其实，在《在colinux上装openvpn access server》一文中我们也运用过类似的cross build技术。其中包括toolchain的构建（用GCC组合mingw headers and libs，重编译工具链为特定目标版本等等。。）。那里是脚本自己生成，这里我们是一步一步自己搭建。native编译环境toolchain与交叉编译toolchain相比，非常重要的一点区别就是：后者环境往往需要自己手动构建出来，且涉及众多。当然还有其它的问题，等等

而cross compile to 硬件平台+位数+os，是一个三位体的组合，任何一个组合变量变化，对应到这种cross compile方案的现实世界的所有实现品，都是有变化和局限的：如mingw-w64只能由linux到windows,windows下的mingw64只能cross compile到arm，。作为跨OS的编译器mingw，它里面的以前只有mingw32只能编译32位windows程序。，通常地，对于开发是放在host端，还是guesture端好一点，即编译时是host2guesture还是guest2host好呢，我倾向于考虑的的是windows 2 linux，因为host往往是工具和IDE平台，server core as guest负责运行就可以，但是现实的情况却是：host2guest大都没有支持，比如windows 2 linux的mingw64实现往往没有反过来丰富。

在这里，我们选择用二个简单的例子来说明，描述host2guest的mingw64 cross compile toolchain的使用，而其实，读者应该尝试组建自己的toolchain，且使用复杂的开源程序来测试，比如含linux windows portable的大量小库这样linux2windows或反向都可以测试，足够复杂可以验证cross compile的可用性。文章最后还希望提出一个msys2cuione的东西，在《发布msyscuione》中msys里面配备的是基于mingw32的统一CUI套件，有点过时，而现在msys2+mingw64出来了。所以这里方案中的msys2也算是对其的升级。

准备windows上的简单cross compile toolchain环境
-----

一般我是不倾向自己编译的，不说了，先下载http://repo.msys2.org/distrib/i686/msys2-i686-20161025.exe，里面也有一个mingw-w64-cross-gcc 5.3.0-1 (mingw-w64-cross-toolchain mingw-w64-cross)，这是win间互编的，不是我们需要的，mingw64 sourceforge中默认的和第三方编译的大都是targetting win的，但是也有一个文件夹是targetting nonwin的，在https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20NonWin/vityan/进去看是提供windows 2 linux cross compile的，不过版本比较老，这也是为什么要自己编译的原因之一，自己编译的方法可以参照colinux的cross build脚本，也可以参照vityan gcc -v等，不过自己编译据说有好多坑。下面说说其简单用法：

使用绿色版cross compile的简单方法：
-----

解压到任意一个文件夹我解压到的是桌面mingw，系统变量中加入mingw/bin，写一个简单的test.c，就是printf("hello,world!!")之类，gcc test.c --sysroot=d:/desktop/mingw，编译通过，上传到linux，正常运行。不加--sysroot会出现ld.exe cant find libc.so.6等错误，当然也可以把文件夹组织成gcc -v出来的结果/mw64src/built_compiler_lnx64，这样就不用--sysroot了。

但是我发现g++编译程序时老出hidden sysbols with DSO etc..之类的错误就放弃了，因为这似乎是个巨大的坑。下回有时间自己编译了再说。

准备windows上的msys2+cmake+cross compile toolchain环境
-----

在编译复杂的程序时，需要专门的cmake工具它名字中的C就是cross compile，cmake安装目录中share/platform中大量脚本都是默认为流行native compile写的，对于cross compile toolchain，我们需要写专门的toolchain file for cmake，然后，在使用它时，,cd到shadow build目录，cmake 源码目录 -DCMAKE_TOOLCHAIN_FILE=./toolChain.cmake(你的toolchain位置)，基本上，其写法要注意以下几点：

```
# this is required
SET(CMAKE_SYSTEM_NAME Linux)
# specify the cross compiler
SET(CMAKE_C_COMPILER /mw64src/built_compiler_lnx64/bin/gcc)
SET(CMAKE_CXX_COMPILER /mw64src/built_compiler_lnx64/bin/g++)
SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --sysroot=/mw64src/built_compiler_lnx64")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --sysroot=/mw64src/built_compiler_lnx64")
# where is the target environment，这里有二目录，第一目录就是第一小节提到的--sysroot
SET(CMAKE_FIND_ROOT_PATH /mw64src/built_compiler_lnx64 /home/rickk/arm_inst)
# search for programs in the build host directories (not necessary)
SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
# for libraries and headers in the target directories
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

这样就完成了。当然，除非自己能编译好高可用的cross compile toolchain，再花耐心处理好各个可能出现的BUG和大小坑（VS 正常流行的natives编译链来说），和处理针对于你要编译的目标的各种大小情况。才能得到正常的，稍微通用的cross compile方案。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/15610692515683/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



