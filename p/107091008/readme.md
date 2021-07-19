利用onedrive加packerpebuilder实现本地网络统一装机
=====

__本文关键字：bash line string to array, sed 按匹配次数替换，sed替换指定次数的某字符串,bash变量替换，即时赋新值，bash 数组下标变量__

在《利用installnet制作一个云装机packerpe》1,2中我们谈到了packerpebuilder.sh用于云主机装机的用处，它的原理是产生一个基于d-i的pe，然后这个pe会使用当初封装的dd url去网络下载镜像并dd到硬件，在《利用od做站和nas》我们谈到了其与终端作pcmate,mobilemate的那些方面（这个系列只讲了做站，即配合client pc as page server mate的方面---想象一下，在客户端我们可以用servering pages，只是拥有了网盘backend的page server app才真正使远程和本地成为一对mates，产生本地远程合一的mate app，未来我们还会讲配合client pc作离线下载，网盘转存等mateable的方面），今天，我们将用onedrve结合packerpebuilder实现本地也能像云主机一样装机，使远程成为本地装机app，实际上这个思路自packerpebuilder一开始就有了，只是一直没有找到合适能用的网盘。直到对onedrive的直链有了研究之后，才有了本文。

不废话了


前置改动
-----

把tmpmirror也消除了。调整了一些注释，如整合checkdeps和selectmirror为prepare prerequisites，selectmirror经过重构变成select1stvalidmirrorfrom3():

```
function Select1stValidMirrorFrom3(){

  [ $# -ge 1 ] || exit 1

  declare -A MirrorTocheck
  MirrorTocheck=(["Debian0"]="" ["Debian1"]="" ["Debian2"]="")
  
  echo "$1" |sed 's/\ //g' |grep -q '^http://\|^https://\|^ftp://' && MirrorTocheck[Debian0]=$(echo "$1" |sed 's/\ //g');
  echo "$2" |sed 's/\ //g' |grep -q '^http://\|^https://\|^ftp://' && MirrorTocheck[Debian1]=$(echo "$2" |sed 's/\ //g');
  echo "$3" |sed 's/\ //g' |grep -q '^http://\|^https://\|^ftp://' && MirrorTocheck[Debian2]=$(echo "$3" |sed 's/\ //g');

  for mirror in $(echo "${!MirrorTocheck[@]}" |sed 's/\ /\n/g' |sort -n |grep "^Debian")
    do
      CurMirror="${MirrorTocheck[$mirror]}"

      [ -n "$CurMirror" ] || continue

      # CheckPass1='0';
      # DistsList="$(wget --no-check-certificate -qO- "$CurMirror/dists/" |grep -o 'href=.*/"' |cut -d'"' -f2 |sed '/-\|old\|Debian\|experimental\|stable\|test\|sid\|devel/d' |grep '^[^/]' |sed -n '1h;1!H;$g;s/\n//g;s/\//\;/g;$p')";
      # for DIST in `echo "$DistsList" |sed 's/;/\n/g'`
        # do
          # [[ "$DIST" == "jessie" ]] && CheckPass1='1' && break;
        # done
      # [[ "$CheckPass1" == '0' ]] && {
        # echo -ne '\njessie not find in $CurMirror/dists/, Please check it! \n\n'
        # bash $0 error;
        # exit 1;
      # }

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

MIRROR=$(Select1stValidMirrorFrom3 'http://httpredir.debian.org/debian' 'http://www.shalol.com/cn/d/debian' 'http://http://archive.debian.org/debian')
[ -n "$MIRROR" ] && echo -en "Select Mirror ......:" && echo -en "[\033[32m ${MIRROR} \033[0m]\n" || exit 1

```

主体改动
-----

整合prepare  parepare dist files包括downloading basic kernel and rootfs files（将它提前，逻辑更合理。）和downloading repo pkgs files，，以及接下来的PrepareDDessentials(其原来内部下载deb的逻辑整合到与下载full udeb一起)，，并将它们都变成可复用的函数和函数调用buildrepo()和PrepareDDessentials().

prepare  parepare dist files与prepare others是并列的：前三者是大资源文件，后三者是小参数文件，将二者中间延时变量变成2s，各内部延时3s（内部还去掉了细节方面，肯定情况下的一些echo输出，改为直接exit 1，改为由主要的几句话来echo，界面输出更清），共5s

wget要调用ssl client才能tls certificate已完善，buildrepo()更强大，支持sed "s/\(+\|~\)/-/g"处理链接中的+号和~号(tcb上的onemanager不支持这类特殊符号)，和更强大更逻辑清楚的拉取安装deb pkgs支持:

```
IncPkgrepo='1'

declare -A OPTPKGS
OPTPKGS=(
  ["libc1"]="pool/main/g/glibc/libc6_2.28-10_amd64.deb"
  ["fmtlibc"]="xz"
  ["binlibc"]=""

  ["common1"]="pool/main/g/gnutls28/libgnutls30_3.6.7-4+deb10u3_amd64.deb"
  ["common2"]="pool/main/p/p11-kit/libp11-kit0_0.23.15-2_amd64.deb"
  ["common3"]="pool/main/libt/libtasn1-6/libtasn1-6_4.13-3_amd64.deb"
  ["common4"]="pool/main/n/nettle/libnettle6_3.4.1-1_amd64.deb"
  ["common5"]="pool/main/n/nettle/libhogweed4_3.4.1-1_amd64.deb"
  ["common6"]="pool/main/g/gmp/libgmp10_6.1.2+dfsg-4_amd64.deb"
  ["fmtcommon"]="xz"
  ["bincommon"]=""

  ["busybox1"]="pool/main/b/busybox/busybox_1.30.1-4_amd64.deb"
  ["fmtbusybox"]="xz"
  ["binbusybox"]="bin/busybox"

  ["wgetssl1"]="pool/main/libi/libidn2/libidn2-0_2.0.5-1+deb10u1_amd64.deb"
  ["wgetssl2"]="pool/main/libp/libpsl/libpsl5_0.20.2-2_amd64.deb"
  ["wgetssl3"]="pool/main/p/pcre2/libpcre2-8-0_10.32-5_amd64.deb"
  ["wgetssl4"]="pool/main/u/util-linux/libuuid1_2.33.1-0.1_amd64.deb"
  ["wgetssl5"]="pool/main/z/zlib/zlib1g_1.2.11.dfsg-1_amd64.deb"
  ["wgetssl6"]="pool/main/o/openssl/libssl1.0.0_1.0.1t-1+deb8u8_amd64.deb"
  ["wgetssl7"]="pool/main/o/openssl/openssl_1.0.1t-1+deb8u8_amd64.deb"
  ["wgetssl8"]="pool/main/w/wget/wget_1.20.1-1.1_amd64.deb"
  ["fmtwgetssl"]="xz"
  ["binwgetssl"]="usr/bin/wget"

  ["webfs1"]="pool/main/libf/libffi/libffi6_3.2.1-9_amd64.deb"
  ["webfs2"]="pool/main/m/mime-support/mime-support_3.62_all.deb"
  ["webfs3"]="pool/main/libu/libunistring/libunistring2_0.9.10-1_amd64.deb"
  ["webfs4"]="pool/main/w/webfs/webfs_1.21-ds1-12_amd64.deb"
  ["fmtwebfs"]="xz"
  ["binwebfs"]=""
)

function buildrepo(){

  if [[ "$IncPkgrepo" == '1' ]]; then

    echo -e "Downloading full udebs pkg files..... [\033[32m ${MIRROR}/dists/jessie/main/debian-installer/binary-amd64/Packages.gz \033[0m]\n"

    repodir='/tmp/boot/var/log/debian'
    mkdir -p $repodir

    udeburl=".*pool\/main\(.*\)udeb.*"
    wget --no-check-certificate -qO- "$MIRROR/dists/jessie/main/debian-installer/binary-amd64/Packages.gz" |gunzip -dc|sed "/$udeburl/!d"|sed "s/Filename: //g"|while read line
    do
      path=${line%/*}
      mkdir -p $repodir/$path
      file=${line##*/}
      wget --no-check-certificate -qO $repodir/$path/$(echo $file|sed "s/\(+\|~\)/-/g") $MIRROR/$line
    done

    mkdir -p $repodir/dists/jessie/main/binary-amd64/
    mkdir -p $repodir/dists/jessie/main/debian-installer/binary-amd64/
    mkdir -p $repodir/dists/jessie/main/installer-amd64/current/images/

    wget --no-check-certificate -qO $repodir/dists/jessie/Release $MIRROR/dists/jessie/Release
    wget --no-check-certificate -qO $repodir/dists/jessie/main/binary-amd64/Release $MIRROR/dists/jessie/main/binary-amd64/Release
    wget --no-check-certificate -qO $repodir/dists/jessie/main/debian-installer/binary-amd64/Release $MIRROR/dists/jessie/main/debian-installer/binary-amd64/Release
    wget --no-check-certificate -qO $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz $MIRROR/dists/jessie/main/debian-installer/binary-amd64/Packages.gz; \
    orisize=$(cat $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz | wc -c); \
    orimd5=$(md5sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }'); \
    orisha1=$(sha1sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }'); \
    orisha256=$(sha256sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }'); \
    gunzip -c $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz > $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages; \
    rm -rf $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz; \
    sed -i "s/\(+\|~\)/-/g" $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages; \
    gzip -c $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages > $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz; \
    rm -rf $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages; \
    cursize=$(cat $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz | wc -c); \
    curmd5=$(md5sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }'); \
    cursha1=$(sha1sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }'); \
    cursha256=$(sha256sum $repodir/dists/jessie/main/debian-installer/binary-amd64/Packages.gz| awk '{ print $1 }')

    toreplace="main\/debian-installer\/binary-amd64\/Packages.gz"
    linenoarray=($(grep -n $toreplace $repodir/dists/jessie/Release |cut -f1 -d:))

    sed -i ${linenoarray[0]}s/$orimd5/$curmd5/ $repodir/dists/jessie/Release
    sed -i ${linenoarray[0]}s/$orisize/$cursize/ $repodir/dists/jessie/Release
    sed -i ${linenoarray[1]}s/$orisha1/$cursha1/ $repodir/dists/jessie/Release
    sed -i ${linenoarray[1]}s/$orisize/$cursize/ $repodir/dists/jessie/Release
    sed -i ${linenoarray[2]}s/$orisha256/$cursha256/ $repodir/dists/jessie/Release
    sed -i ${linenoarray[2]}s/$orisize/$cursize/ $repodir/dists/jessie/Release

    wget --no-check-certificate -qO $repodir/dists/jessie/main/installer-amd64/current/images/udeb.list $MIRROR/dists/jessie/main/installer-amd64/current/images/udeb.list

    chmod -R 0644 $repodir/
  fi

  echo -e "Downloading optional deb pkg files...... [\033[32m ${MIRROR}/dists/jessie/main/binary-amd64/Packages.gz \033[0m]\n";

  for pkg in `echo "$1" |sed 's/,/\n/g'`
    do
    
      [[ -n "${OPTPKGS[$pkg"1"]}" ]] && {
        for subpkg in $(echo "${!OPTPKGS[@]}" |sed 's/\ /\n/g' |sort -n |grep "^$pkg")
          do
            cursubpkgfile="${OPTPKGS[$subpkg]}"
            [ -n "$cursubpkgfile" ] || continue

            cursubpkgfilepath=${cursubpkgfile%/*}
            mkdir -p $repodir/$cursubpkgfilepath
            cursubpkgfilename=${cursubpkgfile##*/}
            cursubpkgfilename2=$(echo $cursubpkgfilename|sed "s/\(+\|~\)/-/g")

            wget --no-check-certificate -qO $repodir/$cursubpkgfilepath/$cursubpkgfilename2 $MIRROR/$cursubpkgfile; \
            [[ "${OPTPKGS["fmt"$pkg]}" == "tar" ]] && ar x $repodir/$cursubpkgfilepath/$cursubpkgfilename2 data.tar.gz && tar xzf data.tar.gz && rm -rf data.tar.gz
            [[ "${OPTPKGS["fmt"$pkg]}" == "xz" ]] && ar x $repodir/$cursubpkgfilepath/$cursubpkgfilename2 data.tar.xz && xz -d data.tar.xz && tar xf data.tar && rm -rf data.tar

          done
            [[ -n "${OPTPKGS["bin"$pkg]}" ]] && mv -f /tmp/boot/${OPTPKGS["bin"$pkg]} /tmp/boot/${OPTPKGS["bin"$pkg]}2
            # [[ ! -f  /tmp/boot/${OPTPKGS["bin"$pkg]}2 ]] && echo 'Error! $1 SUPPORT ERROR.' && exit 1;
      }

    done

}
buildrepo libc,common,busybox,wgetssl;
```

PrepareDDessentials()也更强大，支持sharepoint和office365个人的302跳转风格，强化《利用installnet制作一个云装机packerpe》2中关于仅支持office365style相关方面功能 --- 其实sharepointstyle和office365 style也可自动公判断，但是我不想折腾了。

```
UNZIP=''
DDURL=''
OFFICE365STYLE='0'
SHAREPOINTSTYLE='1'

function PrepareDDessentials(){

  if [[ -n "$tmpDDURL" ]]; then
    echo "$tmpDDURL" |grep -q '^http://\|^ftp://\|^https://';
    [[ $? -ne '0' ]] && echo 'No valid URL in the DD argument,Only support http://, ftp:// and https:// !' && exit 1;

    IMGHEADER="$(curl -Is "$tmpDDURL")";
    IMGTYPE="$(echo "$IMGHEADER" | grep -E -o '200|302')" || IMGTYPE='0';

    # [[ "$IMGTYPE" -ne '0' ]] && echo 'not a raw,tar,gunzip or 301/302 ref file, exit ... !' && exit 1 || {

      [[ "$IMGTYPE" == '200' ]] && IMGHEADERCHECKPASS2="$(echo "$IMGHEADER" |grep -E -o 'raw|qcow2|gzip|x-gzip')" && {
        [[ "$IMGTYPECHECKPASS2" == 'raw' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS2" == 'qcow2' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS2" == 'gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS2" == 'x-gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS2" == 'gunzip' ]] && UNZIP='2' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
      }

      [[ "$IMGTYPE" == '302' && "$OFFICE365STYLE" == '1' ]] && { \

        IMGHEADERCHECKPASS2="$(echo "$IMGHEADER" |grep 'Location: http'|sed 's/Location: //g')" && IMGHEADERCHECKPASS2=${IMGHEADERCHECKPASS2%$'\r'} && \
        IMGHEADERCHECKPASS3="$(curl -Is "$IMGHEADERCHECKPASS2" |grep 'content-location: http'|sed 's/content-location: //g')" && IMGHEADERCHECKPASS3=${IMGHEADERCHECKPASS3%$'\r'} && \

        IMGTYPECHECKPASS2="$(curl -Is "$IMGHEADERCHECKPASS3" | grep -E -o 'raw|qcow2|gzip|x-gzip|gunzip')" && {
          [[ "$IMGTYPECHECKPASS2" == 'raw' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'qcow2' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'x-gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'gunzip' ]] && UNZIP='2' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
        }
      }



      [[ "$IMGTYPE" == '302' && "$SHAREPOINTSTYLE" == '1' ]] && { \

        IMGHEADERCHECKPASS2="$(echo "$IMGHEADER" |grep 'Location: http'|sed 's/Location: //g')" && IMGHEADERCHECKPASS2=${IMGHEADERCHECKPASS2%$'\r'} && \

        IMGTYPECHECKPASS2="$(curl -Is "$IMGHEADERCHECKPASS2" | grep -E -o 'raw|qcow2|gzip|x-gzip|gunzip')" && {
          [[ "$IMGTYPECHECKPASS2" == 'raw' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'qcow2' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'x-gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
          [[ "$IMGTYPECHECKPASS2" == 'gunzip' ]] && UNZIP='2' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
        }
      }



      [[ "$UNZIP" == '' ]] && echo 'didnt got a unzip mode, exit ... !' && exit 1;
      [[ "$DDURL" == '' ]] && echo 'didnt got a ddurl, exit ... !' && exit 1;

      [[ "$IMGTYPE" == '0' ]] && echo 'not a raw,tar,gunzip or 301/302 ref file, exit ... !' && exit 1;
    # }

  else
    echo 'Please input vaild image URL! ';
    exit 1;
  fi

}
echo -e 'prepare DDessentials ......';
PrepareDDessentials;
sleep 3s
```


使用方法
------

中途提示备份,会给你30s上传/tmp/boot/var/log/debian仓库到onedrive或其它服务器创建镜像，。

```
  [[ "$tmpINSTANTWITHOUTVNC" == '0' ]] &&  echo "finished, auto reboot after 30s...(if needed, you can press ctrl c to interrupt to bak the repodir:$repodir, then manually reboot to continue)";sleep 30s
```

preseed中的mirrorhost换成你的od上传地址。

然后就然后了。。。

------


未来，我们要往packerpebuilder中集成不死booter。这样装机永远都不会因为抹了第一个硬盘哭了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107091008/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



