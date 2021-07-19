一种虚拟boot作通用bootloader及一种通用qemu os的设想
=====

__本文关键字：自带bios的boot，自带虚拟BOOT的BIOS,packer下以tc+gcc方式编译Cloverefibooter__

物联和云越来越流行了，OS的融合潮流变得不可逆转，在多场景，一云多端OS方面，比起鸿蒙，fuchsia，pvelinux，libos这些从OS层面努力的方向：虚拟化方向（其中鸿蒙fuschia是重新发明OS而pvelinux这些不是），m1芯片下的macos/ipados/ios融合，更像是从硬件开始作为起点的方向。而从boot虚拟化方向的OS工作，如coreboot，更像是整合方向的融合：不重造OS甚至拉高替代OS的一部分。

在前面《Boot界的”开源os“ : coreboot，及再谈云OS和本地OS统一装机的融合》我们讲到，开源coreboot是固件界的“OS”（libreboot努力使之完全没有闭源成份），linux本身是一个开源，但真正伟大的“非玩具”超规模级现代OS。背后可以没有一个公司，却连接起所有公司和组织，个人真正为它维护，想想西方难于组织起全国民级搞防疫，在现在来看linux的成功也依然是难想象的，还有gnu tools。---- 本文提到的coreboot也是这样一种类似linuxkernel复杂度，由一群来自社会各界的人联合，以松散方式完成但实实在在不可思议发生了的作品。

在《云主机装黑果实践(9)：继续处理云主机上黑果clover preboot问题》中，我们讲到在OSX 10.11.6+xcode7.3.1下从低版本编译clover至r5068，在那文我们讲到，clover的编译很复杂，clover在edk2的框架下被编译，edk2/edksetup.sh被设置为souce到clover的ebuild中起作用。edk2中有一个basetools集，里面建立了一个类似cmake之类的复杂体系，当你调用clover/ebuild.sh时，会分解为类似build --skip-autogen -D USE_LOW_EBDA -p Clover/Clover.dsc  -a X64 -b RELEASE -t GCC49 -n 3 的具体命令，然后调用edk2来编译，调用edk2 basetools中的生成框架逻辑在build/中生成toolchain命名的框架文件夹，如RELEASEXCODE5，然后调用basetools中来编译。

> edk2开源方式代替intel的uefi实现tianocore，coreboot可接入它以payload，，clover使用edk，主要使用edk中的duetpkg和omvf而来的(DUET is designed to boot UEFI over a legacy BIOS. ),而omvf是为支持qemu而存在的。

> 事实上到这里为止，该如何理解clover? 整个edk2你可以理解为可以刷进固件的bootloader，也可以在clover中，配合rEFIt_UEFI被当成booter+firmware的组合工作，这就是一种“自带bios的boot，自带虚拟BOOT的BIOS”，它可以统一云主机实机装机，实现通用虚拟云主机BOOT(虚拟机就应当用虚拟boot)，也可以实现在虚拟boot中集成各种工具，写进实体机固件或仅放在硬盘配合boot在启动时发挥作用，实现诸如《一个设想，在统一bios/uefi firmware，及内存中的firmware中为pebuilder.sh建立不死booter》作不死booter之类的作用，当然，也可以集成轻量虚拟机，实现类似headfirmware之类的作用，还可以集成oskernel，实现linuxboot之类的作用。

注意：不要用在白苹果上使用clover会破坏固件。其它机器不会。

在linux gcc下编译clover r5068
-----

在《云主机装黑果实践(9)》中我们是用xcode5编译clover的，xcode是适配具体osx版本的，因此现在,我们探索在gcc和tinycorelinux下编译clover的方法：

我们首先讨论在osx上cross gcc编译clover的方法，因为r5068在osx上除了xcode还有cross的gcc8,gcc4.8.gcc4.9可用(在/basetools/conf/tools_def.template中定义，它还有一种GCC5支持但是clover的脚本中却没有相关buildgcc5.sh)，，r5068是https://sourceforge.net/p/cloverefiboot/code/5068/在2019-09-04之后的tag，我们配合使用的edk2是edk2-code-stable201908-trunk。由于gcc49适合01.02.2016 it is possible to compile full Clover package only with gcc-4.9，所以对于r5068是过时了，实际上xcode5也没有能完全编译r5068到成功结束我们只是取得了clover就了事。在OSX 10.11.6+xcode7.3.1下cross compile gcc49实际上也不成功。gcc4.8肯定也没得用了，

那么gcc8呢？实测在osx10.15.7上，配合xcode12 command tool cross compile gcc8然后编译r5068是成功的，而且一次成功（中间会有一些python error但不影响总体）到最后，不只是止于生成到clover.efi。

我的方法是先编译出cctools-895.tar.gz，nasm-2.14.02.tar.xz这二个到~/src/opt/local(cctools 895,921。前者在11.06下可通过，后者通不过。在osx10.15上你当然可以尝试921)，然后buildgcc8.sh，成功，最后TOOLCHAIN_DIR=~/src/opt/local ./ebuild.sh -gcc49一定执行cp -R Patches_for_EDK2/* ../进行patch一次(因为,ebuild.sh中的patch代码被注释掉了。)，之后用全新的目录来执行（不能用上一次xcdoe5或patch过后的结果覆盖执行），对的，虽然是gcc8但指定了toolchain_dir环境变量依然可用-gcc49 或-t GCC49参数。如果碰到permission denied，把export TOOLCHAIN_DIR=~/src/opt/local放到ebuild.sh中然后sudo ./ebuild.sh -gcc49。


对于在packer下以tc+gcc方式编译Cloverefibooter这样的实践，下面是我的一个初步尝试。

基于《一个fully retryable的rootbuild packer脚本,从0打造matecloudos(1):实现compiletc tools》中的packer模板和buildtools脚本的开头部分，结合buildgcc8作修改成buildrom：
(clover的buildgcc8.sh是我在packer序列脚本中追求的，“all slient compiling”和“判断连续命令最后一条是否出错”的典范。而且，3,脚本与大文件分开，源码包在外部下载，以及“4.fully retryable”。注意到它还使用了bash的eval。):

```
1，开头部分处理：
TC_TGT:删掉
TC=/mnt/sda1/tmp/tc，由/mnt/sda1/tmp/tc改为/mnt/sda1/tmp
PATH处理：PATH=/usr/local/bin。。。。。（这个路径下有tc compiletc中的bins，及busybox wget,如果不让path /usr/local/bin 优先/usr/bin,会让wget.tcz与busybox wget混淆（后者不支持ssl））
mkdir -p /usr/local/etc/ssl/certs/ (这个ssl fix可以整合进packer模板或iso镜像，wget,curl需要)
su - tc -c 'tce-load -iw -s compiletc wget curl patch rsync python3.6'

然后是从buildgcc8中提出来的部分（这个toolchain用于处理basetools中的C能正常编译从而编译整个clover），作处理：
（注意export与免export定义出的变量的不同，前者是可以进入环境变量的）
export DIR_MAIN=${DIR_MAIN:-$TC/build}
export PREFIX=${PREFIX:-$TC/tools}     (这样就组成了/mnt/sda1/tmp/src,build,tools的合理结构)

export MAKEFLAGS="-j 2" （这个一定要保留，可以缩短至少十几分钟的时间，我们packer模板是设置了双核支持的）
export DOWNLOADPREFIX=${DOWNLOADPREFIX:-http://www.shalol.com/d/src/buildmatecloudos}  （里面会是包没有文件夹层次）

2，然后是主体部分：
download source,update clover的patches到edk2，basetools很娇弱。如果你碰到py multiprocess connect broken pipeline就是fresh的源码没有做好，不够干净。
然后修改Clover/ebuild.sh加入：
SYSNAME=Linux
TOOLCHAIN_DIR=/usr/local

3，最后
在linux下进入Clover直接sudo ./ebuild.sh编译clover，需要用-gcc53或-t GCC53而不是49，因为在tools_def中,49只用于osx或linux中通过cross compile命名为x86_64-clover-linux-gnu的新toolchain。
你可能会遇到all warnings treated as error,修改tools-def中的gcc53相关参数即可
buildTime=$((stopBuildEpoch - startBuildEpoch))改为这个去掉expr。获得执行时间，你可以决定startBuildEpoch的放置位置，以自由决定包含在其中的compile xxx || exit 1统计的函数执行量，以及时间跨度。

4，补充

如果你想重新考虑进buildgcc自己编译toolchain，那么还要加上以下：
.......
export BINUTILS_VERSION=${BINUTILS_VERSION:-binutils-2.32}    (bintuils不改了还是用buildgcc8的吧)
export GCC_VERSION=${GCC_VERSION:-9.2.0}  （8.3.0,9.2.0在osx下都可crosscompile编译成功，在tc11下仅920成功）(gcc依赖库buildtools和buildgcc8都一样) (linux下不需要cctools912仅需要nasm2.14.02)
......
export BUILD=""
export HOST="x86_64"  (接下来有介绍为什么这样设置)
export TARGET="x86_64-pc-linux-gnu"   （这个就代替了上面的TC_TGT，注意这里与原buildgcc8 cross compile用的x86_64-clover-linux-gnu的不同,如果你指定target=x86_64-clover-linux-gnu，只要target名字与build/host不同，虽然都是x64下for linux 64，但gcc编译脚本都将其视为某种cross compile，host = target，但是build不同，叫做crossed native；build = target，但是host不同，叫做crossback。三者都不同的话，叫做Canadian cross。那么32到64或64到 32不算cross compile？我们在前面文章利用hashicorp packer把dbcolinux导出为虚拟机和docker格式(2)其实实践过。）
......

ExtractTarball改为DownloadandExtractTarball ()与前者整合，这样可以下载一个编译一个，不用全部预下载。在local package=${tarball%%.tar*}和tarball="${DIR_DOWNLOADS}/$tarball"之间把原来属于download的逻辑写进来即可，注意到由curl改为了wget:
if [[ ! -f ${DIR_DOWNLOADS}/${tarball} ]]; then
        echo "Status: ${tarball} not found."
        wget --no-check-certificate -qO- ${DOWNLOADPREFIX}/${tarball} > ${DIR_DOWNLOADS}/${tarball} || exit 1
fi
把所有涉及到extract的地方改为新函数名。把#mountRamDisk都注释。放入CompileNasm 2.14.02的函数。放入buildgcc8的CompileLibs函数。

不需要CompileBinutils。在tc11 gcc930上native编译gcc。完全可以像编译普通程序那样单个pass命令native编译的gcc，直接使用native上的binutils，不需要涉及到even native compile binutils。你也可以在编译出新gcc之前或之后native compile新的bintuils，这样系统上有二个bintuils或一个到二个gcc，对于待编译出的新gcc，buildgcc8中的nativecompilebintuils(被注释)是设想先用新编出的gcc再来编译出这个bintuils的。期间需要pathmunge $PREFIX/bin一次 （利用路径下新出的gcc）。所以实际上，nativecompilebinutils为新出的gcc所用（你可能需要pathmunge让gcc用上新binutils），是不必要的。

重点部分GCC_native () ：
local cmd="LD=/usr/local/bin/ld AR=/usr/local/bin/ar RANLIB=/usr/local/bin/ranlib ${GCC_DIR}/configure --build=${BUILD} --host=${HOST} --target=${TARGET} --program-prefix='' --program-suffix='' --prefix='$PREFIX' ......
这里build为空，host=x86_64,不加这个，gcc configure不通过，提示whether we are cross compiling error: cannot run C compiled分析原因应该是它决策不出host参数，不知这是一次native compile还是cross compile，因而不知去哪里找bintuils中的ld，虽然/usr/local/bin就在PATH,ld就在PATH中，虽然gcc/config.guess也会自动检测你的native tripe组参数，但是直接用build=host=target=x86_64-pc-linux-gnu也会提示上述出错信息。
但是直接把bintuils喂给它，可以bypass上面所有这些判断（这是gcc编译脚本的bug?），故host用x86_64,并直接给出命令变量LD=/usr/local/bin/ld AR=/usr/local/bin/ar RANLIB=/usr/local/bin/ranlib，以及target。
GCC_native()中TOOLCHAIN_SDK_DIR相关的逻辑可以完全删除。只是这条ln -f "$PREFIX"/bin/gcc "$PREFIX"/bin/cc最好保留。
--program-prefix='' --program-suffix=''是因为上面ln -f和edk2 basetools的tools_def都指定toolchain为gcc
```


对集进pebuilder的设想,及实现通用虚拟云主机实现装机BOOT，和利用它打造不死boot
-----

我们可以把pebuilder.sh现在的镜像(黑果黑群黑win)都设置成一个单独的rom区，在peubild.sh中的那个preseed中就分区，把这个通用boot放进去。然后所有的OS镜像都放在vda2，打造通用qemu os，把常用OS都qemu云guest os化。这样的一个ROM可以在实机上弄成ISO版本或EXE安装版本或U盘版本供实机使用。

最后，还是那句，注意不要在白果上使用clover这类虚拟固件，因为它会破坏实体白果的固件。

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/109557459/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




