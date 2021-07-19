一个fully retryable的rootbuild packer脚本,从0打造matecloudos(2):以lfs9观点看compiletc tools
=====

__本文关键字，bash 命令替换嵌套,bash echo -e输出换行,lfs Constructing a Temporary System,How to uninstall gcc installed from source__

在《一种虚拟boot作通用bootloader及一种通用qemu os的设想》和《一个fully retryable的rootbuild packer脚本,从0打造matecloudos(1):》中，我们都讲到tc11上编译/交叉编译gcc920的实践，其中前者是我基于clover build gcc8修改而来的，这套脚本有很多增强和亮点，如downloadandextracttarballs()脚本与大文件分开，源码包在外部下载，，“all slient compiling”和“判断连续命令最后一条是否出错”的方法。。，后者则是我针对tc官方的compiletc11，结合这些亮点和增强，打造的最终“力求fully retryable”的packer脚本(我们选择了tclinux这类有"动态mountable,包管理重启后没有污染","包被放在硬盘上也不占内存"，“系统与数据分离”优点的云系统来作为m同样是mate"cloud"os的基础，用packer脚本来一步步构建它)。而其实，我们之前很多文章如《把dbcolinux导出为虚拟机和docker格式》《将虚拟机集成在bios中：avatt》，也是这类实践的例子。

> 说到这个cross compile in packer，为什么要花大力气去编译toolchain且做成packer? 因为我们要力求把这个过程做成可观察参考的动态例子（packer中一切都是从0开始provisione起来的），搜索引擎搜索到的那些，都是基于传统的方法（如https://wiki.osdev.org/GCC_Cross-Compilerc能查到的那些（《一种虚拟boot...qemu os的设想》和《dbcolinux》《avatt》也属于这种方法），而lfs9(http://www.linuxfromscratch.org/lfs/downloads/9.0/LFS-BOOK-9.0.pdf)是业界的标准实践路子（《一个fully retryable的rootbuild packer脚本》和compiletc11就是属于lfs9的路子），其实编译gcc也不是什么特别复杂的工作，只是这里面的问题是从来找不到一个必然会成功的例子中间有很多坑你只能去实践：

> 1，toolchain是属于基础工具，基于“先用起来主义”随着历史发展起来的gcc很复杂，有很多配置和参数(整个gcc也没有retryable的make uninstall的支持，只有部分有)，版本之间又有差别,binutils这种东西又是独立gcc的，虽然有lfs这种标准教程和很多crossbuild工具/buildroot工具套件，但是除非你像lfs一样一步一步照做，否则离构建成功，该踩的坑你还是要踩（实际上照lfs做你照样会踩坑）。
2，如果你选择手动，除去上面提到的过程中出现的坑，结果也会出现一些，如果你不注意，有时会碰到虽然toolchain构建成功了，但是异常运行的情况（实际效果不是我们要的）。而lfs和buildroot这种套件又没有针对构建中每一个pass，pass中的subpass的中间调试信息。保证每一步结果都是正确的到最后必然正确的设施。compiletc11的脚本中有一些。
3，....
4，这一切都导致了能编译一套toolchain也其实是工作量不小的事情，也是我们这个《一个fully retryable的rootbuild packer脚本》系列要解决的问题。

不废话了。上脚本：

重新改的ezremasterd iso和packer provisioner部分
-----

这个脚本主要是清晰化了做进iso和写进packer文件的界限。被注释的都是做进了ezremasterd iso的，当然，具体方法跟被注释的语句不是一一对应的，但是效果相当。你可以将一切都保留在packer脚本中，去掉注释稍微修改即可，这样可以免ezremaster一次iso，我这样做只是为“最终服务于显式构建tc11主要逻辑，不让packer变得过乱”。

```
"provisioners": [
	{
    	"type": "shell",
    	"pause_before":"1s",
    	"inline":[
		"export PATH=$PATH:/usr/local/sbin:/usr/local/bin",

		"echo 'PREPARE HD'",
		"sudo parted -s /dev/sda mktable msdos",
		"sudo parted -s /dev/sda mkpart primary ext3 2048s 100%",
		"sudo parted -s /dev/sda set 1 boot on",
		"sudo mkfs.ext3 /dev/sda1",
		"sudo rebuildfstab",
		"sudo mount /dev/sda1",

	  	"echo 'make iso perisit for after reboot useablity(commented statements were remasterd into iso already)'",
		"sudo grub-install --boot-directory=/mnt/sda1/boot /dev/sda",
	  	"sudo cp /mnt/sr0/boot/corepure64.gz /mnt/sda1/boot/corepure64.gz",
	  	"sudo cp /mnt/sr0/boot/vmlinuz64 /mnt/sda1/boot/vmlinuz64",
	  	"sudo sh -c 'echo set timeout=3 > /mnt/sda1/boot/grub/grub.cfg'",
	  	"sudo sh -c 'echo menuentry \"tclinuxforselfbootstrape for after reboot useablity\" { >> /mnt/sda1/boot/grub/grub.cfg'",
	  	"#append bootcodes: loglevel=3 tce=sda1 opt=sda1 home=sda1 restore=sda1 cde",
	  	"sudo sh -c 'echo linux /boot/vmlinuz64 loglevel=3 tce=sda1 opt=sda1 home=sda1 restore=sda1 cde >> /mnt/sda1/boot/grub/grub.cfg'",
	  	"sudo sh -c 'echo initrd /boot/corepure64.gz >> /mnt/sda1/boot/grub/grub.cfg'",
	  	"sudo sh -c 'echo } >> /mnt/sda1/boot/grub/grub.cfg'",

	  	"echo 'ADD PKG(commented statements were remasterd into iso already)'",
		"sudo mkdir -p /mnt/sda1/opt /mnt/sda1/home /mnt/sda1/tce",
		"sudo tce-setup",
		"sudo rm /opt/tcemirror && sudo touch /opt/tcemirror",
		"sudo sh -c 'echo http://10.211.55.2:8000/tinycorelinux/ > /opt/tcemirror'",
		"sudo sh -c 'sed -i s/wget[[:space:]]*-c[[:space:]]/\"wget -cq \"/g /usr/bin/tce-load'",
		"#tce-load -iw -s openssh parted grub-multi",
		"tce-load -iw -s bash",

		"echo 'add neccessy and fix the small system issues(commented statements were remasterd into iso already)'",
		"sudo rm -rf /usr/bin/ar",
	  	"#sudo passwd tc",
	  	"#tc",
	  	"#tc",
		"#sudo cp /usr/local/etc/ssh/sshd_config.ori /usr/local/etc/ssh/sshd_config",
		"#sudo mkdir -p /var/lib/sshd/",
		"#sudo mkdir -p /usr/local/etc/ssl/certs/",
	  	"#sudo /usr/local/etc/init.d/openssh start",

	  	"echo 'PRE SAVE STATE(commented statements were remasterd into iso already)'",
	  	"sudo sh -c 'echo opt/tcemirror >> /mnt/sda1/opt/.filetool.lst'",
	  	"#sudo sh -c 'echo usr/local/etc/ssh > /mnt/sda1/opt/.filetool.lst'",
	  	"#sudo sh -c 'echo etc/passwd >> /mnt/sda1/opt/.filetool.lst'",
	  	"#sudo sh -c 'echo etc/shadow >> /mnt/sda1/opt/.filetool.lst'",
	  	"sudo /bin/tar -C / -T /mnt/sda1/opt/.filetool.lst -czf /mnt/sda1/mydata.tgz",
	  	"#sudo mv ~/tce /mnt/sda1/",
	  	"#sudo cp -R /opt /mnt/sda1",

	  	"echo 'prepare for uploading'",
		"sudo mkdir -p /mnt/sda1/tmp",
		"sudo chown tc:staff /mnt/sda1/tmp/"
		]
	}
```


修正的CompilePackage
------

注意这个脚本被作为bash module，被下一个脚本以. /mnt/sda1/tmp/src/common/common形式调用，注意这个.后面有一个空格，它表示source后面的脚本。然后里面的全局变量和全局函数就可以被父shell调用了，也就是下一个脚本。


```
#!/usr/local/bin/bash

### Download and Extract ###

......


### Compiling package ###

export DIR_LOGS=${DIR_LOGS:-/mnt/sda1/tmp/logs}
[ ! -d ${DIR_LOGS} ]       && mkdir ${DIR_LOGS}

CompilePackage ()
{

    local package="$1"
    packagebase=${package%%-*}
    packagepass=${package##*:}
    packageentry=$packagebase"_"$packagepass

    local packagefiles=${package%:*}
    local packagedir=$(DownloadandExtractTarballs "${packagefiles}") || exit 1
    local packagecheckfile=$packageentry".ok"

    packageprecmds=${PROCESSTABLE[$packageentry"_precmds"]} ; [[ $packageprecmds =~ "DEFERSUBME" ]] && packageprecmds=${packageprecmds//DEFERSUBME/'`'}
    packageconfqz=${PROCESSTABLE[$packageentry"_confqz"]} ; [[ $packageconfqz =~ "DEFERSUBME" ]] && packageconfqz=${packageconfqz//DEFERSUBME/'`'}
    packageconfcmd=${PROCESSTABLE[$packageentry"_confcmd"]} ; [[ $packageconfcmd =~ "DEFERSUBME" ]] && packageconfcmd=${packageconfcmd//DEFERSUBME/'`'}
    packageconfhz=${PROCESSTABLE[$packageentry"_confhz"]} ; [[ $packageconfhz =~ "DEFERSUBME" ]] && packageconfhz=${packageconfhz//DEFERSUBME/'`'}
    packagemakeqz=${PROCESSTABLE[$packageentry"_makeqz"]} ; [[ $packagemakeqz =~ "DEFERSUBME" ]] && packagemakeqz=${packagemakeqz//DEFERSUBME/'`'}
    packagemakehz=${PROCESSTABLE[$packageentry"_makehz"]} ; [[ $packagemakehz =~ "DEFERSUBME" ]] && packagemakehz=${packagemakehz//DEFERSUBME/'`'}
    packageinstallqz=${PROCESSTABLE[$packageentry"_installqz"]} ; [[ $packageinstallqz =~ "DEFERSUBME" ]] && packageinstallqz=${packageinstallqz//DEFERSUBME/'`'}
    packageinstallhz=${PROCESSTABLE[$packageentry"_installhz"]} ; [[ $packageinstallhz =~ "DEFERSUBME" ]] && packageinstallhz=${packageinstallhz//DEFERSUBME/'`'}
    packagepostcmds=${PROCESSTABLE[$packageentry"_postcmds"]} ; [[ $packagepostcmds =~ "DEFERSUBME" ]] && packagepostcmds=${packagepostcmds//DEFERSUBME/'`'}

    packagecustproc=${PROCESSTABLE[$packageentry"_custproc"]} ; [[ $packagecustproc =~ "DEFERSUBME" ]] && packagecustproc=${packagecustproc//DEFERSUBME/'`'}

    if [ ! -f $DIR_LOGS/"$packagecheckfile" ]; then

        cd ${packagedir}
        #[[ "$packagepass" -gt 1 ]] && rm -rf $packagedir

        cd $packagedir;mkdir -p "build"$packagepass;cd "build"$packagepass

        startBuildEpoch=$(date -u "+%s")
        if [ -n "$packageprecmds" ]; then
            logfile="$DIR_LOGS/${packageentry}.precmds.log.txt"
            echo "-  ${packageentry} precmds...."
            eval "$packageprecmds" > "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo -e "Error processing packageprecmds: \n$packageprecmds" && exit 1
        fi

        if [ ! -n "$packagecustproc" ]; then
            logfile="$DIR_LOGS/${packageentry}.configure.log.txt"
            echo "-  ${packageentry} configure..."
            [[ ! -n "$packageconfcmd" ]] && eval "$packageconfqz ../configure $packageconfhz" > "$logfile" 2>&1 || eval "$packageconfqz $packageconfcmd $packageconfhz" > "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo "Error configuring ${packageentry} ! Check the log $logfile" && exit 1

            logfile="$DIR_LOGS/${packageentry}.make.log.txt"
            echo "-  ${packageentry} make..."
            eval "CC=\"gcc -$slient1\" $packagemakeqz make -$slient3 $packagemakehz" 1> "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo "Error making ${packageentry} ! Check the log $logfile" && exit 1

            if [ ! -n "$packagepostcmds" ]; then
                logfile="$DIR_LOGS/${packageentry}.install.log.txt"
                echo "-  ${packageentry} install..."
                eval "$packageinstallqz make -$slient3 install $packageinstallhz" 1> "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo "Error installing ${packageentry} ! Check the log $logfile" && exit 1 || touch $DIR_LOGS/$packagecheckfile
            else
                logfile="$DIR_LOGS/${packageentry}.install.log.txt"
                echo "-  ${packageentry} install..."
                eval "$packageinstallqz make -$slient3 install $packageinstallhz" 1> "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo "Error installing ${packageentry} ! Check the log $logfile" && exit 1

                logfile="$DIR_LOGS/${packageentry}.postcmds.log.txt"
                echo "-  ${packageentry} postcmds...."
                eval "$packagepostcmds" > "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo -e "Error processing packagepostcmds: \n$packagepostcmds" && exit 1 || touch $DIR_LOGS/$packagecheckfile
            fi
        else
            if [ ! -n "$packagepostcmds" ]; then
                logfile="$DIR_LOGS/${packageentry}.custproc.log.txt"
                echo "-  ${packageentry} custproc..."
                eval "$packagecustproc" > "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo -e "Error processing packagecustproc: \n$packagecustproc" && exit 1 || touch $DIR_LOGS/$packagecheckfile
            else
                logfile="$DIR_LOGS/${packageentry}.custproc.log.txt"
                echo "-  ${packageentry} custproc..."
                eval "$packagecustproc" > "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo -e "Error processing packagecustproc: \n$packagecustproc" && exit 1

                logfile="$DIR_LOGS/${packageentry}.postcmds.log.txt"
                echo "-  ${packageentry} postcmds...."
                eval "$packagepostcmds" > "$logfile" 2>&1
                [[ $? -ne 0 ]] && echo -e "Error processing packagepostcmds: \n$packagepostcmds" && exit 1 || touch $DIR_LOGS/$packagecheckfile
            fi
        fi
        stopBuildEpoch=$(date -u "+%s");buildTime=$((stopBuildEpoch - startBuildEpoch));if [ $buildTime -gt 59 ]; then timeToBuild=$(printf "%dm%ds" $((buildTime/60%60)) $((buildTime%60))); else timeToBuild=$(printf "%ds" $((buildTime))); fi;printf -- "-  %s %s %s\n" "$packageentry" "Total Time Spent: " "$timeToBuild"

    else
        echo $package is already built and installed!
    fi
}
```

对应lfs9的Constructing a Temporary System部分的脚本
------

基本上对于lfs9可以这样理解：

constructing a temp system有二个toolchain main pass(p1-1这样的表示：main pass1中的subpass1)，一个是binutils-2.33.1.tar.xz:p1 gcc-9.2.0.tar.xz:p1-1 linux-5.4.3.tar.xz:p1 glibc-2.30.tar.xz:p1 gcc-9.2.0.tar.xz:p1-2，另一个是binutils-2.33.1.tar.xz:p2 gcc-9.2.0.tar.xz:p2

对于第一个pass,是构建出了x86_64-tc-linux-gnu binutils和x86_64-tc-linux-gcc，属于仅换了名字的“cross compile”，由vendor "pc"换成了tc，其它主要的部件都没变，可见本质还是native compile，第二个是用第一个重新构建出我们机器上tce-load -iw compiletc安装的x86_64-pc-linux-gnu binutils和gcc组成的toolchain，本质也是native compile，这样我们的/tools/下会有x86_64-tc-linux-gnu和x86_64-pc-linux-gnu二套toolchain。包括tce-load -iw compiletc安装的会有三套toolchain。

这样有什么用呢，这是lfs9为了构建出一个足够clean的toolchain，对于每一个pass和每一个subpass都是严格按职责细分的结果，这里仅提示重点部分：

在第一个main pass中，前一个gcc-9.2.0.tar.xz:p1-1使得它编译出来的程序仅调用/tools/lib64/下面的glibc，glibc-2.30.tar.xz:p1中实际构建出了这些libs，并写了一个测试glibc_p1_postcmds。输出结果如果是调用了/tools/lib64下面的lib，就表示通过。gcc-9.2.0.tar.xz:p1-2则是仅构建c++stdlib。注意喂给这些pass,subpass中的build,host,target tripe参数的组合。

相比mainpass1自然的过渡风格，在第二个mainpass中，这里有一个相当大的坑：因为pass2，实际上是用x86_64-tc-linux-gnu-gcc x86_64-tc-linux-gnu-ar x86_64-tc-linux-gnu-ranlib来native编译x86_64-pc-linux-gnu toolchain（由于/tools/bin一开始就被设置为靠PATH前面，gcc脚本会自然识别到x86_64-tc-linux-gnu-*），但是却不直接用--host=x86_64-tc-linux-gnu，而是用了build=host=target=x86_64-pc-linux-gnu的强制组合（这个不是compiletc11脚本中的写法，但是与config guess和实际要求的结果是一致的，只是我为了在这里脚本不会产生可能的异常而强制的，这种可能的异常就是我上面提到的“除去上面提到的过程中出现的坑，结果也会出现一些”，马上就提到），在mainpass2构建成功后，前面的main pass1作为它的接力被一定程度废弃了，现在转为main pass2来发挥主要作用了，因为main pass2构建过后的toolchain负责接下来所有从m4-1.4.18.tar.gz:p1到xz-5.2.4.tar.gz:p1的大量utils的x86_64-pc-linux-gnu native编译，还有，gcc-9.2.0.tar.xz:p2不分gcc-9.2.0.tar.xz:p2-1，gcc-9.2.0.tar.xz:p2-2是因为c++stdlib不用再拆分了。跟mainpass1一样同样要注意喂给这些pass,subpass中的build,host,target tripe参数的组合。

> 在tc11 iso+gcc920 src中，这种强制可能会产生《一种虚拟boot...qemu os的设想》文出现的cant run compiled c programs错误(见那文详解)(实际上这条命令的configure在进入tc11手动是可以的，只是脚本中不行，所以要从脚本中找原因)，《一种虚拟boot...qemu os的设想》文权宜是使用host=x86_64代替host=x86_64-pc-linux-gnu，但是这样虽然可以让脚本继续，但是结果最终是异常的，所以是不对的。

 > 要在脚本中保证最终结果，就需要强制build=host=target=x86_64-pc-linux-gnu，且使得编译能继续，解决方法是在["binutils_p2_postcmds"]中设置一条ln -sf /tools/lib /tools/lib64"，然后 ["gcc_p2_postcmds"]="ln -sf /tools/bin/x86_64-pc-linux-gnu-gcc /tools/bin/cc;再修正一下)。这样下来，脚本能跑了，最终的结果也是正确的：ldd测试这些out binary会发现正确调用了（如果不修正cc，会自动回退到tce-load -iw compiletc中的/usr/local/bin/cc，导致你会发现用gcc-p2编译的dummy.c运行时会发生ld cant find crt0在对应log中输出为空，这是异常产生的上级原因，产生异常的根源原因就在binutils p2中那条ln修正，tc11官方iso的bintuils没有x86...ar，只有ar，这也是一个可能产生异常的原因）

```
#!/usr/local/bin/bash

#tc11_x86_64 (on corepure64)

su - tc -c 'tce-load -iw -s compiletc rsync bc'
su - tc -c 'tce-load -iw -s compiletc perl5 ncursesw-dev bash mpc-dev udev-lib-dev texinfo coreutils glibc_apps rsync gettext python3.6 automake autoconf-archive'
slient1=w
slient2=/dev/null
slient3=s
export MAKEFLAGS="-j 4"

declare -A PROCESSTABLE=(
    ["binutils_p1_confhz"]="--target=x86_64-tc-linux-gnu --prefix=/tools --with-sysroot=/mnt/sda1/tmp --with-lib-path=/tools/lib --disable-nls --disable-werror"

    ["gcc_p1-1_precmds"]="rm -rf ../mpfr;cp -rf ../../mpfr-4.0.2 ../mpfr;rm -rf ../gmp;cp -rf ../../gmp-6.1.2 ../gmp;rm -rf ../mpc;cp -rf ../../mpc-1.1.0 ../mpc; \
    for file in ../gcc/config/linux.h ../gcc/config/i386/linux.h ../gcc/config/i386/linux64.h; \
    do \
        cp -f \$file \$file.orig;rm -rf \$file; \
        sed -e 's#/lib\(64\)\?\(32\)\?/ld#/tools&#g' -e 's#/usr#/tools#g' \$file.orig > \$file; \
        echo '#undef STANDARD_STARTFILE_PREFIX_1' >> \$file;echo '#undef STANDARD_STARTFILE_PREFIX_2' >> \$file;echo '#define STANDARD_STARTFILE_PREFIX_1 \"/tools/lib/\"' >> \$file;echo '#define STANDARD_STARTFILE_PREFIX_2 \"\"' >> \$file; \
        touch \$file.orig; \
    done; \
    sed -e '/m64=/s/lib64/lib/' -i.orig ../gcc/config/i386/t-linux64"
    ["gcc_p1-1_confhz"]="--target=x86_64-tc-linux-gnu --prefix=/tools --with-glibc-version=2.11 --with-sysroot=/mnt/sda1/tmp --with-newlib --without-headers --with-local-prefix=/tools --with-native-system-header-dir=/tools/include --disable-nls --disable-shared --disable-multilib --disable-decimal-float --disable-threads --disable-libatomic --disable-libgomp --disable-libquadmath --disable-libssp --disable-libvtv --disable-libstdcxx --enable-languages=c,c++"

    ["linux_p1_custproc"]="cp -f /mnt/sda1/tmp/src/1.buildbase/kernelandtoolchain/config-5.4.3-tinycore64 ../.config;cd ../ && make mrproper && make INSTALL_HDR_PATH=dest headers_install"
    ["linux_p1_postcmds"]="cp -rf dest/include/* /tools/include"

    #["glibc_p1_precmds"]="#edit manual/libc.tcexinfo;#remove @documentencoding UTF-8"
    ["glibc_p1_confhz"]="--host=x86_64-tc-linux-gnu --prefix=/tools --build=DEFERSUBME../scripts/config.guessDEFERSUBME --enable-kernel=4.19.10 --with-headers=/tools/include"
    ["glibc_p1_postcmds"]="echo 'int main(){}' > dummy.c; \
    x86_64-tc-linux-gnu-gcc dummy.c; \
    readelf -l a.out | grep ': /tools'; \
    rm dummy.c a.out; \
    ln -sf /tools/lib /tools/lib64"

    ["gcc_p1-2_confcmd"]="../libstdc++-v3/configure"
    ["gcc_p1-2_confhz"]="--host=x86_64-tc-linux-gnu --prefix=/tools --disable-multilib --disable-nls --disable-libstdcxx-threads --disable-libstdcxx-pch --with-gxx-include-dir=/tools/x86_64-tc-linux-gnu/include/c++/9.2.0"

    ["binutils_p2_confqz"]="CC=x86_64-tc-linux-gnu-gcc AR=x86_64-tc-linux-gnu-ar RANLIB=x86_64-tc-linux-gnu-ranlib"
    ["binutils_p2_confhz"]="--build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --target=x86_64-pc-linux-gnu --prefix=/tools --disable-nls --disable-werror --with-lib-path=/tools/lib --with-sysroot"
    ["binutils_p2_postcmds"]="make -C ld clean; \
    make -C ld LIB_PATH=/usr/lib:/lib; \
    cp ld/ld-new /tools/bin"

    ["gcc_p2_precmds"]="cat ../gcc/limitx.h ../gcc/glimits.h ../gcc/limity.h > /tools/lib/gcc/x86_64-tc-linux-gnu/9.2.0/include-fixed/limits.h"
    ["gcc_p2_confqz"]="CC=x86_64-tc-linux-gnu-gcc CXX=x86_64-tc-linux-gnu-g++ AR=x86_64-tc-linux-gnu-ar RANLIB=x86_64-tc-linux-gnu-ranlib"
    ["gcc_p2_confhz"]="--build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --target=x86_64-pc-linux-gnu --prefix=/tools --with-local-prefix=/tools --with-native-system-header-dir=/tools/include --enable-languages=c,c++ --disable-libstdcxx-pch --disable-multilib --disable-bootstrap --disable-libgomp"
    ["gcc_p2_postcmds"]="ln -sf /tools/bin/x86_64-pc-linux-gnu-gcc /tools/bin/cc; \
    echo 'int main(){}' > dummy.c; \
    cc dummy.c; \
    readelf -l a.out | grep ': /tools'; \
    rm dummy.c a.out"

    ["m4_p1_precmds"]="sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' ../lib/*.c;echo '#define _IO_IN_BACKUP 0x100' >> ../lib/stdio-impl.h"
    ["m4_p1_confhz"]="--prefix=/tools"

    ["ncurses_p1_precmds"]="sed -i s/mawk// ../configure"
    ["ncurses_p1_confhz"]="--prefix=/tools --with-shared --without-debug --without-ada --enable-widec --enable-overwrite"
    ["ncurses_p1_postcmds"]="ln -sf libncursesw.so /tools/lib/libncurses.so"

    ["bash_p1_confqz"]="LDFLAGS=-L/tools/lib"
    ["bash_p1_confhz"]="--prefix=/tools --without-bash-malloc"
    ["bash_p1_postcmds"]="ln -s bash /tools/bin/DEFERSUBMEecho 'sh'DEFERSUBME"

    ["bison_p1_confhz"]="--prefix=/tools"
    ["bzip2_p1_custproc"]="cd ../ && make && make PREFIX=/tools install"

    ["coreutils_p1_confqz"]="FORCE_UNSAFE_CONFIGURE=1"
    ["coreutils_p1_confhz"]="--prefix=/tools --enable-install-program=hostname"

    ["diffutils_p1_confhz"]="--prefix=/tools"
    ["file_p1_confhz"]="--prefix=/tools"
    ["findutils_p1_confhz"]="--prefix=/tools"
    ["gawk_p1_confhz"]="--prefix=/tools"

    ["gettext_p1_precmds"]="cp /usr/local/lib/libgnuintl.so.8 /usr/local/lib/libgnuintl.so"
    ["gettext_p1_confhz"]="--disable-shared"
    ["gettext_p1_postcmds"]="cp gettext-tools/src/msgfmt /tools/bin;cp gettext-tools/src/msgmerge /tools/bin;cp gettext-tools/src/xgettext /tools/bin"

    ["grep_p1_confhz"]="--prefix=/tools"
    ["gzip_p1_confhz"]="--prefix=/tools"

    ["make_p1_precmds"]="sed -i '211,217 d; 219,229 d; 232 d' ../glob/glob.c"
    ["make_p1_confhz"]="--prefix=/tools --without-guile"

    ["patch_p1_confhz"]="--prefix=/tools"

    ["perl_p1_custproc"]="cd ../ && ./Configure -des -Dprefix=/tools -Dlibs=-lm -Uloclibpth -Ulocincpth && make"
    ["perl_p1_postcmds"]="cp perl cpan/podlators/scripts/pod2man /tools/bin;mkdir -p /tools/lib/perl5/5.30.0;cp -R lib/* /tools/lib/perl5/5.30.0"

    ["Python_p1_precmds"]="sed -i '/def add_multiarch_paths/a \\        return' ../setup.py"
    ["Python_p1_confhz"]="--prefix=/tools --without-ensurepip"

    ["sed_p1_confhz"]="--prefix=/tools"
    ["tar_p1_confqz"]="FORCE_UNSAFE_CONFIGURE=1"
    ["tar_p1_confhz"]="--prefix=/tools"
    ["texinfo_p1_confhz"]="--prefix=/tools"
    ["xz_p1_confhz"]="--prefix=/tools"
)

#########begein main#########

#set +h
#umask 022
TC=/mnt/sda1/tmp
LC_ALL=POSIX
TC_TGT=x86_64-tc-linux-gnu
PATH=/tools/bin:/usr/local/bin:/bin:/usr/bin
export PROCESSTABLE TC LC_ALL TC_TGT PATH

#rm -rf $TC/tools $TC
#mkdir -p $TC
mkdir -p $TC/tools
chown tc:staff $TC/tools
ln -sf $TC/tools /

export

这里就是上一节说的bash module应用。

. /mnt/sda1/tmp/src/common/common

for packagedef in binutils-2.33.1.tar.xz:p1 gcc-9.2.0.tar.xz,mpfr-4.0.2.tar.xz,gmp-6.1.2.tar.xz,mpc-1.1.0.tar.gz:p1-1 linux-5.4.3.tar.xz:p1 glibc-2.30.tar.xz:p1 gcc-9.2.0.tar.xz:p1-2 binutils-2.33.1.tar.xz:p2 gcc-9.2.0.tar.xz:p2 m4-1.4.18.tar.gz:p1 ncurses-6.1.tar.gz:p1 bash-5.0.tar.gz:p1 bison-3.4.2.tar.xz:p1 bzip2-1.0.8.tar.gz:p1 coreutils-8.31.tar.xz:p1 diffutils-3.7.tar.xz:p1 file-5.37.tar.gz:p1 findutils-4.7.0.tar.xz:p1 gawk-5.0.1.tar.xz:p1 gettext-0.20.1.tar.gz:p1 grep-3.3.tar.xz:p1 gzip-1.10.tar.xz:p1 make-4.2.1.tar.gz:p1 patch-2.7.6.tar.gz:p1 perl-5.30.0.tar.gz:p1 Python-3.8.0.tar.gz:p1 sed-4.7.tar.xz:p1 tar-1.32.tar.xz:p1 texinfo-6.7.tar.gz:p1 xz-5.2.4.tar.gz:p1

do

  CompilePackage $packagedef || exit 1

done

#########end main#########
```
以上脚本经过了调试，全程是可以正确运行的。在Constructing a Temporary System过后，脚本会进一步chroot到/tools/bin，这样就彻底隔离了本机上tce-load -iw compiletc安装的版本带来的脚本能继续但结果异常的情况。

相比前面文章，本文将begin main,end main块放到最后，将那个大大的processtable放在前面（注意到里面的$TC,$TC_TGT都被硬编了），我们这样做，就是为了和接下来讲解chroot下的脚本内容安排时，统一将processtable一起export过去，做到结构一致和清晰化。







