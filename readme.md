<h1 align="center">
  <br>
  https://gitee.com/minlearn/minlearnprogramming/：minlearn的一云多端云OS/统一编程栈方案。
  <br>
</h1>

<div style="align:center;display:flex;justify-content:center">

<a href="https://blog.csdn.net/minlearn/">
<div style="height:20px;width:132px;border:0px solid #000;background:transparent;margin:2;display:inline-block;overflow:hidden" >
  <div style="float:left;width:31;background:#555;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">blog:</font></div>
  <div style="float:right;width:101;background:#e05d44;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">blog.csdn.net</font></div>
</div>
</a>

<a href="http://zhihu.com/column/c_1277301162652901376">
<div style="height:20px;width:132px;border:0px solid #000;background:transparent;margin:2;display:inline-block;overflow:hidden" >
  <div style="float:left;width:31;background:#555;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">blog:</font></div>
  <div style="float:right;width:101;background:#e05d44;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">zhihu.com</font></div>
</div>
</a>

<a href="http://github.com/minlearn/">
<div style="height:20px;width:132px;border:0px solid #000;background:transparent;margin:2;display:inline-block;overflow:hidden" >
  <div style="float:left;width:31;background:#555;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">proj:</font></div>
  <div style="float:right;width:101;background:#e05d44;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">github.com</font></div>
</div>
</a>

<a href="https://gitee.com/minlearn">
<div style="height:20px;width:132px;border:0px solid #000;background:transparent;margin:2;display:inline-block;overflow:hidden" >
  <div style="float:left;width:31;background:#555;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">proj:</font></div>
  <div style="float:right;width:101;background:#e05d44;color:#FFF;font-size:10px;line-height:20px;font-family:Verdana,Geneva,DejaVu Sans,sans-serif">gitee.com</font></div>
</div>
</a>


</div>

这是一套我2016-2020博客的汇编集和实践库,定位与主题为一云多端云OS/统一编程栈方案，分为minlearnprogramming文档库和minstack演示库

## techdocs

《minlearnprogramming》描述了一云多端云OS/统一编程栈的中心思想和minstack的架构原理和实现，分为2个文档子集：

* [minlearnprogrammingvol1](toc.md/#vol1minstack):《最小化学编程vol1:minstack最小开发栈选型与实训》。vol1描述了minstack的架构,原理和实现。除了架构和原理，还介绍了在云主机上融合各种常见和专用qemu的unix系云OS实践，虚拟融合APP管理面板和IDE面板的实践方案。

minstack的架构和实现见以下demos部分。

* [minlearnprogrammingvol2](toc.md/#vol2matecloud):《最小化学编程vol2:matecloud最小编程学习集与语言开发实训》。vol2部分承接vol1，从建立一套cloude app/matecloudapps开始，把包括nodejs高级语言与appdev实践有机浓缩在10几篇文章中。

## demos

minstackos是一个基于debian,整合了pve和electron开发栈的devops as desk系统，以配合我在《minlearnprogramming》一云多端云OS/统一开发栈的想法。

架构上----- 多场景一云多端OS有多种实现，除了苹果统一芯片华为googlefuchsia统一OS，还有统一boot和libos等虚拟方案:  

> minstack是一套企图将统一OS统一应用容器组成的一云多端OS平台做入boot的方案，这种虚拟机，app容器合一的架构特性保证了虚拟到各os的app共享同样级别的virutal appliance基础  
> 同样集成于boot的Electron web栈则保证了桌面/分布式同型，问题域集成和appmodel，再加上full support的开发/可视调试/发布合一  
> 在app层，把所有electron stack的APP整理成一个OS的应用库形成单栈应用OS，把所有个人可能遇到的开发用基础云APP弄成ide devable和pve backended  
> 以上成全我们最终的统一开发的minstack最简编程实训栈

实现上----- minstackos是多场景一云多端OS多种实现之一：

> 实现特点：采用live机制，整个系统处在readonly文件中，启动时mount进来， 与隔离在文件夹内的数据系统互不污染  
> 直接用原始的linux lxc实现应用沙盒，更易理解，不使用docker这种反使用习惯的只读容器,也不像一些nas发行版本一样使用隐蔽的实现,上提aufs容器抽象至系统统层故容器层可以简单沙盒化。  
> 内置常用devops apps(目前集成了beakerbrowser,code-server，gogs/drone等)和日常网盘应用(目前有cloudreve,magnetw,aria2等)  
> 可用于开发也可用于写作同步等生产。可用于云主机虚拟机本机装机（虚拟机目前只适配了pd16，本机适配了mbp12,1）  
> 强调DEVOPS即桌面即浏览器的沉浸环境(devops as desk)，和日常使用即编程的碎片化应用方式。

minstackos安装方法见接下来demos->livepve和diweb

* livepve和diweb：一个live debian/live pve方案及在线安装/构建镜像方案：方法教程见[这里](/p/livepvediweb/)
* [matecloude/matecloudapp:原生云APP](/p/cloude/)，matecloud is a cloudexplorer/desk environment，默认提供个人常用的一些云APP。

## res

* 🌈 offline pdf edition: [minlearnprogramming](/_build/book.pdf)

---

本项目长期保存,联系作者协助定制minstackos包括不限于机型适配，应用集成等。

![](/p/livepvediweb/logo123zd15sz150.png)




