云主机装黑果实践(9)：继续处理云主机上黑果clover preboot问题
=====

__本文关键字：clover fake display，unbound variable python_command, manual build compiling Clover from source,EDK2_REV,mac Merge lines between two patterns using sed, gnu sed vs bsd sed，sed同时匹配多个模式1管道符方式2多e选项3分号连接4大括号一次执行,sed同时匹配多个模式异与或模式，sed模式空间匹配多个模式循环内欠结构模式空间__

在前面，我们确定了一切重新开始从clover突破的思路，由于能驱动mojave的最低是4514+，因此，我们最终希望得到一个4514+ 云机上能过preboot阶段的clover(vs chameleon，clover分开了bootstrape和firmware，其核心部件不是boot6,boot7而是cloverx64.efi，clover的boot是已编译好的就在源码包里无须编译得到且不常变，得到编译结果后，我们只须保持boot和前面文章中的镜像文件中的大部分不变，替换cloverx64.efi就可以测试)，能从souceforge clover project files栏直接下载到的clover版本是断续的，实测过preboot的版本是3799其下一个版本3811开始的稍高版本在轻量云机上就报type x64 0D通用保护错误。而且，上述能下载到的二进制版本，其说明文件或执行结果中(clover about menu)除了自身并没有关于任何其它源码组件，比如edk2的rev号，而edk2是clover的一个大件，所以这给我们判断异常发生在那层对应关系的哪个部件中带来了麻烦 ，比如我们无法判断下载到的3799可能并没有使用其发布日最新的edk2,type x64 0d是否主要由cloverx64中的edk2源码部分导致而来。—— (尽管如此，我们在其论坛中https://www.insanelymac.com/forum/topic/304530-clover-change-explanations/发现了一些信息，比如，Rev 3264 Compatibility with new EDK2 18475 (and more?).Rev 3338 Compatibility with recent EDK2, because of changes in Autogen.py build tool.Rev 3471 ，It is now possible to build Clover with -xcode5 using versions of Xcode prior to 7.3 - as long as DevicePathToText.c is patched from Patches_for_EDK2.Rev 3952 New patch_for_edk2 rev 23215+ preventing load efi drivers and applications.With this patch we can use recent EDK2 at least 23447.)。Rev 4331 : 使用25687，REV 4496 : Synchronized Patches_for_EDK2 with r27233,Rev 5068 : Clover at sourceforge is synced to EDK2 tagged as edk2-stable201908 https://github.com/CloverHackyColor/edk2/archive/edk2-stable201908.zip，Rev 5104 Clover switched to C++ programming language

PS：edk2与ovmf与clover的关系。
EDK, EDK II是intel关于uefi实现:tianocore的一个开源实现。clover是作为edk2的一个子组件存在编译出来的，clover用它作虚拟uefi界面与引导osx的主要工作。OVMF "is a project to enable UEFI support for Virtual Machines，面向虚拟机的。clover就是使用它的。BIOS/CSM,EFI,UEFI,现在的UEFI就是它的升级版本，即：tianocore->edk2->ovmf（UEFI对qemu的实现）->clover。clover中有一个rEFIt_UEFI，应该是clover引导osx的中心模块，而edk2是用来提供firmware的中心模块。

clover也是一个十几年的项目了，早期clover的编译默认同步当时最新的edk2，后期clover edk2对应已经时间上脱节很多了。且都移到了github了，只能凭有上述insanelymac上clover change explanations上记载的来确定版本对应关系。最终我们要得到一个接近前文用的5070左右过preboot的编译结果，且修正上述不保留revs对应的版本，只能像当初处理chameleon一样，从源码级编译出一个能在云机上克服preboot就出问题的版本，我们只能亲自动手，手动控制这二个rev关系，从编译中寻求答案。

准备工作
-----

早期clover的编译http://clover-wiki.zetam.org/Development已挂掉，其步骤是把对应版本的cloverefiboot-code-rxxxx改名为Clover放到edk2源码根中，然后在这二套源码中交替运行脚本，先编译全局性的toolchain,mtoc,nasm,gettext,extras等，然后update clover下的 patchs到edk2根，最后在clover下运行编译脚本，带动edk2下的编译脚本。实际结果cloverx64.efi生成在edk2/build/具体toolchain命名的文件夹结果中(我们并不需要全套clover，直到make pkg)。以上这是流程，工具链方面，Currently, 01.02.2016 it is possible to compile full Clover package only with gcc-4.9.Since April 2016 the best compilation is possible with XCODE5 toolset, so no more gcc-4.x needed.Tested on Xcode 4.4.1 and Xcode 7.3.1，Gcc49 toolchain被设定在linux上运行和osx下都可运行，但osx下默认Xcode，我们也使用Xcode。—— 而且我们不使用其它的第三方的整合编译方案build command（micky968,cloveglow,etc..）直接用源码包的。这样方便我们从0开始学习，下面开始：

我们先准备好虚拟机环境，文章1，文章4装的是10.11.4，10.12，文章7用了10.13，由于clover推荐xcode731,这里我们使用10.11.6搭配xcode7.3.1（注意非command tools），从10.11.6开始苹果安装包好像加了厚厚的验证，系统内部也加了sip，往PD中安装时很麻烦，跟文章7一样，果断关PD host网设好时间，一切重新开始，不过100%的时候卡了好久我强制重启，进入系统（如果你还碰到没有符合资源的软件包，安装时需要登录icloud之类，果断一切重来），但是我在10.11.6上安装xcode7.3.1失败，提示Xcode[45010]:  DVTDownloadable: Failed to preflight installation Error Domain=PKInstallErrorDomain Code=102 "The package “MobileDeviceDevelopment.pkg” is untrusted." 又是时间问题，把控制面板中自动设置时间去掉，在客户机断网(使用服务使其不活跃)，Edit and set the date of your Mac as Jan 1st, 2016.或者设置pd客户机与主机时间不同步统统上，最后完成安装xcode。在mac下sudo 拷贝和删除文件时提醒Operation not permitted，无法显示隐藏文件，。是因为Max OS X El 中增加了rootless功能， 即sudo也不能操作部分文件目录， 利用configuration window -> Hardware -> Boot Order，boot into OS X Recovery Mode on Parallels Desktop,Enable Select boot device on startup，重启进入发现这也是一个虚拟EFI之类的东西，csrutil disable即可。

前面测出3811就报异常了，于是这里先从如下3810，22803源码组合开始，从https://sourceforge.net/p/cloverefiboot/code/3800/tarball?path=以及http://sourceforge.net/projects/edk2/22803/tarball?path=）下载源码包。解压到一个文件夹内。使用如下脚本先预处理脚本,这个updatesrcforonce.sh脚本放在下载解压后的edk2 src和clover src组成的文件夹并列的位置，修改下面开头的二个rev号，执行它，正确的结果是没有任何回显，不能多次执行，多次执行破坏源码重新解压并update即可，我都是在host OS X 10.15上执行然后将处理过后源码放进pd osx10.11.6编译：：

```
#下面脚本工作在大约rev 4000之前的脚本
export cloverrev=r3811
export edkrev=r22803

#(处理ebuild,保存revs完善源码缺失)
export reporev="repoRev\=\"0000"
export newreporev="repoRev\=\"$cloverrev,$edkrev"
sudo sed -i "" "s/$reporev/$newreporev/g" cloverefiboot-code-$cloverrev/ebuild.sh
# (处理ebuild,默认使用Xcode，即不使用toolchain仅nasm,与下面的toolchain gcc49二选一)
export nasmprefix='echo[[:space:]]*\"NASM_PREFIX'
export newnasmprefix='export NASM_PREFIX\=\~\/src\/opt\/local\/bin\/ \&\& echo \"NASM_PREFIX'
sudo sed -i "" "s/$nasmprefix/$newnasmprefix/g" cloverefiboot-code-$cloverrev/ebuild.sh
# (处理ebuild,替换toolchain为gcc49)
#export toolchaindir='\"\$CLOVERROOT\"\/..\/..\/toolchain'
#export newtoolchaindir='\~\/src\/opt\/local'
#sudo sed -i "" "s/$toolchaindir/$newtoolchaindir/g" cloverefiboot-code-$cloverrev/ebuild.sh
#（处理ebuild,替换edk2dir）
export edk2dir="cd[[:space:]]*\"\$CLOVERROOT\"\/.."
export newedk2dir="cd \"\$CLOVERROOT\"\/..\/edk2-code-$edkrev-trunk\/"
sudo sed -i "" "s/$edk2dir/$newedk2dir/g" cloverefiboot-code-$cloverrev/ebuild.sh
#(处理ebuild,替换platformfile)
export platformfile='Clover\/Clover.dsc'
export newplatformfile='\$\{CLOVERROOT\}\/Clover.dsc'
sudo sed -i "" "s/$platformfile/$newplatformfile/g" cloverefiboot-code-$cloverrev/ebuild.sh

#(替换patchs引用为edk2根，因为将来要update.sh)
export patchesdir='Clover\/Patches_for_EDK2\/'
sudo sed -i "" "s/$patchesdir//g" {cloverefiboot-code-$cloverrev/Clover.dsc,cloverefiboot-code-$cloverrev/Clover.fdf}
#(替换FLASH_DEFINITION)
export cloverfdffile="Clover\/Clover.fdf"
export newcloverfdffile="..\/cloverefiboot-code-$cloverrev\/Clover.fdf"
sudo sed -i "" "s/$cloverfdffile/$newcloverfdffile/g" cloverefiboot-code-$cloverrev/Clover.dsc
#(替换剩下的Clover\路径引用)
cloverrest="Clover\/"
newcloverrest="..\/cloverefiboot-code-$cloverrev\/"
sudo sed -i "" "s/$cloverrest/$newcloverrest/g" {cloverefiboot-code-$cloverrev/Clover.dsc,cloverefiboot-code-$cloverrev/Clover.fdf}

#替换.inf,(下面命令无回显，要验证结果 需要  grep -rl "被替换的字符串" *，如果没有结果了,说明替换成功了，后来执行时如果build.py... : error xxx tbuild error，也是这里没替换成功，好好检查)
export cloverpkgfile="Clover\/CloverPkg"
export newcloverpkgfile="..\/cloverefiboot-code-$cloverrev\/CloverPkg"
export grubfsdir="Clover\/GrubFS"
export newgrubfsdir="..\/cloverefiboot-code-$cloverrev\/GrubFS"
export fsinjectdir='Clover\/FSInject'
export newfsinjectdir="..\/cloverefiboot-code-$cloverrev\/FSInject"
sudo find . -name "*.inf"|xargs grep -rl "Clover/CloverPkg"|xargs sed -i "" "s/$cloverpkgfile/$newcloverpkgfile/g"
sudo find . -name "*.inf"|xargs grep -rl "Clover/GrubFS"|xargs sed -i "" "s/$grubfsdir/$newgrubfsdir/g"
sudo find . -name "*.inf"|xargs grep -rl "Clover/FSInject"|xargs sed -i "" "s/$fsinjectdir/$newfsinjectdir/g"
#替换.inf(for高级一点的版本)
export grubfsdir2="Clover\/FileSystems\/GrubFS"
export newgrubfsdir2="..\/cloverefiboot-code-$cloverrev\/FileSystems\/GrubFS"
sudo find . -name "*.inf"|xargs grep -rl "Clover/FileSystems/GrubFS"|xargs sed -i "" "s/$grubfsdir2/$newgrubfsdir2/g"

#修改-Werror(出错时使用)
#export werror='-Werror[[:space:]]*'
#sudo grep -rl "\-Werror" *|xargs sed -i "" "s/$werror//g"

sudo mv edk2-code-$edkrev-trunk/edk2/* edk2-code-$edkrev-trunk/
sudo rm -rf edk2-code-$edkrev-trunk/edk2
#如果手动在Finder中替换，要选择合并而不是替换。
sudo cp -R -f cloverefiboot-code-$cloverrev/Patches_for_EDK2/* edk2-code-$edkrev-trunk/
sudo rm -rf cloverefiboot-code-$cloverrev/Patches_for_EDK2
```

新建一个download放到与二个src root和update.sh并列，(离线下载好nasm-2.12.02.tar.bz2，修改buildnasm中的export DIR_DOWNLOADS=${DIR_DOWNLOADS:-$MAIN_TOOLS/download}为export DIR_DOWNLOADS=${DIR_DOWNLOADS:-$PWD/download}，在PD osx10.11.6中编译出nasm。

最好clover src下ebuild.sh，如下这几行注释掉：# Bash options set -e # errexit set -u # Blow on unbound variable(不去掉，高版本edk2会出现unbound variable python_command),下面就可以开始了。

编译测试一个直到rev 5068不报异常的cloverx64.efi
-----

在PD osx10.11.6中cd到src root/clover src root，sudo rm -rf /usr/local/bin/mtoc,mtoc.NEW，sudo ./ebuild.sh cleanall sudo ebuild.sh(-mc不要了，因为我们的测试环境中cdboot就是-mc的,-mc是使用boot7，相当于ebuild.sh -h出来的-D USE_BIOS_BLOCKIO=1，默认没使用-gcc49，默认Xcode5，第一次执行不用执行cleanall，实际这个cleanall是个鸡肋，不要用)

ebuild.sh这是一个企图涵盖linux/osx的脚本，在linux和osx下都可编译使用gcc49 toolchain，(-h查看其参数，也存在一个对最终结果的配置过程)，执行后，clover源码作为一个组件被转到edk2编译过程，在那里会Initializing workspace，根据Clover src root/Clover.dsc(ebuild.sh默认-p Clover.dsc)定义的配置文件转到edk2 srcroot/build/Conf开始编译，最后生成结果cloverx.efi或clover64.efi到edk2 srcroot/build/toolchain named dir下:

如果出现tbuild执行错误就是上面准备工作的脚本没成功替换或重复替换了且preboot通过(通过tcpe上传cloverx64.efi到云机，测试,一般一个版本测试三次，因为有时能preboot的都偶尔会异常，跟chamleon尿性一样，这时直接重启服务器,冷启而不是从tcpe中reboot热启)，

3810 2016-10-14，edk 22803,通过编译和preboot，3811 2016-10-14，edk 22803，通过编译和preboot，利用半分思路测试，成功时大胆跨100，1000个rev，不成功时跟据最近一个成功视版本跨度取一半继续测试：3900，2016-11-02, edk 23078，没编译通过，提示 implicit declaration of function 'ARRAY_SIZE' is invalid in C99，看编译出错提示，在对应的文件中，把#define ARRAY_SIZE(x) (sizeof(x) / sizeof((x)[0])) 加进去(从22943开始)，编译通过，preboot也通过,4000 2017-02-06,23837，2.4k第一个版本,异常,3950 2016-12-01,23452，异常,3925 2016-11-17，23297，异常,3915 2016-11-09,23212，异常,3910 2016-11-07,23098，异常,3905 23091，异常

分析3901-3905，23079-23091异常，考虑主要是edk2 rev的变化导致的，现在用3901 23079-23091来测试,23088通过。23089通过。23090 通过,直到23091有了下面的变化。
https://sourceforge.net/p/edk2/code/23091/log/?path=：by edk2buildsystem:
MdePkg/BaseMemoryLib*: check for zero length in ZeroMem ()
Unlike other string functions in this library, ZeroMem () does not
return early when the length of the input buffer is 0. So add the
same to ZeroMem () as well, for all implementations of BaseMemoryLib
living under MdePkg/
This fixes an issue with the ARM implementation of BaseMemoryLibOPtDxe,
whose InternalMemZeroMem code does not expect a length of 0, and always
writes at least a single byte.

按上面的log，revert changes since edk r23091即可。我们发现3901-3906,23091可以通过preboot了，本地虚拟机也可。

继续从clover开始,往前几个版本又不行了，又黑屏了
3906 2016-11-07,23098 通过,但是preboot会重启
3907 2016-11-07,23098 ,23098 编译未通过,提示error: use of undeclared identifier 'CHAR_NULL’,，从这个版本23089更改了CHAR_NULL的定义位置。把提示找不到符号的源码加进去如下：

```
#define CHAR_NULL             0x0000
#define CHAR_BACKSPACE        0x0008
#define CHAR_TAB              0x0009
#define CHAR_LINEFEED         0x000A
#define CHAR_CARRIAGE_RETURN  0x000D
```

编译通过，preboot不通过异常。
这次是Commit [r3907] https://sourceforge.net/p/cloverefiboot/code/3907/log/?path= :update CPU definitions by ErmaC,这个commit提到的rEFIt_UEFI/Platform/下几个源码文件里有很多cpu model变化，我们记得变色龙的修正路子也是类似。要revert changes since clover r3907，把Platform.h中的define CPU_MODEL_SKYLAKE_S先删掉，然后把接下来一行CPU_MODEL_SKYLAKE_D改为CPU_MODEL_SKYLAKE_S，其它三文件StateGenerator.c,cpu.c,platformdata.c删除CPU_MODEL_SKYLAKE_D删除所有跟CPU_MODEL_SKYLAKE_D有关的orphaned case skylaked逻辑块。

5068是最接近我们上文5070的，现在直接来尝试编译5068+https://github.com/CloverHackyColor/edk2/archive/edk2-stable201908.zip，这里就不需要用上面的for 4000的预处理脚本了。revert changes since rev 23091也不需要，只需要手动处理3907的情况。toolchaindirs的逻辑还是需要处理一下的，手动去掉ebuild.sh中的nasm判断逻辑，修正其它（所幸我们不需要修正22943,23089的二段）直接改名为Clover放到edk2 src下，osx下手动粘贴合并非替换，手动工作量也小，另外，mtoc也需要编译了。手动准备mtoc，里面的是cctools-921.tar.gz，但是在osx10.11.6上会出错，用回cctools-895.tar.gz，指定xcode5,因为会默认xcode8，即sudo ebuild.sh -xcode5，

出来结果，在轻量上过preboot（表现是过preboot，但提示多处patchs，一段时间会重启，preboot后卡住无提示不重启/重启无提示/出错有提示，都大有讲究。可能都与源码中的cpu处理逻辑有关），本机虚拟机也可代替5070的cloverx64.efi用。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106338355/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


