把DI当online packer用:利用installnet制作一个云装机packerpe（1）
=====

__本文关键字：pebuilder,云主机装机就用云镜像装机。直接在网络上组装生成目标系统，模拟群晖的webassit,wget --spider to stdout not work__

前面《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式》系列中我们讲到了packer，这是一个类虚拟机管理器的工具，它有一个iso模式，允许用户提供一个最原始的iso和一系列脚本。，然后在这个系统上运行脚本内命令行，完成格式硬盘，安装软件包，动态生成目标系统镜像内容。关机即得这个镜像。packer也常被用来在本地生成虚拟机镜像，然后上传导入到云主机供云主机用，然而，这有一个缺点，生成镜像通常很大，本地网络拉取一些特定区的镜像速度也不太好，最后，还得上传，那么，有没有一种在云主机上类packer，边拉取软件边生成目标镜像的工具呢？如果有，是否适用所有系统呢，比如windows不提供linux方式的kernel与rootfs分离，也不支持软件包仓库丰富系统，是不是也同样适用这个过程呢？

—————
Ps:debianinstaller，是一种debian netboot system组成的类packer的环境，当可引导安装介质(安装光盘，安装U盘)启动后，可以选择进入文本向导模式或图形向导模式，然后完成如下操作  加载 kernel，initrd，加载额外的Udeb包,完成安装界面初始化环境; 如果启动参数包含preseed文件，则会按照preseed文件内定义的规则自动执行,如果没有对应的规则,则返回交互界面; 交互界面会引导用户完成键盘，时区，主机名，网络，用户密码，分区等设置，存储在当前安全器环境中; 完成分区格式化等操作后，该磁盘会被挂载到 /target，安装器会调用debootstrap在 /target 完成核心系统的构建; 安装器通过执行 chroot 操作进入以 /target 为根的系统中，完成软件源的更新，执行 tasksel，弹出可选软件菜单; 安装可选的软件包, 完成上述操作后,最后配置 grub, 将之前保存的全部设置应用，完成安装过程.  
—————

可见，整个debian installer 类似packer，只不过这里bootstrap系统是debian（packer iso模式下的iso）。工具是Debian-installer等，脚本是preseed,provision过程是运行这个pe回放preseed。当被用于云主机时，使用debian installerb也可在其中完成识别硬盘格式化等操作，，实际上它开始是一个按preseed自动化的livecd，先启动进livecd，还原回放preseed时，可以在这里继续安装完整系统（preseed）完成即得到目标镜像，debian-installer不光用来生成debian派生系的镜像，对于windows,osx这种没有在线组装能力的系统而言，可直接在preseed早期就通过wget,tar,dd的管理道dd镜像完成安装目标系统 ——— 格盘，装系统，这其实是一种比服务商提代的镜像恢复方式和dd方式更符合云机装机的原生方式。《云主机上装windows iso》《virtio 0pe》这是我早期使用virtiope网络装机的文章，我们讲到过pebuilder，我们还谈到过群晖的webassit都是类比物，在后面一些文章，尤其是《云主机上装黑群，黑果》系列中我们都提到过moeclue的installnet.sh脚本，用它网络装机可在线完成。就相当pebuilder ，生成的pe相当packer ，

下面来完善这个脚本

注意！！！！！！！！：使用此脚本安装镜像默认会抹除你硬盘中的所有内容。请谨慎尝试。
注意！！！！！！！！：使用此脚本安装镜像默认会抹除你硬盘中的所有内容。请谨慎尝试。
注意！！！！！！！！：使用此脚本安装镜像默认会抹除你硬盘中的所有内容。请谨慎尝试。


前置
-----

主要是把selectmirror精简为dd only，把一些全局变量置为局部过程变量，不用作为参数喂给脚本。

```
#!/bin/bash

## License: GPL,Written By MoeClub.org,moded by minlearn for dd purpose only

export tmpDDURL=''
export tmpMIRROR=''
export tmpINSTANTWITHOUTVNC='0'

export ipAddr=''
export ipMask=''
export ipGate=''

export UNKNOWHW='0'
export UNVER='6.4'

while [[ $# -ge 1 ]]; do
  case $1 in
    -dd|--ddurl)
      shift
      tmpDDURL="$1"
      shift
      ;;
    --mirror)
      shift
      tmpMIRROR="$1"
      shift
      ;;
    -i|--instantwithoutvnc)
      shift
      tmpINSTANTWITHOUTVNC="$1"
      shift
      ;;
    --ip-addr)
      shift
      ipAddr="$1"
      shift
      ;;
    --ip-mask)
      shift
      ipMask="$1"
      shift
      ;;
    --ip-gate)
      shift
      ipGate="$1"
      shift
      ;;
    *)
      if [[ "$1" != 'error' ]]; then echo -ne "\nInvaild option: '$1'\n\n"; fi
      echo -ne " Usage(args are self explained):\n\tbash $(basename $0)\t-dd/--ddurl\n\t\t\t\t--mirror\n\t\t\t\t-i/--instantwithoutvnc\n\t\t\t\t--ip-addr/--ip-gate/--ip-mask\n\t\t\t\t\n"
      exit 1;
      ;;
    esac
  done

[[ "$EUID" -ne '0' ]] && echo "Error:This script must be run as root!" && exit 1;

function CheckDependence(){
  FullDependence='0';
  for BIN_DEP in `echo "$1" |sed 's/,/\n/g'`
    do
      if [[ -n "$BIN_DEP" ]]; then
        Founded='0';
        for BIN_PATH in `echo "$PATH" |sed 's/:/\n/g'`
          do
            ls $BIN_PATH/$BIN_DEP >/dev/null 2>&1;
            if [ $? == '0' ]; then
              Founded='1';
              break;
            fi
          done
        if [ "$Founded" == '1' ]; then
          echo -en "$BIN_DEP";
        else
          FullDependence='1';
          echo -en "[\033[31mNot Install\033[0m]";
        fi
        echo -en "\t\t\t[\033[32mok\033[0m]\n";
      fi
  done
  if [ "$FullDependence" == '1' ]; then
    echo -ne "\n\033[31mError! \033[0mPlease use '\033[33mapt-get\033[0m' or '\033[33myum\033[0m' install it.\n\n\n"
    exit 1;
  fi
}

clear && echo -e "\n\n\n\n\n\n\n\n\n\n\033[36m# Check Dependence\033[0m\n"

CheckDependence wget,ar,awk,grep,sed,cut,cat,cpio,curl,gzip,find,dirname,basename;

function SelectMirror(){

  [ $# -ge 1 ] || exit 1
  AddonMirror=$(echo "$1" |sed 's/\ //g')

  declare -A MirrorBackup
  MirrorBackup=(["Debian0"]="" ["Debian1"]="http://httpredir.debian.org/debian")
  echo "$AddonMirror" |grep -q '^http://\|^https://\|^ftp://' && MirrorBackup[{"Debian"}0]="$AddonMirror"

  for mirror in $(echo "${!MirrorBackup[@]}" |sed 's/\ /\n/g' |sort -n |grep "^Debian")
    do
      CurMirror="${MirrorBackup[$mirror]}"
      [ -n "$CurMirror" ] || continue

      CheckPass1='0';
      DistsList="$(wget --no-check-certificate -qO- "$CurMirror/dists/" |grep -o 'href=.*/"' |cut -d'"' -f2 |sed '/-\|old\|Debian\|experimental\|stable\|test\|sid\|devel/d' |grep '^[^/]' |sed -n '1h;1!H;$g;s/\n//g;s/\//\;/g;$p')";
      for DIST in `echo "$DistsList" |sed 's/;/\n/g'`
        do
          [[ "$DIST" == "jessie" ]] && CheckPass1='1' && break;
        done
      [[ "$CheckPass1" == '0' ]] && {
        echo -ne '\njessie not find in $CurMirror/dists/, Please check it! \n\n'
        bash $0 error;
        exit 1;
      }

      CheckPass2=0
      ImageFile="SUB_MIRROR/dists/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz"
      [ -n "$ImageFile" ] || exit 1
      URL=`echo "$ImageFile" |sed "s#SUB_MIRROR#${CurMirror}#g"`
      wget --no-check-certificate --spider --timeout=3 -o /dev/null "$URL"
      [ $? -eq 0 ] && CheckPass2=1 && echo "$CurMirror" && break
    done

    [[ $CheckPass2 == 0 ]] && {
      echo -ne "\033[31mError! \033[0minitrd.gz not find in $CurMirror/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/! \n";
      bash $0 error;
      exit 1;
    }
}

sleep 5s

echo -e "\n\n\033[36m# Select Mirror\033[0m\n"
LinuxMirror=$(SelectMirror "$tmpMIRROR")
echo -e "${LinuxMirror}"
```

缓存udebs和kernel,rootfs:
-----

把udebs仓库也缓存下来放进pe，做成本地镜像。因为rootfs中有一个httpd，可以通过preseed启动，做成仓库镜像服务器。可以看到preseed中地址用了127.0.0.1，验证也去掉了By default the installer requires that repositories be authenticated using a known gpg key. This setting can be used to disable that authentication,a udeb就是一个方便在liveos中运行的简化的deb。

```
sleep 5s

echo -e "\n\033[36m# All prerequisites Done,Begin processing [DD URL:$tmpDDURL,Instant without vnc:$tmpINSTANTWITHOUTVNC]\033[0m\n"

[[ -d /tmp/boot ]] && rm -rf /tmp/boot;
mkdir -p /tmp/boot/usr/bin;
cd /tmp/boot;

LIBC6_SUPPORT='pool/main/g/glibc/libc6_2.19-18+deb8u10_amd64.deb'
WGETSSL_SUPPORT=''
declare -A HTTPD_SUPPORT
HTTPD_SUPPORT=(
["webfs1"]="pool/main/w/webfs/webfs_1.21+ds1-10_amd64.deb"
["webfs2"]="pool/main/g/gnutls28/libgnutls-deb0-28_3.3.8-6+deb8u7_amd64.deb"
["webfs3"]="pool/main/p/p11-kit/libp11-kit0_0.20.7-1_amd64.deb"
["webfs4"]="pool/main/libt/libtasn1-6/libtasn1-6_4.2-3+deb8u3_amd64.deb"
["webfs5"]="pool/main/n/nettle/libnettle4_2.7.1-5+deb8u2_amd64.deb"
["webfs6"]="pool/main/n/nettle/libhogweed2_2.7.1-5+deb8u2_amd64.deb"
["webfs7"]="pool/main/g/gmp/libgmp10_6.0.0+dfsg-6_amd64.deb"
["webfs8"]="pool/main/libf/libffi/libffi6_3.1-2+deb8u1_amd64.deb"
["webfs9"]="pool/main/m/mime-support/mime-support_3.58_all.deb"
)
DDPROGRESS_SUPPORT=''
UNZIP=''

[[ -n "$LIBC6_SUPPORT" ]] && {
  echo -ne 'Add libc6 support(binary-amd64/Package)...'

  wget --no-check-certificate -qO ${LIBC6_SUPPORT##*/} $LinuxMirror/$LIBC6_SUPPORT; \
  ar x ${LIBC6_SUPPORT##*/} data.tar.gz; tar xzf data.tar.gz; \
  rm -rf data.tar.gz ${LIBC6_SUPPORT##*/}

  [[ ! -f  /tmp/boot/lib/x86_64-linux-gnu/libc.so.6 ]] && echo 'Error! LIBC6_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
  # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
}


if [[ -n "$tmpDDURL" ]]; then
  echo "$tmpDDURL" |grep -q '^http://\|^ftp://\|^https://';
  [[ $? -ne '0' ]] && echo 'No valid URL in the DD argument,Only support http://, ftp:// and https:// !' && exit 1;
  curl -Is "$tmpDDURL" | grep -q 'gzip';
  [[ $? -ne '0' ]] && echo 'not tar or gunzip file !' && exit 1 || UNZIP='0';
else
  echo 'Please input vaild image URL! ';
  exit 1;
fi

echo "$tmpDDURL" |grep -q '^https://'
[[ $? -eq '0' ]] && {

  [[ -n "$WGETSSL_SUPPORT" ]] && {
    echo -ne 'Add ssl support(binary-amd64/Package)...'

    wget --no-check-certificate -qO- $LinuxMirror/$WGETSSL_SUPPORT |tar -x
    mv -f /tmp/boot/usr/bin/wget /tmp/boot/usr/bin/

    [[ ! -f  /tmp/boot/usr/bin/wget2 ]] && echo 'Error! WGETSSL_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
    # sed -i 's/wget\ -qO-/\/usr\/bin\/wget2\ --no-check-certificate\ --retry-connrefused\ --tries=7\ --continue\ -qO-/g' /tmp/boot/preseed.cfg
    # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
  }
    # || {
    # echo -ne 'Not ssl support package! \n\n';
    # exit 1;
    # }
}

#[[ -n "$HTTPD_SUPPORT" ]] && {
  echo -ne 'Add httpd support(webfs binary-amd64 Package)...'

  for pkg in $(echo "${!HTTPD_SUPPORT[@]}" |sed 's/\ /\n/g' |sort -n |grep "^webfs")
    do
      CurPkg="${HTTPD_SUPPORT[$pkg]}"
      [ -n "$CurPkg" ] || continue

      wget --no-check-certificate -qO ${CurPkg##*/} $LinuxMirror/$CurPkg; \
      ar x ${CurPkg##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
      rm -rf data.tar ${CurPkg##*/}
    done

    [[ ! -f  /tmp/boot/usr/bin/webfsd ]] && echo 'Error! HTTPD_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
    # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
#}


echo -e "Downloading kernel : ${LinuxMirror}/....../debian-installer/amd64/initrd.gz"

IncFirmware='0'

wget --no-check-certificate -qO '/boot/initrd.img' "${LinuxMirror}/dists/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz"
[[ $? -ne '0' ]] && echo -ne "\033[31mError! \033[0mDownload 'initrd.img' for \033[33mdebian\033[0m failed! \n" && exit 1
wget --no-check-certificate -qO '/boot/vmlinuz' "${LinuxMirror}/dists/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux"
[[ $? -ne '0' ]] && echo -ne "\033[31mError! \033[0mDownload 'vmlinuz' for \033[33mdebian\033[0m failed! \n" && exit 1
MirrorHost="$(echo "$LinuxMirror" |awk -F'://|/' '{print $2}')";
MirrorFolder="$(echo "$LinuxMirror" |awk -F''${MirrorHost}'' '{print $2}')";

if [[ "$IncFirmware" == '1' ]]; then
  wget --no-check-certificate -qO '/boot/firmware.cpio.gz' "http://cdimage.debian.org/cdimage/unofficial/non-free/firmware/jessie/current/firmware.cpio.gz"
  [[ $? -ne '0' ]] && echo -ne "\033[31mError! \033[0mDownload 'firmware' for \033[33mdebian\033[0m failed! \n" && exit 1
fi

vKernel_udeb=$(wget --no-check-certificate -qO- "http://jessieMirror/dists/$DIST/main/installer-amd64/current/images/udeb.list" |grep '^acpi-modules' |head -n1 |grep -o '[0-9]\{1,2\}.[0-9]\{1,2\}.[0-9]\{1,2\}-[0-9]\{1,2\}' |head -n1)
[[ -z "vKernel_udeb" ]] && vKernel_udeb="3.16.0-6"

echo -e "Downloading debs : ${LinuxMirror}/pool/....."

IncUdebrepo='1'

if [[ "$IncUdebrepo" == '1' ]]; then

  mkdir -p /tmp/boot/var/log/debian;

  udeburl=".*pool\/main\(.*\)udeb.*"
  wget --no-check-certificate -qO- "$LinuxMirror/dists/jessie/main/debian-installer/binary-amd64/Packages.gz" |gunzip -dc|sed "/$udeburl/!d"|sed "s/Filename: //g"|while read line
  do
    path=${line%/*}
    mkdir -p /tmp/boot/var/log/debian/$path
    file=${line##*/}
    wget --no-check-certificate -qO /tmp/boot/var/log/debian/$path/$file $LinuxMirror/$line
  done

  mkdir -p /tmp/boot/var/log/debian/dists/jessie/main/binary-amd64/
  mkdir -p /tmp/boot/var/log/debian/dists/jessie/main/debian-installer/binary-amd64/
  mkdir -p /tmp/boot/var/log/debian/dists/jessie/main/installer-amd64/current/images/

  wget --no-check-certificate -qO /tmp/boot/var/log/debian/dists/jessie/Release $LinuxMirror/dists/jessie/Release
  wget --no-check-certificate -qO /tmp/boot/var/log/debian/dists/jessie/main/binary-amd64/Release $LinuxMirror/dists/jessie/main/binary-amd64/Release
  wget --no-check-certificate -qO /tmp/boot/var/log/debian/dists/jessie/main/debian-installer/binary-amd64/Release $LinuxMirror/dists/jessie/main/debian-installer/binary-amd64/Release,同路径下还有一个Packages.gz
  wget --no-check-certificate -qO /tmp/boot/var/log/debian/dists/jessie/main/installer-amd64/current/images/udeb.list $LinuxMirror/dists/jessie/main/installer-amd64/current/images/udeb.list

  chmod -R 0644 /tmp/boot/var/log/debian/
fi
```


————

脚本版权归原作者所有。还可以像群晖web assit一样,https://wiki.debian.org/DebianInstaller/WebInstaller。preseed还可用于网络和qemu启动


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336149/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



