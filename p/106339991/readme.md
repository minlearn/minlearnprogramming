在tinycolinux上编译jupyter和rootcling组建混合cpp,python学习环境
=====

__本文关键字：升级/枚举tinycorelinux上的gcc，在tinycorelinux上安装python jupyter__

在前面《tinycolinux上编译odoo》中我们谈到python在流行的“one host one guest”学习语言选型组合中是对应于cpp的，还谈到一些混合语言工具，如terralang,rootcling等，见《发布qtcling》和《发布terracling》，技术界二二相对的事物总有惊人的对应：cpp,py组合的cling就相当于lua,c组合的terralang：

事实上该如何评价cling和c++,py的关系呢：要把rootcling当工具而不是语言。它是搭建一个混合C++和PY的语言系统的REPL环境和学习平台的极好工具，但是我们要实际拿来用中心依然是分开了的，独立的二门语言，即C++和PY --- 毕竟C++历史上不是以REPL方式拿来用的，terralang之于lua+c也是一样的道理。

在更早一些的文章中我们提到和发布过《发布engitor》，jupyter只不过是IDE B/S化了，想象那个python idle ide，jupyter pythonkernel notebook本身就是这个IDE的在线化和极大化(它支持更多语言和可渲染HTML等)。只不过，在那里我们还以技术狂想的形式设想了它其它方面的用途：它还可能与服务器运行设施结合，给设计人员或开发人员提供在线支持开发的可能for both maintainer and developer（传统上都是线上运营线下开发），更进一步，它还可以发展成visual builder技术，以实现applevel同时在线运行和可视化编辑的平台，我们称它为appstackx。

可视化的基础概念是以拖拉方式就能使其在一起工作的可复用件，以前是lib reuse，组件就是一些二进制接口透露出来的服务就能成为可复用件的东西，是demo as reuseable software componets当然，脚本语言的组件天然是源码形式的。无论如何，这距我们的理想：tool as framework but not engine又进了一步：它使得中心可复用件的engine变得谈化，用随手能找到的工具来代替，由于工具不准备作复用件进入架构层，所以就谈化了架构的存在降低了学习成本使得软件开发真正意义上变成了组装测试----要知道，为庞大复杂的软件系统划模块定接口是一件多么可怕的事，而一个新手随便找到能工作起来的东西搭个系统可以给他多大的自信和帮助（以后深入学习组件内部）。这叫入阶平滑无欺。

下面，我们在tinycolinux上一步一步建立起这个REPL环境和其jupyter支持（root cling源码中有支持将这个c++ repl kernel为jupyter使用的模块clingkernel和kernel.json文件），这就需要同时在tinycolinux源码编译出rootcling,python等，又涉及到编译最新的cmake，所以不妨看下《在tinycolinux上创建应用》的开头我们为一个全新平台准备gcc toolchain支持的描述 --- 我们这里只升级GCC和GCC里面带的LIBSTDCXX，而会不是GLIBSTDC。

在tinycolinux上编译gcc 4.8.1和cmake
-----

首先，cling会用到新的支持C++11的GCC来编译且会引用到GCC的头文件来运行，所以我们使用在前文一直使用的gcc4.6.1来bootstrap到4.8.1，下载4.8.1的源码http://ftp.tsukuba.wide.ad.jp/software/gcc/releases/gcc-4.8.1/gcc-4.8.1.tar.bz2,在/home/tc下解压它，cd gcc-4.8.1 && ./contrib/download_prerequisites 会把编译用到的库下载解压好，我们想直接覆盖原来的GCC461安装，所以直接 cd .. && mkdir gcc481build && ./configure --enable-checking=release --enable-languages=c,c++ --disable-multilib && sudo make install，由于不带prefix，它配置出来的configure和make之后的结果会默认安装并覆盖GCC461，也会升级libstdcxx.so，这样就完成了我们的目的：在本系统上枚举一个新的高版本的gcc,gcc -v 发现输出4.8.1。

由于编译GCC，PYTHON，和接下来的CLING，可能会产生大量中间文件，所以tinycolinuxhd镜像放大为4G，将新GCC产生的/usr/lib/libstdc++.so改动链接指到/usr/local/libstdc++.so.6.0.18，而非随系统自带的libstdc++.so.6.0.13，否则接下来的CMAKE在./configure过程中会提示找不到c++stdlib 4.3.15，而且，4.x的curl.tcz，expat2.tcz，git.tcz，libssh2.tcz,libssl-0.9.8.tcz,openssl-1.0.0.tcz,sqlite3-dev.tcz全部下好按以前安装tcz的方法安装好，未来都有用。

现在安装升级cmake(在lnmp src中有一个旧版本2.x的cmake)，以前的cling和llvm都是用标准./configure的现在都改用CMAKE了，依然配置安装到默认/usr/目录,我下载的源码是cmake-3.10.1.tar.gz,在/home/tc下解压./configure && sudo make install,cmake -v发现是3.10。

安装在前文《编译odoo》中的python，由于jupyter会用到sqlite3模块，所以安装完sqlite3-dev.tcz重新源码跑一次并安装，（最好重启一次）python的./configure会自动发现sqlite3开发库会生成_sqlite3.pyd之类的支持。

这三大件准备好了就差不多了。

在tinycolinux上编译root cling和配置jupyter支持
-----

跟下载gcc481源码一样，用GIT工具（上面提到要安装tcz）以以下过程分别检出llvm,clang,cling的源码（编译llvm会统一编译clang,cling），我检出是20180115左右前后的版本，为了控制tinycolinuxhd大小，检出后删除根下.git和tools/clang,tools/cling下的.git：

```
git clone http://root.cern.ch/git/llvm.git src

cd src
git checkout cling-patches

cd tools
git clone http://root.cern.ch/git/cling.git
git clone http://root.cern.ch/git/clang.git

cd clang
git checkout cling-patches

cd ../..
```

建立与src并列的clingbuild，执行以下CMAKE配置过程：

cmake -DCMAKE_BUILD_TYPE=MinSizeRel -DCMAKE_INSTALL_PREFIX=/usr/local/cling -DPYTHON_EXECUTABLE=/usr/local/python/bin/python2.7 -DLLVM_TARGETS_TO_BUILD="XCore;X86" -DLLVM_BUILD_LLVM_DYLIB=true -DLLVM_LINK_LLVM_DYLIB=true -DLLVM_BUILD_TOOLS=false -DLLVM_BUILD_EXAMPLES=false -DLLVM_BUILD_TESTS=false -DLLVM_BUILD_DOCS=false ../src

以上cmake配置过程会显示cling未来会引用GCC481的哪些路径下的头文件，如果找不到就直接调用GCC动态调试路径。

编译并安装cmake --build .   ，编译完整个cling会占用大约2G不到，sudo cmake --build . --target install安装，安装也才300多M。

测试一下/usr/local/cling/bin/cling发现是5.0.0版本，现在来开启它源码自带的jupyter支持。首先在python中开启juypter notebook：

sudo /usr/local/python/bin/pip install jupyter，安装完后运行：/usr/local/python/bin/jupyter notebook --ip=0.0.0.0,(有时默认只绑定127.0.0.1)，可以看到python2.7的kernel已在ip:8888下完全正确运行了。

当然，如果嫌老是打全路径太麻烦，可以export PATH=(注意等号左右无空格)$PATH:/usr/local/python/bin。

现在，将cling源码下附带的jupyter支持开启，到/usr/local/cling/lib/jupyter/下，会发现有setup.py和几个文件夹里有kernel.json文件。

首先为python安装clingkernel支持，就是setup.py能做到的：sudo /usr/local/python/bin/python /usr/local/cling/lib/jupyter/setup.py

然后将某一个文件夹里的对应的 kernel.json注入到jupyter，让它知道：sudo /usr/local/python/bin/jupyter kernelspec install /usr/local/cling/lib/jupyter/某kernel.json所在文件夹。

完工，重新开启jupyter notebook会发现可用的c++ repl !!

-------

始终要记得，这是一个混合了python和C++的repl学习环境和工具，缺一不可成就cpp,py这对one host one guest好CP。下面就介绍在tinycolinux上安装terralang吧。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339991/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



