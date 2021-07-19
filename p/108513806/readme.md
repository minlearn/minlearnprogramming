rcore,zcore,兼谈fuchsia:一种快速编程教学系统和rust编程语言快速学习项目
=====

__本文关键字：一种快速编程教学系统和编程语言设想,把devops和hypersior集成到os和app,learn rust:the hard way笨方法学rust__

在前面《云APP，virtual appliance：unikernel与微运行时的绝配,统一本地/分布式语言与开发设想》和《一种设想：为linux建立一个微内核,融合OS内核与语言runtime设想》中，我们都讲到“语言机制与OS机制一一对应的运行时”的思想，这些属于平台与语言连接处的联系对于编程学习至关重要，直接影响着你在这个OS上学LANG写APPDEV的曲线，（正如“业务逻辑与APP编程”处的连接一样（《一种开发发布合一，语言问题合一的shell programming式应用开发设想》一文，业务逻辑如果采用尽量接近平台的某种逻辑，也会变得相对容易，这是后话，目前，我们仅关注前半句），这种思想得以最大体现的二个地方，一是发明一个OS，二是学习一个语言。

rcore,zcore：
-----

拿发明OS来说，实际上oskernel作为对裸机编程的典范，也需要极高的抽象，内核的发明是一个重写语言的过程（这种工作却没有被用户空间的语言继续下来），采用什么样的语言来写内核，决定了工程的难易度，甚至直接决定最终设计，(比如windows在kernel中实现了一个对象系统，它是用C模拟的是在发明reimplent c，，如果采用的语言抽象层次高，这类事情成本就低，）比如，采用rust，它抽象高,batteryincluded，写出的应用代码量就小。甚至可以拉高抽象决定设计，做出language based os:一种general kernel design using language-based protection instead of hardware protection，libos。

> rCore是c版本的清华大学教学操作系统uCore(u:micro)的Rust移植版本,它里面的syscalls是兼容linux的，可以运行userspace的linux用户程序，后来，他们参考rust-osdev.com织织的成果，将整个rcore又重写为zcore，以接近fuchsia使用的zircon微内核，（Fuchsia 是谷歌试图使用单一操作系统去统一整个生态圈的一种尝试，类似华为的多场景OS鸿蒙），这种微内核特征在《一种设想：为linux建立一个微内核,融合OS内核与语言runtime设想》已经说过：拉高抽象结构，将被兼容内核放在下层用户进程级，这其实就是wsl1等 os subsystem的套路，也是libos，“一个进程一个OS”的思想，以及language based os。

> 目前为止，它们的代码http://github.com/rcore-os/中，有很多子项目(rboot,rcore-user,etc..)，最重要的当然是http://github.com/rcore-os/zCore，它重写了rcore为微内核zircon-object(包含内存管理和任务管理二部分)，有兼容以前rcore的linux-object和linux-syscalls,linux-loader，还增加了对fuchsia的zircon-object（除内存管理和任务管理的其它部分)和zircon-syscalls,zircon-loader。作为最终结果的zcore，可在unix(linux/macos)下以libos方式运行linux(make rootfs,cargo run --release -p linux-loader /bin/busybox)和fuchsia的程序(cargo run --release -p zircon-loader prebuilt/zircon/x64)。也可bearmetal以独立qemu运行(make image,cd zCore && make run mode=release linux=1,cd zCore && make run mode=release)。如果不好理解，你可以把整个项目成果想象成拥有wsl1 subsystem和win子系统的win10(wsl2改为更按近vm的实现方式)。

> 编译时，rust源码编译跟cpp一样慢，这是真的（不过想象其运行期安全和抽象在运行期0成本就忍了）。下载大量crates时。记得把register设为ustc的源。不过，我编译上述项目时，在osx14下运行cargo run --release -p zircon-loader prebuilt/zircon/x64和时cd zCore && make run mode=release都有失败。

以上都不是重点，这里才是：在重写zcore的过程中，他们发现fuchsia的微内核中的系统跟rust语言的很多机制紧密对应（非精确一一对应）。----- 除了他们的V3 rev文档库https://github.com/rcore-os/rCore-Tutorial中将这些开发过程都详尽写出来了，在https://rucore.gitbook.io/rust-os-docs/也有一篇《Rust语言速成》还提到，快速学rust最好的办法是发明一个rust os，顺便与C的交互也学习，探索高级语言的局限（而这在以前的c语言+linux这类教育中是不太可能的，因为前者抽象不高涵盖不了语言学习的典型场景而后者也不能）。。。

fuchsia其它特点:
----

继《一种设想：为linux建立一个微内核,融合OS内核与语言runtime设想》我们还发现了fuchsia中的其它特色。

zircon内核实现里有个虚拟机管理器：virtual app有时称为guest app，这一再印证，从《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》一直写到这的“碎片化app，microapp，微内核OS，碎片化os，virtualapp”也正是zircon要集成到内核中的方案，记得华为的鸿蒙是“全场景分布式”，而最终，各种沙盒各种层次的虚拟化不就是为了使开发系统和部署APP结合，而且能做到更分布更细粒吗。只有细化才能分布式。

如果说云计算无O也就无APP，或许最接近原生的分布式APP：要算这种micro virtual app吧。----- 而其实统一语言，和统一内核，都是为了让真正的applvl的virtualapp出现。不过业界早有docker这样的暂代方案出现。

第二： Zircon 上的一层名叫 Garnet。 Garnet 包含Xi Core，the Xi Core engine for the Xi text and code editor. 它是Xi文本和代码编辑器的底层引擎，在系统中集成IDE，是为了降低可视调试难度。这接上了我《jupyter:devops tool with visual debug》《elmlang》的“开发要接上系统，集成在系统内部以降低难度，devops与rad结合”那些观点。

第三：zircon的flutter层面。这种用web前端技术完成的app ui，如果有了wasm的加持，可以出现在系统的各个层面，不光是web browser内。如果browser都虚拟化了，碎片化了。那么整个cli都是可以web方式开发的，共享统一appstack开发结构(一种ui决定一种app)。



-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108513806/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




