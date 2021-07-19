一个fully retryable的rootbuild packer脚本,从0打造matecloudos(3):以lfs9观点看compiletc tools in advance
=====

__本文关键字：shell中的数组作为参数传递且带下标,bash 数组作为变量，bash 数组作为环境变量，bash中的以及多层单双引号转义处理，bash中Shell嵌套中的$转义处理, chrooted shell $ escape__

在《一个fully retryable的rootbuild packer脚本,从0打造matecloudos(2)》中，我们见到了一种用数组化命令字符串和统一compiletarget()的的方式来构建lfs9的compiletc11基础部分，在那文的结尾，我们提到，为了清晰化这种布局，我们把那个大数组放到了前面，把common部分也作为bash module，这样主脚本文件的后面是逻辑主体，很短小很清晰，我们还提到，利用export可以将上层shell的processtable带入到下层shell，这就是本文要讲到的chroot环境和通过``，$()等命令替换，甚至bash -c ""等方式开启的子shell环境。

> 怎么在上下级shell间传递一个大大的命令数组？为什么要这样做？
因为bash中，主从shell间的变量是主shell的变量能传到子shell，且数组不是一级公民，不能直接传递，也不好通过传递${arr[@]}的方式进行（因为如果这样，在子shell中还要重新组装一次为新数组），所以我们在主shell中，把数组先设为一个大大的字符串（数组化每条命令字符串和字符串化整个数组的区别正是本文与上文的区别之一），然后通过export导出这个大串到子shell，在子shell中重新构建为数组只需一个declare -A，declare -A是bash中定义关联数组的方法，我们在上二文都用到了，必须显式定义。而且它会进入到shell的环境变量，通过export和declare -A可以看到（注意，如果你在脚本中自动输出，里面的条目都是打乱的，并不影响脚本逻辑使用，如果手动粘贴命令，则可以看到按定义顺序显示的数组条目）。

接上文，本文讲解的是compiletc11的第三部分，将上面涉及到的要点进行讲解（注意上面提到三种环境chroot,substutue和bash -c，都将在接下来涉及）。

废话不说，上脚本。


增加了chroot支持的common
-----

主要是把路径形式按chrootmode设置调用是chroot的/下还是非chroot下的/mnt/sda1/tmp下（文章2的普通模式）

```
#!/usr/local/bin/bash

export chrootmode='0'

while [[ $# -ge 1 ]]; do
  case $1 in
    -c|--chroot)
      shift
      chrootmode="$1"
      shift
      ;;
    *)
      if [[ "$1" != 'error' ]]; then echo -ne "\nInvaild option: '$1'\n\n"; fi
      echo -ne " Usage(args are self explained):\n\tbash $(basename $0)\t-c/--chroot\n\t\t\t\t\n"
      exit 1;
      ;;
    esac
  done

DOWNLOADPREFIX=${DOWNLOADPREFIX:-http://10.211.55.2:8000/buildmatecloudos}

[[ "$chrootmode" == '0' ]] && DIR_DOWNLOADS=${DIR_DOWNLOADS:-/mnt/sda1/tmp/build} || DIR_DOWNLOADS=${DIR_DOWNLOADS:-/build}
[[ "$chrootmode" == '0' ]] && DIR_GCC=${DIR_GCC:-/mnt/sda1/tmp/build} || DIR_GCC=${DIR_GCC:-/build}
[[ "$chrootmode" == '0' ]] && DIR_LOGS=${DIR_LOGS:-/mnt/sda1/tmp/logs} || DIR_LOGS=${DIR_LOGS:-/logs}

[ ! -d ${DIR_DOWNLOADS} ]        && mkdir ${DIR_DOWNLOADS}
[ ! -d ${DIR_GCC} ]        && mkdir ${DIR_GCC}
[ ! -d ${DIR_LOGS} ]       && mkdir ${DIR_LOGS}

然后是二个函数不变
......

```

buildrootbase中的字串化大数组以及多层引号转义处理
-----

buildrootbase是另起的脚本文件，对应lfs9的compiletc11的“make basic filesystem”，以示与上文“constructing a temp system”分开。(如果你要看详细的lfs，从一个低的版本开始，如lfs62，高版本lfs9有相当大的省略)

这里面的重点是：由于是一个大“”括起来的大字串，包括开头(和末尾的)在内，实际上是整个数组定义字串化的全部内容，所以相比文章中pure processtable，这里命名为processtablestr，与前者比较主要的区别在：里面如果有双引号要用\转义，如果双引号中还有双引号，那么双引号要改成单引号，因为bash只允许最大三级的引号嵌套（这是我总结的不知对不对），即3级转义“”：一级不用转，二级用\，三级用'，直观的手段是你可以直接在一个bash shell中粘贴这个"processtablestr="大字串"以进行测试，如果没有出错，差不多就行了，如果有出错，在转化成数组后，调用数组时，会提示出错must use subscript when assigning associative array。


```
PROCESSTABLESTR="(
    [\"linux_p2_custproc\"]=\"cp -f /src/1.buildbase/kernelandtoolchain/config-5.4.3-tinycore64 ../.config;cd ../ && make mrproper && make headers\"
    [\"linux_p2_postcmds\"]=\"find usr/include -name '.*' -delete; \
    rm usr/include/Makefile; \
    cp -rv usr/include/* /usr/include\"

    [\"glibc_p2_precmds\"]=\"cd ../ && patch -Np1 -i /src/1.buildbase/kernelandtoolchain/glibc-2.30-fhs-1.patch; \
    sed -i '/asm.socket.h/a# include <linux/sockios.h>' sysdeps/unix/sysv/linux/bits/socket.h; \
    ln -sfv ../../lib/ld-linux-x86-64.so.2 /lib64; \
    cd buildp2; \
    echo 'CFLAGS += -mtune=generic -Os -pipe' > configparms\"
    [\"glibc_p2_confqz\"]=\"CC='gcc -ffile-prefix-map=/tools=/usr'\"
    [\"glibc_p2_confhz\"]=\"--prefix=/usr --disable-werror --libexecdir=/usr/lib/glibc --enable-kernel=4.19.10 --enable-stack-protector=strong --with-headers=/usr/include libc_cv_slibdir=/lib --enable-obsolete-rpc\"
    [\"glibc_p2_mdlcmds1\"]=\"find . -name config.make -type f -exec sed -i 's/-g -O2//g' {} \\; ; \
    find . -name config.status -type f -exec sed -i 's/-g -O2//g' {} \\; \"
    [\"glibc_p2_mdlcmds2\"]=\"touch /etc/ld.so.conf; \
    sed '/test-installation/s@DEFERSUBMEPERLDEFERSUBME@echo not running@' -i Makefile\"
    [\"glibc_p2_postcmds\"]=\"cp ../nscd/nscd.conf /etc/nscd.conf; \
    mkdir -p /var/cache/nscd; \
    make localedata/install-locales; \
    sed -i 's@lib64/ld-linux-x86-64.so.2@lib/ld-linux-x86-64.so.2@' /usr/bin/ldd; \
    mv -v /tools/bin/{ld,ld-old}; \
    mv -v /tools/x86_64-pc-linux-gnu/bin/{ld,ld-old}; \
    mv -v /tools/bin/{ld-new,ld}; \
    ln -sv /tools/bin/ld /tools/x86_64-pc-linux-gnu/bin/ld; \
    gcc -dumpspecs | sed -e 's@/tools@@g' -e '/\*startfile_prefix_spec:/{n;s@.*@/usr/lib/ @}' -e '/\*cpp:/{n;s@$@ -isystem /usr/include@}' > /tools/lib/gcc/x86_64-tc-linux-gnu/9.2.0/specs; \
    sed -i 's@lib64/ld-linux-x86-64.so.2@lib/ld-linux-x86-64.so.2@' /tools/lib/gcc/x86_64-pc-linux-gnu/9.2.0/specs; \
    echo 'int main(){}' > dummy.c; \
    cc dummy.c -v -Wl,--verbose &> dummy.log; \
    readelf -l a.out | grep ': /lib'; \
    grep -o '/usr/lib.*/crt[1in].*succeeded' dummy.log; \
    grep -B1 '^ /usr/include' dummy.log; \
    grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'; \
    grep '/lib.*/libc.so.6 ' dummy.log; \
    grep 'found' dummy.log; \
    rm -v dummy.c a.out dummy.log\"

    ......

)"
```

这里仅给出部分是因为还没有最终调试正确出所有的编译条目，注意与上一文的细节区别。

buildrootbase中的主逻辑以及子Shell和Shell嵌套中的$转义处理
-----

下面的mkdir和mount为了追求脚本在调试时可retryable，都尽量用了判断和-p。mount部分需要重构。

```
#########begin main#########

TC=/mnt/sda1/tmp
mkdir -p $TC/dev
mkdir -p $TC/proc
mkdir -p $TC/sys
mkdir -p $TC/run
[[ ! -e $TC/dev/console ]] && mknod -m 600 $TC/dev/console c 5 1
[[ ! -e $TC/dev/null ]] && mknod -m 666 $TC/dev/null c 1 3
if awk -v status=1 '$2 == "$TC/dev" {status=0} END {exit status}' /proc/mounts; then   echo "yes"; else   mount --bind /dev $TC/dev; fi
if awk -v status=1 '$2 == "$TC/dev/pts" {status=0} END {exit status}' /proc/mounts; then   echo "yes"; else   mount -t devpts devpts $TC/dev/pts -o gid=5,mode=620; fi
if awk -v status=1 '$2 == "$TC/proc" {status=0} END {exit status}' /proc/mounts; then   echo "yes"; else   mount -t proc proc $TC/proc; fi
if awk -v status=1 '$2 == "$TC/sys" {status=0} END {exit status}' /proc/mounts; then   echo "yes"; else   mount -t sysfs sysfs $TC/sys; fi
if awk -v status=1 '$2 == "$TC/run" {status=0} END {exit status}' /proc/mounts; then   echo "yes"; else   mount -t tmpfs tmpfs $TC/run; fi
chown -R root:root $TC/tools
ln -sf $TC/tools /

注意这里，这里完成了开头讲的通过env变量export传递processtablestr，和稍后在chrooted bash shell中将这个big strdeclare -A重构为数组的动作，注意$(echo \$PROCESSTABLE)，实际上开启了一个子shell作命令替换，这是一个用$()产生子shell的用法，所以这里要\转义$这个变量替换，所以这里可以解释为，export是没经过转义的原始字符串，而命令替换内部的echo出来的经过了转义的。否则$PROCESSTABLE根本取不到值。

实际上在讲到bash中嵌套的命令替换/变量替换的相关技术中，一般都会讲到内部的变量要经过转义。

chroot $TC /tools/bin/env -i MAKEFLAGS="-j 2" PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin PROCESSTABLE="${PROCESSTABLESTR}" /tools/bin/bash -c "
declare -A PROCESSTABLE=$(echo \$PROCESSTABLE); \
export; \

......

那么重点来了，如果上述是二级嵌套，那么这里又重新启动了一个bash -c，属于三级shell嵌套了，联系到上一节的引号嵌套转义，可以用相似的处理方法，将这个shell cmd str用''括起来（实际上''本来也是一种标识性的转义），而里面的命令和变量替换符，如CompilePackage \$packagedef，需要转义一次，否则$packagedef也根本取不到值。所以这里发生了一次三级引号转义和普通命令变量替换符$转义。

exec /tools/bin/bash --login +h -c ' \

touch /var/log/{btmp,lastlog,faillog,wtmp}; \
chmod 664  /var/log/lastlog; \
chmod 600  /var/log/btmp; \

这里的-c 1，使得buildrootbase采用正确的chroot相关路径设定。

. /src/common/common -c 1 ; \

for packagedef in linux-5.4.3.tar.xz:p2 glibc-2.30.tar.xz:p2 zlib-1.2.11.tar.gz:p1 file-5.37.tar.gz:p2 readline-7.0.tar.gz:p1 m4-1.4.18.tar.gz:p2 bc-2.2.0.tar.xz:p1 binutils-2.33.1.tar.xz:p3 gmp-6.1.2.tar.xz:p1 mpfr-4.0.2.tar.xz:p1 mpc-1.1.0.tar.gz:p1 gcc-9.2.0.tar.xz:p4; \
do \
  CompilePackage \$packagedef || exit 1; \
done; \
' \
"

#########end main#########
```


----------


稍后的文章会给出完全调试正确后的结果，这里仅给出技术路线。





