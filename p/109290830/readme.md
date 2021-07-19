一个fully retryable的rootbuild packer脚本,从0打造matecloudos(1):实现compiletc tools
=====

__本文关键字：云packer类cloudinit,pebuilder.sh本地版，tc as general rootbuild,子shell启动脚本,可在cloudide terminal下运行，bash 数组 包含反引号命令替换会被执行，bash将任意命令放在数组中却能正常调用的方法,bash 命令字符串cmstr换行显示__

在生成黑群，黑苹果的文章中，我们用的都是在deepin+kvm/qemu的架构中进行的。除了依赖客户机desktop环境其导出镜像的过程也不是纯粹自动化的（vs命令行下一条命令跑到底的方式）。在以前《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式》系列文章和buildtc系列脚本中，我们用的是packer+本地虚拟机的方式，这种方式可以一条命令跑到底得到最终镜像。但同样依赖具体OS下的具体虚拟机还有一个iso(这相当于pebuilder.sh packer开始时的主机os)，这二种方式都存在环境条件假定，比如你不能在一台无法开kvm虚拟也没有桌面的linux服务器上进行，在前面pebuilder.sh中我们描述过一种debianinstall os，pebuilder.sh里面的脚本其实只是预处理和打包，其实真正的pack(provision)过程在于启动时的喂给它的那段preseed.cfg as boot command，借助pkg仓库这成为在线安装os/使os变得可在线化安装的良好方法(非dd方式)但它局限于debian且不是预生成方式，最后，以上都不是从0的源码构建的。

那么为了追求从0开始更彻底更可控的效果，有没有一种纯云上packer生成并从源码构建并得到offline镜像结果并为pebuilder.sh所用的方式呢？----- 即三个要求，1，在服务器环境和纯命令行中也能进行，2，offline预生成镜像，3源码编译的方式进行 ------ 所以现在，让我们来探求在任何环境下命令行生成镜像的方法，使用通用跨平台packer+qemu的组合(qemu独立使用是一个通用虚拟机而以不跟kvm一起)，服务于本文目的：从0实现tclinux based mateclous离线镜像。这段代码取自http://mirrors.163.com/tinycorelinux/11.x/x86_64/release/src/toolchain/compile_tc11_x86_64。,与前面面向cross compile和分离目录定制的方向不同，这里只面向在64tc11从0开始源码自举构建一个干净的64tc11。

思路是将compile_tc11_x86_64中所有编译目标弄成一个查表的参数数组，然后统一调用编译，为什么一定要弄成数组呢，这样可以清晰化所有的pass。

在云主机装上packer和qemu(我用的>2G ubt1804,编译gcc需要多点内存,packer为以前文章的1.4.1版本没变，qemu为apt-get install出来的qemu4)，根据上面的代码，准备所有的源码文件和目录，新建一个rootbuild.sh里面是packer build -force -on-error=ask ./matecloudos-$1.packer，以后有多个packer只需换后缀参数就可以了，本文是matecloudos-tools.packer，其它的packer逻辑在另外的文章中按需给出(设立多个packer不致于让所有工作都放在一步，可以持续集成，当然在一个packer中设好retry也是一样但这需要更多工作)，(等所有的packer都能成功可考虑合成一个packer)。

运行代码rootbuild.sh packer文件名，会执行本文的主体代码matecloudos-tools.packer，（如果你是tnt-pd16(用的sudo修复网络启动的pd)，需要sudo build.sh xxxx才能唤起prctl window view），直接给代码：

matecloudos-tools.packer：压缩boot-command和切换shell
-----

qemu相当于qemu-iso，其实qemu支持linux as firmware(xhyve也支持)，即参数中直接喂入-kernel,-initrd,-append，我们可以实现直接用linux作为iso，然后通过  qemuargs = [[ "-kernel", "/boot/xxx" ],[ "-initrd", "/boot/ixxx" ],[ "-append", " 'xxx'" ]]喂入，但packer规定必须要有一个iso启动不能完全控制启动过程。所以不准备模拟这效果了

因为我们定位于在服务器和任意网络环境生成，所以所有资源全是wgetable的。还有个问题是：因为它是打字形式的命令输入，不好处理要协调延时，所以写短一点省事，本来bootcomomand phase可以写很多。这次不行,,所以<enter><wait20>这样的故意延时很重要，因为iso中并没有安装kvm等加速器，在boot阶段大量启动显示信息会很慢。这也使得bootcommand要尽量短。，能放的都放到接下来的provisioners中去,这对整个脚本-on-error-ask后选择能有机会retry而避免涉及到格盘这种操作也是一种好的支持。压缩bootcommand也是为了将一切可能的输出结果转移到provisioners阶段，也可以省掉prlctl window view或促成headless而主要输出保留在packer那个窗口，这个阶段也是正常的输出主要场所而我们希望直接在livecd中一次完成所有事情，。2G内存是为了接下来编译gcc不致内存耗光，"headless": true保证了在无x11的服务器环境也能运行其实还有个disable_vnc可用。里面只有一条export是为了与接下来切换shell讲解部分中的其它export作对应。

> CorePure64-11.1-openssl-remaster.iso是在TinyCorePure64-11.1.iso中进桌面使用ezremaster集成openssh,parted,grub2-multi(这个不集成接下来的tce-load会失效因为没有分区)生成的(其中openssh集成到extract to initrd其它二个outside initrd on boot，而以前文章中，我们在packer中写到硬盘版tc中集成,这很麻烦比如inbuilt需求在packer中集成tcz你需要unsquashfs tczs和复制ld.conf.cache（值得一提的是boot_command中允许写sudo reboot不会退出packer这让我们可以在硬盘上准备一个非最终目标的tc不断reboot后构建，用于保存系统中间状态再构建），，ezremaster能自动处理依赖还让你进入extract目录定制的机会，我给ssh复制了一个sshd_config和mkdir了一个/var/lib/sshd目录写入了分区用的parted -s /dev/vda mktable msdos;parted -s /dev/vda mkpart primary ext3 2048s 100%;parted -s /dev/vda set 1 boot on;mkfs.ext3 /dev/vda1;rebuildfstab;mount /mnt/vda1;grub-install --boot-directory=/mnt/vda1/boot /dev/vda和一句tce-setup最后启用ssh用的/usr/local/init.d/openssh start到bootlocal.sh(注意这3个先后，这个文件里默认是sudo的)，还sudo定制了tc密码,给bootcode增加了tce=vda1 opt=vda1 home=vda1 restore=vda1到extract/..../isolinux.cfg还另外为vda1版本写了一份覆盖f2)，这样就充分压缩了boot-command

```
  "builders":[{
        "type": "qemu",
	"iso_url": "http://www.shalol.com/d/mirrors/tinycorelinux/CorePure64-11.1-ezremasterd.iso",
	"iso_checksum_type": "sha256",
	"iso_checksum": "ec6173e4ee9d64276ce07f4bb362cd7d9dfb2964ce13e9a6e0695fcdeab89b11",

	"vm_name":"tclinuxforselfbootstrape",
	"net_device": "virtio-net",
        "disk_interface": "virtio",
        "disk_size": 20000,
	"format": "raw",
        "cpus": 2,
        "memory": 2048,

	"headless": true,
	"vnc_port_min": 5960,
	"vnc_port_max": 5960,
	"boot_wait": "10s",

	"boot_command":[	
	  	"<enter><wait20>",
	  	"export<return>"
	],

	"ssh_username": "tc",
	"ssh_password": "tc",

	"shutdown_command": "echo 'tc' | sudo poweroff",
        "output_directory": "/Users/minlearn/packer/iso"

  }],

matecloudos-tools.packer provisioners:在这里我们处理即时shell和shell变量嵌套，我们不将逻辑写在packer中，packer主文件只用来设置框架。主要的逻辑放在接下来的sh中进行，这样才能避免packer json不断转义限制, packer中每一段shell chunk都会生成一个目标机/tmp下的sh，这里的问题是切换shell，父shell和子shell的变量是隔离的。sh方式运行脚本，会重新开启一个子shell，脚本中export输出的SHLVL可以看到，无法继承父进程的普通变量，能继承父进程export的全局变量。source或者. 方式运行脚本，会在当前shell下运行脚本，相当于把脚本内容加载到当前shell后执行，自然能使用前面定义的变量。compile_tc11_x86_64主要使用bash，因此必须保证所有的脚本shebang都是bash的而不是tc默认的ash。

注意此时还是tclinux的ash。

  "provisioners": [
	{
    	"type": "shell",
    	"pause_before":"1s",
    	"inline":[
		"export",

		"sudo mkdir -p /mnt/vda1/opt /mnt/vda1/home",

		"sudo rm /opt/tcemirror && sudo touch /opt/tcemirror",
		"sudo sh -c 'echo http://www.shalol.com/d/mirrors/tinycorelinux/ > /opt/tcemirror'",
		"sudo sh -c 'sed -i s/wget[[:space:]]*-c[[:space:]]/\"wget -cq \"/g /usr/bin/tce-load'",

	  	"echo ADD BASH PKG",
		"tce-load -iw -s bash",
	  	"#sudo rm -rf /mnt/vda1/boot/tmp/bin/sh",
	  	"#sudo ln -sf /usr/local/bin/bash /mnt/vda1/boot/tmp/bin/sh",
	  	"#sudo sh -c 'sed -i s#bin/sh#usr/local/bin/bash#g /etc/passwd'",

	  	"echo PRE SAVE STATE",
	  	"sudo sh -c 'echo opt/tcemirror >> /mnt/vda1/opt/.filetool.lst'",
	  	"#sudo sh -c 'echo etc/passwd >> /mnt/vda1/opt/.filetool.lst'",
	  	"sudo /bin/tar -C / -T /mnt/vda1/opt/.filetool.lst -czf /mnt/vda1/mydata.tgz",

		"echo fix the system ar",
		"sudo rm -rf /usr/bin/ar",

	  	"echo prepare for uploading",
		"sudo mkdir -p /mnt/vda1/tmp",
		"sudo chown tc:staff /mnt/vda1/tmp/",
		"#sudo rm -rf /mnt/vda1/tmp/src"

		]
	},

注意为了测试，src我暂时放在了本地跟脚本逻辑一起，未来download src的方式会放到主体脚本逻辑中一一wget。还注意到qemu虚拟网卡网络很不稳定，前面装tce都有可能出错，更别说这里的wget src.tar了，故暂时用本地上传。这是packer141 bug?qemu bug?换版本组合会好点？

	{
  	"type": "file",
  	"source": "./src",
  	"destination": "/mnt/vda1/tmp"
	},

这里将shebang切成了bash，且开始用sudo，如果这里不用sudo，那么接下来要打大量sudo，且接下来脚本中echo >,sed  这样的语句sudo sh -c ''发挥不了作用，很诡异，没有继续研究。

	{
    	"type": "shell",
    	"pause_before":"1s",
    	"execute_command": "sed -i s#bin/sh#usr/local/bin/bash#g {{ .Path }} && sudo {{ .Path }}",
    	"scripts":
		[
		"./src/2.buildtc/buildtools"
		]
    	}

  ]

```

tc脚本
-----

这里的又一个跟命令有效性相关的问题在于，将任意命令放在数组中，当展开的时候，其作用并不是像展开的字符串一次喂入的效果一样。有时候不能正常调用转义不掉，猜测是变量替换会把空格隔开的单位分行放置，比如custproc如果前面有环境变量会当成语句被执行。所以我尽量把所有的正常proc都拆开放置，并把命令字符（排除参数）放到非数组部分。尽量不做allinoneproc到数组中。这里着重要提到的是``，相当于$()，它是优先调用符（vs bash的变量替换，这是bash的命令替换），它如果放在数组中会被定义数组的时候就给优先执行了，所以必须用手动替换的方法设置CMDSEP绕过自动替换，在数组外想办法让它正常执行。

注意到shebang是手动加的
```
#!/usr/local/bin/bash

#tc11_x86_64 (on corepure64)

su - tc -c 'tce-load -iw -s compiletc rsync bc'
su - tc -c 'tce-load -iw -s compiletc perl5 ncursesw-dev bash mpc-dev udev-lib-dev texinfo coreutils glibc_apps rsync gettext python3.6 automake autoconf-archive'
slient1=w
slient2=/dev/null
slient3=s
export MAKEFLAGS="-j 2" 

#set +h
#umask 022
#用了/tmp而不是/tmp/tc，这样build src tools可以并列
TC=/mnt/sda1/tmp
LC_ALL=POSIX
TC_TGT=x86_64-tc-linux-gnu
PATH=/tools/bin:/usr/local/bin:/bin:/usr/bin
export TC LC_ALL TC_TGT PATH

export

#rm -rf $TC/tools $TC
mkdir -p $TC $TC/tools
chown tc:staff $TC/tools
#只要mount才能隐藏二级地址类301,302？
ln -sf $TC/tools /

#bash数组的每一条（成员）其实都是单条独立语句，可以放数组外分开定义。所以可以使用\换行放置，不会被归入到echo cmdstr结果中。

# 对于以下cmdstr
# 只要是语句，加不加DEFERSUBME都会被数组外bash eval当成语句发生变量替换/展开（即使是echo cmdstr都会发生变量替换，动态展开）,加个DEFERSUBME会更清楚
# 如果是非环境变量的替换/展开，需要把$转义一下，如下面的for file ... $file要转成\$file，如果是环境变量如$TC则不需要，后者在不执行时就被静态展开了

declare -A PROCESSTABLE

export PROCESSTABLE=(
    ["binutils_p1_confhz"]="--prefix=/tools --with-sysroot=$TC --with-lib-path=/tools/lib --target=$TC_TGT --disable-nls --disable-werror"

    ["gcc_p1_precmds"]="rm -rf ../mpfr;cp -rf ../../mpfr-4.0.2 ../mpfr;rm -rf ../gmp;cp -rf ../../gmp-6.1.2 ../gmp;rm -rf ../mpc;cp -rf ../../mpc-1.1.0 ../mpc; \
    for file in ../gcc/config/linux.h ../gcc/config/i386/linux.h ../gcc/config/i386/linux64.h; \
    do \
        cp -f \$file \$file.orig;rm -rf \$file; \
        sed -e 's#/lib\(64\)\?\(32\)\?/ld#/tools&#g' -e 's#/usr#/tools#g' \$file.orig > \$file; \
        echo '#undef STANDARD_STARTFILE_PREFIX_1' >> \$file;echo '#undef STANDARD_STARTFILE_PREFIX_2' >> \$file;echo '#define STANDARD_STARTFILE_PREFIX_1 \"/tools/lib/\"' >> \$file;echo '#define STANDARD_STARTFILE_PREFIX_2 \"\"' >> \$file; \
        touch \$file.orig; \
    done; \
    sed -e '/m64=/s/lib64/lib/' -i.orig ../gcc/config/i386/t-linux64"
    ["gcc_p1_confhz"]="--target=$TC_TGT --prefix=/tools --with-glibc-version=2.11 --with-sysroot=$TC --with-newlib --without-headers --with-local-prefix=/tools --with-native-system-header-dir=/tools/include --disable-nls --disable-shared --disable-multilib --disable-decimal-float --disable-threads --disable-libatomic --disable-libgomp --disable-libquadmath --disable-libssp --disable-libvtv --disable-libstdcxx --enable-languages=c,c++"

    ["linux_p1_custproc"]="make proper && make INSTALL_HDR_PATH=dest headers_install"
    ["linux_p1_postcmds"]="cp -r -f dest/include/* /tools/include"

    #["glibc_p1_precmds"]="#edit manual/libc.tcexinfo;#remove @documentencoding UTF-8"
    ["glibc_p1_confhz"]="--prefix=/tools --host=$TC_TGT --build=DEFERSUBME../scripts/config.guessDEFERSUBME --enable-kernel=4.19.10 --with-headers=/tools/include"
    #["glibc_p1_postcmds"]="echo 'int main\(\){}' > dummy.c;$TC_TGT-gcc dummy.c;readelf -l a.out | grep ': /tools';rm dummy.c a.out"

    ["gcc_p2_confhz"]="--host=$TC_TGT --prefix=/tools --disable-multilib --disable-nls --disable-libstdcxx-threads --disable-libstdcxx-pch --with-gxx-include-dir=/tools/$TC_TGT/include/c++/9.2.0"

    ["binutils_p2_confqz"]="CC=$TC_TGT-gcc AR=$TC_TGT-ar RANLIB=$TC_TGT-ranlib"
    ["binutils_p2_confhz"]="--prefix=/tools --disable-nls --disable-werror --with-lib-path=/tools/lib --with-sysroot"

    #["gcc_p3_precmds"]="#rm -rf DEFERSUBMEdirname DEFERSUBME$TC_TGT-gcc -print-libgcc-file-nameDEFERSUBMEDEFERSUBME/include-fixed/limits.h;cat gcc/limitx.h gcc/glimits.h gcc/limity.h > DEFERSUBMEdirname DEFERSUBME$TC_TGT-gcc -print-libgcc-file-nameDEFERSUBMEDEFERSUBME/include-fixed/limits.h"
    ["gcc_p3_confqz"]="CC=$TC_TGT-gcc CXX=$TC_TGT-g++ AR=$TC_TGT-ar RANLIB=$TC_TGT-ranlib"
    ["gcc_p3_confhz"]="--prefix=/tools --with-local-prefix=/tools --with-native-system-header-dir=/tools/include --enable-languages=c,c++ --disable-libstdcxx-pch --disable-multilib --disable-bootstrap --disable-libgomp"
    #["gcc_p3_postcmds"]="ln -sf gcc /tools/bin/cc;echo 'int main\(\){}' > dummy.c;$TC_TGT-gcc dummy.c;readelf -l a.out | grep ': /tools';rm dummy.c a.out"

    ["m4_p1_precmds"]="sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' lib/*.c;echo '#define _IO_IN_BACKUP 0x100' >> lib/stdio-impl.h"
    ["m4_p1_confhz"]="--prefix=/tools"

    ["ncurses_p1_precmds"]="sed -i s/mawk// configure"
    ["ncurses_p1_confhz"]="--prefix=/tools --with-shared --without-debug --without-ada --enable-widec --enable-overwrite"
    ["ncurses_p1_postcmds"]="ln -sf libncursesw.so /tools/lib/libncurses.so"

    ["bash_p1_confhz"]="LDFLAGS=-L/tools/lib"
    ["bash_p1_confhz"]="--prefix=/tools --without-bash-malloc"

    ["bison_p1_confhz"]="--prefix=/tools"
    ["bzip2_p1_installhz"]="PREFIX=/tools install"

    ["coreutils_p1_confqz"]="FORCE_UNSAFE_CONFIGURE=1"
    ["coreutils_p1_confhz"]="--prefix=/tools --enable-install-program=hostname"

    ["diffutils_p1_confhz"]="--prefix=/tools"
    ["file_p1_confhz"]="--prefix=/tools"
    ["findutils_p1_confhz"]="--prefix=/tools"
    ["gawk_p1_confhz"]="--prefix=/tools"

    ["gettext_p1_confhz"]="--disable-shared"
    #["gettext_p1_postcmds"]="#cp gettext-tools/src/msgfmt /tools/bin;#cp gettext-tools/src/msgmerge /tools/bin;#cp gettext-tools/src/xgettext /tools/bin"

    ["grep_p1_confhz"]="--prefix=/tools"
    ["gzip_p1_confhz"]="--prefix=/tools"

    #["make_p1_precmds"]="sed -i '211,217 d; 219,229 d; 232 d' glob/glob.c"
    ["make_p1_confhz"]="--prefix=/tools --without-guile"

    ["patch_p1_confhz"]="--prefix=/tools"

    ["perl_p1_custproc"]="sh Configure -des -Dprefix=/tools -Dlibs=-lm -Uloclibpth -Ulocincpth && make"
    #["perl_p1_postcmds"]="#cp perl cpan/podlators/scripts/pod2man /tools/bin;#mkdir -p /tools/lib/perl5/5.30.0;#cp -R lib/* /tools/lib/perl5/5.30.0"

    ["Python_p1_precmds"]="sed -i '/def add_multiarch_paths/a \\        return' setup.py"
    ["Python_p1_confhz"]="--prefix=/tools --without-ensurepip"

    ["sed_p1_confhz"]="--prefix=/tools"
    ["tar_p1_confhz"]="--prefix=/tools"
    ["texinfo_p1_confhz"]="--prefix=/tools"
    ["xz_p1_confhz"]="--prefix=/tools"
)

#以下代码来自cloverboot，作了修改和增强，比如接受tarballs数组
#直接用

export DOWNLOADPREFIX=${DOWNLOADPREFIX:-http://www.shalol.com/d/mirrors/buildmatecloudos}
export DIR_GCC=${DIR_GCC:-/mnt/sda1/tmp/build}
[ ! -d ${DIR_GCC} ]        && mkdir ${DIR_GCC}

### Download and Extract ###

# Function to extract source tarballs
DownloadandExtractTarballs ()
{
    exec 3>&1 1>&2 # Save stdout and redirect stdout to stderr

    tarballs=(`echo $1 | tr ',' ' '`)

    #local maintarball=${tarballs[0]}
    #local package=${maintarball}
    local package_top_level_dir=""

    for i in "${!tarballs[@]}"; do

        tarball="${DIR_DOWNLOADS}/${tarballs[i]}"

        if [[ ! -f ${tarball} ]]; then echo "Status: ${tarballs[i]} not found.";wget --no-check-certificate -qO- ${DOWNLOADPREFIX}/${tarballs[i]} > ${tarball} || exit 1; fi


        local filetype=$(file -L --brief "${tarball}" | tr '[A-Z]' '[a-z]')
        local tar_filter_option=""

        case ${filetype} in # convert to lowercase
            gzip\ *)  tar_filter_option='--gzip' ;;
            bzip2\ *) tar_filter_option='--bzip2';;
            lzip\ *)  tar_filter_option='--lzip' ;;
            lzop\ *)  tar_filter_option='--lzop' ;;
            lzma\ *)  tar_filter_option='--lzma' ;;
            xz\ *)    tar_filter_option='--xz'   ;;
            *tar\ archive*) tar_filter_option='';;
            *) echo "Unrecognized file format of '${tarball}'"
           exit 1
           ;;
        esac

        # Get the root directory from the tarball
        local first_line=$(dd if="${tarball}" bs=1024 count=256 2>/dev/null | \
            tar -t $tar_filter_option -f - 2>/dev/null | head -1)
            local top_level_dir=${first_line#./}  # remove leading ./
        top_level_dir=${top_level_dir%%/*}    # keep only the top level directory

        # save the main tarball topleveldir name
        [[ $i -eq 0 ]] && package_top_level_dir=${top_level_dir}

        [ -z "$top_level_dir" ] && echo "Error can't extract top level dir from $tarball" && exit 1

        if [[ ! -d "${DIR_GCC}/$top_level_dir" ]]; then
            echo "- ${tarballs[i]} extracting..."
            rm -rf "${DIR_GCC}/$top_level_dir" # Remove old directory if exists
            rm -rf "$DIR_GCC/tmp"   # Remove old partial extraction
            mkdir -p "$DIR_GCC/tmp" # Create temporary directory
            tar -C "$DIR_GCC/tmp" -x "$tar_filter_option" -f "${tarball}"
            mv "$DIR_GCC/tmp/$top_level_dir" "$DIR_GCC/$top_level_dir"
            rm -rf "$DIR_GCC/tmp"
            echo "- ${tarballs[i]} extracted"
        fi

    done

    # Restore stdout for the result and close file descriptor 3
    exec 1>&3-
    echo "${DIR_GCC}/$package_top_level_dir" # Return the full path where the main tarball has been extracted
}

### Compiling package ###

export DIR_LOGS=${DIR_LOGS:-/mnt/sda1/tmp/logs}
[ ! -d ${DIR_LOGS} ]       && mkdir ${DIR_LOGS}

CompilePackage ()
{

    local package="$1"
    # 百分号是去掉右边，二个百分号最大模式直到遇到最后一个-
    packagebase=${package%%-*}
    # 井号是去掉左边，二个井号最大模式直到遇到第一个:
    packagepass=${package##*:}
    packageentry=$packagebase"_"$packagepass

    # 百分号是去掉右边，一个百分号最小模式直到遇到最后一个:
    local packagefiles=${package%:*}
    local packagedir=$(DownloadandExtractTarballs "${packagefiles}") || exit 1
    local packagecheckfile=$packageentry".ok"

    packageprecmds=${PROCESSTABLE[$packageentry"_precmds"]} ; [[ $packageprecmds =~ "DEFERSUBME" ]] && packageprecmds=${packageprecmds//DEFERSUBME/'`'}
    packageconfqz=${PROCESSTABLE[$packageentry"_confqz"]} ; [[ $packageconfqz =~ "DEFERSUBME" ]] && packageconfqz=${packageconfqz//DEFERSUBME/'`'}
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
        echo "precmds:" "$packageprecmds"
        eval "$packageprecmds"
        if [ ! -n $packagecustproc ]; then eval "$packagecustproc" > $slient2;[[ $? -eq 0 ]] && touch $DIR_LOGS/$packagecheckfile; else 
            logfile="$DIR_LOGS/${packageentry}.configure.log.txt"
            echo "-  ${packageentry} configure..."
            #echo "configure:" "$packageconfqz ../configure $packageconfhz"
            eval "$packageconfqz ../configure $packageconfhz" > "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo "Error configuring ${packageentry} ! Check the log $logfile" && exit 1

            logfile="$DIR_LOGS/${packageentry}.make.log.txt"
            echo "-  ${packageentry} make..."
            eval "CC=\"gcc -$slient1\" $packagemakeqz make -$slient3 $packagemakehz" 1> "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo "Error making ${packageentry} ! Check the log $logfile" && exit 1

            logfile="$DIR_LOGS/${packageentry}.install.log.txt"
            echo "-  ${packageentry} install..."
            eval "$packageinstallqz make -$slient3 install" 1> "$logfile" 2>&1
            [[ $? -ne 0 ]] && echo "Error installing ${packageentry} ! Check the log $logfile" && exit 1 || touch $DIR_LOGS/$packagecheckfile
        fi
        echo "postcmds:" "$packagepostcmds"
        eval "$packagepostcmds"
        stopBuildEpoch=$(date -u "+%s");buildTime=$((stopBuildEpoch - startBuildEpoch));if [ $buildTime -gt 59 ]; then timeToBuild=$(printf "%dm%ds" $((buildTime/60%60)) $((buildTime%60))); else timeToBuild=$(printf "%ds" $((buildTime))); fi;printf -- "-  %s %s %s\n" "$packageentry" "Total Time Spent: " "$timeToBuild"

    else
        echo $package is already compiled and installed!
    fi
}  

echo ==============compile tools...============================


for packagedef in binutils-2.33.1.tar.xz:p1 gcc-9.2.0.tar.xz,mpfr-4.0.2.tar.xz,gmp-6.1.2.tar.xz,mpc-1.1.0.tar.gz:p1 linux-5.4.3.tar.xz:p1 glibc-2.30.tar.xz:p1 binutils-2.33.1.tar.xz:p2 gcc-9.2.0.tar.xz:p2 gcc-9.2.0.tar.xz:p3 m4-1.4.18.tar.gz:p1 ncurses-6.1.tar.gz:p1 bash-5.0.tar.gz:p1 bison-3.4.2.tar.xz:p1 bzip2-1.0.8.tar.gz:p1 coreutils-8.31.tar.xz:p1 diffutils-3.7.tar.xz:p1 file-5.37.tar.gz:p1 findutils-4.7.0.tar.xz:p1 gawk-5.0.1.tar.xz:p1 gettext-0.20.1.tar.gz:p1 grep-3.3.tar.xz:p1 gzip-1.10.tar.xz:p1 make-4.2.1.tar.gz:p1 patch-2.7.6.tar.gz:p1 perl-5.30.0.tar.gz:p1 Python-3.8.0.tar.gz:p1 sed-4.7.tar.xz:p1 tar-1.32.tar.xz:p1 texinfo-6.7.tar.gz:p1 xz-5.2.4.tar.gz:p1

do

  CompilePackage $packagedef || exit 1

done
```

此脚本还需要调试，这段脚本仅是compile_tc11_x86_64的前三分之一的部分。未来会加入新的processcmd数组和处理逻辑的组合。
调试方法，我们调用处设置了echo cmdstr，如果发现被截断或无法原形显示，就在cmdstr数组条目中进行处理修正，不断整合正确结果到脚本

------

发现一个一个可以解决云笔记的痛点问题所在。即文本的版本记录应该配合github desktop这种能动态显历史更改的东西来用。


-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/109290830/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




