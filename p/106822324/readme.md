把DI当online packer用:利用installnet制作一个云装机packerpe（2）
=====

__本文关键字：自建udeb站镜像。local repository debian installer preseed,Preseed install from local repository,start http server in a preseed file,debian中用NC+SHELL实现的一个web服务,浏览器装机,Preseed failed with exit 2,d-i preseed/early_command string stuck hang,linux grep -o 只匹配字串__

继把Debianinstaller当online packer用文章1，本文继续讲解生成packerpe。

给前面的脚本增加和镜像格式判断，统一wget和dd progress
-----

增强主要是为了pipecmdstr处整合，稍后会看到(因此也顺便支持了onelist,oneindex这种网盘外链作为ddurl地址，其实质是302跳转体)。这段替换前文的cd /tmp/boot;到echo -e "Downloading kernel : ${LinuxMirror}/....../debian-installer/amd64/initrd.gz"间的内容：

```
LIBC6_SUPPORT='pool/main/g/glibc/libc6_2.28-10_amd64.deb'
declare -A SHARED_SUPPORT
SHARED_SUPPORT=(
["webfs1"]="pool/main/g/gnutls28/libgnutls30_3.6.7-4+deb10u3_amd64.deb"
["webfs2"]="pool/main/p/p11-kit/libp11-kit0_0.23.15-2_amd64.deb"
["webfs3"]="pool/main/libt/libtasn1-6/libtasn1-6_4.13-3_amd64.deb"
["webfs4"]="pool/main/n/nettle/libnettle6_3.4.1-1_amd64.deb"
["webfs5"]="pool/main/n/nettle/libhogweed4_3.4.1-1_amd64.deb"
["webfs6"]="pool/main/g/gmp/libgmp10_6.1.2+dfsg-4_amd64.deb"

["webfs7"]="pool/main/libf/libffi/libffi6_3.2.1-9_amd64.deb"
["webfs8"]="pool/main/m/mime-support/mime-support_3.62_all.deb"
["webfs9"]="pool/main/libu/libunistring/libunistring2_0.9.10-1_amd64.deb"

["wgetssl7"]="pool/main/libi/libidn2/libidn2-0_2.0.5-1+deb10u1_amd64.deb"
["wgetssl8"]="pool/main/libp/libpsl/libpsl5_0.20.2-2_amd64.deb"
["wgetssl9"]="pool/main/p/pcre2/libpcre2-8-0_10.32-5_amd64.deb"
["wgetssl10"]="pool/main/u/util-linux/libuuid1_2.33.1-0.1_amd64.deb"
["wgetssl11"]="pool/main/z/zlib/zlib1g_1.2.11.dfsg-1_amd64.deb"
)

HTTPD_SUPPORT='pool/main/w/webfs/webfs_1.21+ds1-12_amd64.deb'
WGETSSL_SUPPORT='pool/main/w/wget/wget_1.20.1-1.1_amd64.deb'
DDPROGRESS_SUPPORT='pool/main/b/busybox/busybox_1.30.1-4_amd64.deb'
UNZIP=''
DDURL=''

[[ -n "$LIBC6_SUPPORT" ]] && {
  echo -ne 'Add libc6 support(binary-amd64/Package)...'

  wget --no-check-certificate -qO ${LIBC6_SUPPORT##*/} $LinuxMirror/$LIBC6_SUPPORT; \
  ar x ${LIBC6_SUPPORT##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
  rm -rf data.tar ${LIBC6_SUPPORT##*/}

  [[ ! -f  /tmp/boot/lib/x86_64-linux-gnu/libc.so.6 ]] && echo 'Error! LIBC6_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
  # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
}

[[ -n "$HTTPD_SUPPORT" ]] && {
  echo -ne 'Add httpd and httpd shared support(binary-amd64/Package)...'

  wget --no-check-certificate -qO ${HTTPD_SUPPORT##*/} $LinuxMirror/$HTTPD_SUPPORT; \
  ar x ${HTTPD_SUPPORT##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
  rm -rf data.tar ${HTTPD_SUPPORT##*/}

  for pkg in $(echo "${!SHARED_SUPPORT[@]}" |sed 's/\ /\n/g' |sort -n |grep "^webfs")
    do
      CurPkg="${SHARED_SUPPORT[$pkg]}"
      [ -n "$CurPkg" ] || continue

      wget --no-check-certificate -qO ${CurPkg##*/} $LinuxMirror/$CurPkg; \
      ar x ${CurPkg##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
      rm -rf data.tar ${CurPkg##*/}
    done

  [[ ! -f  /tmp/boot/usr/bin/webfsd ]] && echo 'Error! HTTPD_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
  # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
}

if [[ -n "$tmpDDURL" ]]; then
  echo -ne 'setting final ddurl and prepare unzip mode...'
  echo "$tmpDDURL" |grep -q '^http://\|^ftp://\|^https://';
  [[ $? -ne '0' ]] && echo 'No valid URL in the DD argument,Only support http://, ftp:// and https:// !' && exit 1;

  IMGHEADER="$(curl -Is "$tmpDDURL")";
  IMGTYPE="$(echo "$IMGHEADER" | grep -E -o 'raw|qcow2|gzip|x-gzip|301|302')" || IMGTYPE='0';
# [[ "$IMGTYPE" -ne '0' ]] && echo 'not a raw,tar,gunzip or 301/302 ref file, exit ... !' && exit 1 || {

    [[ "$IMGTYPE" == 'raw' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]\n";
    [[ "$IMGTYPE" == 'qcow2' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]\n";
    [[ "$IMGTYPE" == 'gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gzip \033[0m]\n";
    [[ "$IMGTYPE" == 'x-gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]\n";
    [[ "$IMGTYPE" == 'gunzip' ]] && UNZIP='2' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gunzip \033[0m]\n";

    [[ "$IMGTYPE" == '302' ]] && IMGHEADERCHECKPASS2="$(echo "$IMGHEADER" |grep 'Location: http'|sed 's/Location: //g')" && IMGHEADERCHECKPASS2=${IMGHEADERCHECKPASS2%$'\r'} && IMGHEADERCHECKPASS3="$(curl -Is "$IMGHEADERCHECKPASS2" |grep 'content-location: http'|sed 's/content-location: //g')" && IMGHEADERCHECKPASS3=${IMGHEADERCHECKPASS3%$'\r'} && IMGTYPECHECKPASS2="$(curl -Is "$IMGHEADERCHECKPASS3" | grep -E -o 'raw|qcow2|gzip|x-gzip|gunzip')" && {
      [[ "$IMGTYPECHECKPASS2" == 'raw' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]\n";
      [[ "$IMGTYPECHECKPASS2" == 'qcow2' ]] && UNZIP='0' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m raw \033[0m]\n";
      [[ "$IMGTYPECHECKPASS2" == 'gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gzip \033[0m]\n";
      [[ "$IMGTYPECHECKPASS2" == 'x-gzip' ]] && UNZIP='1' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]\n";
      [[ "$IMGTYPECHECKPASS2" == 'gunzip' ]] && UNZIP='2' && DDURL="$tmpDDURL" && sleep 3s && echo -en "[\033[32m gunzip \033[0m]\n";
    }

    [[ "$IMGTYPE" == '0' ]] && echo 'not a raw,tar,gunzip or 301/302 ref file, exit ... !' && exit 1;
# }

else
  echo 'Please input vaild image URL! ';
  exit 1;
fi

#echo "$DDURL" |grep -q '^https://'
#[[ $? -eq '0' ]] && {

  [[ -n "$WGETSSL_SUPPORT" ]] && {
    echo -ne 'always Add wgetssl and wgetssl support(binary-amd64/Package)...'

    wget --no-check-certificate -qO ${WGETSSL_SUPPORT##*/} $LinuxMirror/$WGETSSL_SUPPORT; \
    ar x ${WGETSSL_SUPPORT##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
    rm -rf data.tar ${WGETSSL_SUPPORT##*/}; mv -f /tmp/boot/usr/bin/wget /tmp/boot/usr/bin/wget2

    for pkg in $(echo "${!SHARED_SUPPORT[@]}" |sed 's/\ /\n/g' |sort -n |grep "^wgetssl")
      do
        CurPkg="${SHARED_SUPPORT[$pkg]}"
        [ -n "$CurPkg" ] || continue

        wget --no-check-certificate -qO ${CurPkg##*/} $LinuxMirror/$CurPkg; \
        ar x ${CurPkg##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
        rm -rf data.tar ${CurPkg##*/}
      done

    [[ ! -f  /tmp/boot/usr/bin/wget2 ]] && echo 'Error! WGETSSL_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
    # sed -i 's/wget\ -qO-/\/usr\/bin\/wget2\ --no-check-certificate\ --retry-connrefused\ --tries=7\ --continue\ -qO-/g' /tmp/boot/preseed.cfg
    # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
  }
    # || {
    # echo -ne 'Not wgetssl support package! \n\n';
    # exit 1;
    # }
#}

[[ -n "$DDPROGRESS_SUPPORT" ]] && {
  echo -ne 'Add ddprogress support(binary-amd64/Package)...'

  wget --no-check-certificate -qO ${DDPROGRESS_SUPPORT##*/} $LinuxMirror/$DDPROGRESS_SUPPORT; \
  ar x ${DDPROGRESS_SUPPORT##*/} data.tar.xz; xz -d data.tar.xz; tar xf data.tar; \
  rm -rf data.tar ${DDPROGRESS_SUPPORT##*/}

  [[ ! -f  /tmp/boot/bin/busybox ]] && echo 'Error! DDPROGRESS_SUPPORT.' && exit 1 || sleep 3s && echo -en "[\033[32mok\033[0m]\n" ;
  # [[ $? -eq '0' ]] && echo -ne 'Success! \n\n'
}
```

如果你发现启动不了文件系统出现制作文件系统时出现的那些错误，那就是可能glibc低了。注意到dd status progress我并未在接下的preseed中用到。

整合精简几大件,the net, the loader, the preseed file
-----

下面是脚本的继续,准备net设置，grub loader和preseed file供打包：

```
sleep 5s

echo -e "\n\033[36m# Preparing others\033[0m\n"

echo -e "the net"

setNet='0'
interface=''

[ -n "$ipAddr" ] && [ -n "$ipMask" ] && [ -n "$ipGate" ] && setNet='1';
[[ -n "$tmpWORD" ]] && myPASSWORD="$(openssl passwd -1 "$tmpWORD")";
[[ -z "$myPASSWORD" ]] && myPASSWORD='$1$4BJZaD0A$y1QykUnJ6mXprENfwpseH0';

if [[ -n "$interface" ]]; then
  IFETH="$interface"
else
  IFETH="auto"
fi

[[ "$setNet" == '1' ]] && {
  IPv4="$ipAddr";
  MASK="$ipMask";
  GATE="$ipGate";
} || {
  DEFAULTNET="$(ip route show |grep -o 'default via [0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.*' |head -n1 |sed 's/proto.*\|onlink.*//g' |awk '{print $NF}')";
  [[ -n "$DEFAULTNET" ]] && IPSUB="$(ip addr |grep ''${DEFAULTNET}'' |grep 'global' |grep 'brd' |head -n1 |grep -o '[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}/[0-9]\{1,2\}')";
  IPv4="$(echo -n "$IPSUB" |cut -d'/' -f1)";
  NETSUB="$(echo -n "$IPSUB" |grep -o '/[0-9]\{1,2\}')";
  GATE="$(ip route show |grep -o 'default via [0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}' |head -n1 |grep -o '[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}')";
  [[ -n "$NETSUB" ]] && MASK="$(echo -n '128.0.0.0/1,192.0.0.0/2,224.0.0.0/3,240.0.0.0/4,248.0.0.0/5,252.0.0.0/6,254.0.0.0/7,255.0.0.0/8,255.128.0.0/9,255.192.0.0/10,255.224.0.0/11,255.240.0.0/12,255.248.0.0/13,255.252.0.0/14,255.254.0.0/15,255.255.0.0/16,255.255.128.0/17,255.255.192.0/18,255.255.224.0/19,255.255.240.0/20,255.255.248.0/21,255.255.252.0/22,255.255.254.0/23,255.255.255.0/24,255.255.255.128/25,255.255.255.192/26,255.255.255.224/27,255.255.255.240/28,255.255.255.248/29,255.255.255.252/30,255.255.255.254/31,255.255.255.255/32' |grep -o '[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}'${NETSUB}'' |cut -d'/' -f1)";
}

[[ -n "$GATE" ]] && [[ -n "$MASK" ]] && [[ -n "$IPv4" ]] || {
  echo "Not found \`ip command\`, It will use \`route command\`."

  ipNum() {
    local IFS='.';
    read ip1 ip2 ip3 ip4 <<<"$1";
    echo $((ip1*(1<<24)+ip2*(1<<16)+ip3*(1<<8)+ip4));
  }

  SelectMax(){
  ii=0;
  for IPITEM in `route -n |awk -v OUT=$1 '{print $OUT}' |grep '[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}'`
    do
      NumTMP="$(ipNum $IPITEM)";
      eval "arrayNum[$ii]='$NumTMP,$IPITEM'";
      ii=$[$ii+1];
    done
  echo ${arrayNum[@]} |sed 's/\s/\n/g' |sort -n -k 1 -t ',' |tail -n1 |cut -d',' -f2;
  }

  [[ -z $IPv4 ]] && IPv4="$(ifconfig |grep 'Bcast' |head -n1 |grep -o '[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}.[0-9]\{1,3\}' |head -n1)";
  [[ -z $GATE ]] && GATE="$(SelectMax 2)";
  [[ -z $MASK ]] && MASK="$(SelectMax 3)";

  [[ -n "$GATE" ]] && [[ -n "$MASK" ]] && [[ -n "$IPv4" ]] || {
    echo "Error! Not configure network. ";
    exit 1;
  }
}

[[ "$setNet" != '1' ]] && [[ -f '/etc/network/interfaces' ]] && {
  [[ -z "$(sed -n '/iface.*inet static/p' /etc/network/interfaces)" ]] && AutoNet='1' || AutoNet='0';
  [[ -d /etc/network/interfaces.d ]] && {
    ICFGN="$(find /etc/network/interfaces.d -name '*.cfg' |wc -l)" || ICFGN='0';
    [[ "$ICFGN" -ne '0' ]] && {
      for NetCFG in `ls -1 /etc/network/interfaces.d/*.cfg`
        do 
          [[ -z "$(cat $NetCFG | sed -n '/iface.*inet static/p')" ]] && AutoNet='1' || AutoNet='0';
          [[ "$AutoNet" -eq '0' ]] && break;
        done
    }
  }
}

[[ "$setNet" != '1' ]] && [[ -d '/etc/sysconfig/network-scripts' ]] && {
  ICFGN="$(find /etc/sysconfig/network-scripts -name 'ifcfg-*' |grep -v 'lo'|wc -l)" || ICFGN='0';
  [[ "$ICFGN" -ne '0' ]] && {
    for NetCFG in `ls -1 /etc/sysconfig/network-scripts/ifcfg-* |grep -v 'lo$' |grep -v ':[0-9]\{1,\}'`
      do 
        [[ -n "$(cat $NetCFG | sed -n '/BOOTPROTO.*[dD][hH][cC][pP]/p')" ]] && AutoNet='1' || {
          AutoNet='0' && . $NetCFG;
          [[ -n $NETMASK ]] && MASK="$NETMASK";
          [[ -n $GATEWAY ]] && GATE="$GATEWAY";
        }
        [[ "$AutoNet" -eq '0' ]] && break;
      done
  }
}

echo -e "the loader"

LoaderMode='0'
setInterfaceName='0'
setIPv6='0'

if [[ "$LoaderMode" == "0" ]]; then
  [[ -f '/boot/grub/grub.cfg' ]] && GRUBVER='0' && GRUBDIR='/boot/grub' && GRUBFILE='grub.cfg';
  [[ -z "$GRUBDIR" ]] && [[ -f '/boot/grub2/grub.cfg' ]] && GRUBVER='0' && GRUBDIR='/boot/grub2' && GRUBFILE='grub.cfg';
  [[ -z "$GRUBDIR" ]] && [[ -f '/boot/grub/grub.conf' ]] && GRUBVER='1' && GRUBDIR='/boot/grub' && GRUBFILE='grub.conf';
  [ -z "$GRUBDIR" -o -z "$GRUBFILE" ] && echo -ne "Error! \nNot Found grub.\n" && exit 1;
else
  tmpINSTANTWITHOUTVNC='0'
fi

if [[ "$LoaderMode" == "0" ]]; then
  [[ ! -f $GRUBDIR/$GRUBFILE ]] && echo "Error! Not Found $GRUBFILE. " && exit 1;

  [[ ! -f $GRUBDIR/$GRUBFILE.old ]] && [[ -f $GRUBDIR/$GRUBFILE.bak ]] && mv -f $GRUBDIR/$GRUBFILE.bak $GRUBDIR/$GRUBFILE.old;
  mv -f $GRUBDIR/$GRUBFILE $GRUBDIR/$GRUBFILE.bak;
  [[ -f $GRUBDIR/$GRUBFILE.old ]] && cat $GRUBDIR/$GRUBFILE.old >$GRUBDIR/$GRUBFILE || cat $GRUBDIR/$GRUBFILE.bak >$GRUBDIR/$GRUBFILE;
else
  GRUBVER='2'
fi


[[ "$GRUBVER" == '0' ]] && {
  READGRUB='/tmp/grub.read'
  cat $GRUBDIR/$GRUBFILE |sed -n '1h;1!H;$g;s/\n/%%%%%%%/g;$p' |grep -om 1 'menuentry\ [^{]*{[^}]*}%%%%%%%' |sed 's/%%%%%%%/\n/g' >$READGRUB
  LoadNum="$(cat $READGRUB |grep -c 'menuentry ')"
  if [[ "$LoadNum" -eq '1' ]]; then
    cat $READGRUB |sed '/^$/d' >/tmp/grub.new;
  elif [[ "$LoadNum" -gt '1' ]]; then
    CFG0="$(awk '/menuentry /{print NR}' $READGRUB|head -n 1)";
    CFG2="$(awk '/menuentry /{print NR}' $READGRUB|head -n 2 |tail -n 1)";
    CFG1="";
    for tmpCFG in `awk '/}/{print NR}' $READGRUB`
      do
        [ "$tmpCFG" -gt "$CFG0" -a "$tmpCFG" -lt "$CFG2" ] && CFG1="$tmpCFG";
      done
    [[ -z "$CFG1" ]] && {
      echo "Error! read $GRUBFILE. ";
      exit 1;
    }

    sed -n "$CFG0,$CFG1"p $READGRUB >/tmp/grub.new;
    [[ -f /tmp/grub.new ]] && [[ "$(grep -c '{' /tmp/grub.new)" -eq "$(grep -c '}' /tmp/grub.new)" ]] || {
      echo -ne "\033[31mError! \033[0mNot configure $GRUBFILE. \n";
      exit 1;
    }
  fi
  [ ! -f /tmp/grub.new ] && echo "Error! $GRUBFILE. " && exit 1;
  sed -i "/menuentry.*/c\menuentry\ \'Packer PE \[debian\ jessie\ amd64\]\'\ --class debian\ --class\ gnu-linux\ --class\ gnu\ --class\ os\ \{" /tmp/grub.new
  sed -i "/echo.*Loading/d" /tmp/grub.new;
  INSERTGRUB="$(awk '/menuentry /{print NR}' $GRUBDIR/$GRUBFILE|head -n 1)"
}

[[ "$GRUBVER" == '1' ]] && {
  CFG0="$(awk '/title[\ ]|title[\t]/{print NR}' $GRUBDIR/$GRUBFILE|head -n 1)";
  CFG1="$(awk '/title[\ ]|title[\t]/{print NR}' $GRUBDIR/$GRUBFILE|head -n 2 |tail -n 1)";
  [[ -n $CFG0 ]] && [ -z $CFG1 -o $CFG1 == $CFG0 ] && sed -n "$CFG0,$"p $GRUBDIR/$GRUBFILE >/tmp/grub.new;
  [[ -n $CFG0 ]] && [ -z $CFG1 -o $CFG1 != $CFG0 ] && sed -n "$CFG0,$[$CFG1-1]"p $GRUBDIR/$GRUBFILE >/tmp/grub.new;
  [[ ! -f /tmp/grub.new ]] && echo "Error! configure append $GRUBFILE. " && exit 1;
  sed -i "/title.*/c\title\ \'DebianNetboot \[jessie\ amd64\]\'" /tmp/grub.new;
  sed -i '/^#/d' /tmp/grub.new;
  INSERTGRUB="$(awk '/title[\ ]|title[\t]/{print NR}' $GRUBDIR/$GRUBFILE|head -n 1)"
}

if [[ "$LoaderMode" == "0" ]]; then
  [[ -n "$(grep 'linux.*/\|kernel.*/' /tmp/grub.new |awk '{print $2}' |tail -n 1 |grep '^/boot/')" ]] && Type='InBoot' || Type='NoBoot';

  LinuxKernel="$(grep 'linux.*/\|kernel.*/' /tmp/grub.new |awk '{print $1}' |head -n 1)";
  [[ -z "$LinuxKernel" ]] && echo "Error! read grub config! " && exit 1;
  LinuxIMG="$(grep 'initrd.*/' /tmp/grub.new |awk '{print $1}' |tail -n 1)";
  [ -z "$LinuxIMG" ] && sed -i "/$LinuxKernel.*\//a\\\tinitrd\ \/" /tmp/grub.new && LinuxIMG='initrd';

  if [[ "$setInterfaceName" == "1" ]]; then
    Add_OPTION="net.ifnames=0 biosdevname=0";
  else
    Add_OPTION="";
  fi

  if [[ "$setIPv6" == "1" ]]; then
    Add_OPTION="$Add_OPTION ipv6.disable=1";
  fi

  BOOT_OPTION="auto=true $Add_OPTION hostname=debian domain= -- quiet"

  [[ "$Type" == 'InBoot' ]] && {
    sed -i "/$LinuxKernel.*\//c\\\t$LinuxKernel\\t\/boot\/vmlinuz $BOOT_OPTION" /tmp/grub.new;
    sed -i "/$LinuxIMG.*\//c\\\t$LinuxIMG\\t\/boot\/initrd.img" /tmp/grub.new;
  }

  [[ "$Type" == 'NoBoot' ]] && {
    sed -i "/$LinuxKernel.*\//c\\\t$LinuxKernel\\t\/vmlinuz $BOOT_OPTION" /tmp/grub.new;
    sed -i "/$LinuxIMG.*\//c\\\t$LinuxIMG\\t\/initrd.img" /tmp/grub.new;
  }

  sed -i '$a\\n' /tmp/grub.new;
fi

[[ "$tmpINSTANTWITHOUTVNC" == '0' ]] && {
  GRUBPATCH='0';

  if [[ "$LoaderMode" == "0" ]]; then
  [ -f '/etc/network/interfaces' -o -d '/etc/sysconfig/network-scripts' ] || {
    echo "Error, Not found interfaces config.";
    exit 1;
  }

  sed -i ''${INSERTGRUB}'i\\n' $GRUBDIR/$GRUBFILE;
  sed -i ''${INSERTGRUB}'r /tmp/grub.new' $GRUBDIR/$GRUBFILE;
  [[ -f  $GRUBDIR/grubenv ]] && sed -i 's/saved_entry/#saved_entry/g' $GRUBDIR/grubenv;
  fi

  cd /tmp/boot;
  COMPTYPE="gzip";
  CompDected='0'
  for ListCOMP in `echo -en 'gzip\nlzma\nxz'`
    do
      if [[ "$COMPTYPE" == "$ListCOMP" ]]; then
        CompDected='1'
        if [[ "$COMPTYPE" == 'gzip' ]]; then
          NewIMG="initrd.img.gz"
        else
          NewIMG="initrd.img.$COMPTYPE"
        fi
        mv -f "/boot/initrd.img" "/tmp/$NewIMG"
        break;
      fi
    done
  [[ "$CompDected" != '1' ]] && echo "Detect compressed type not support." && exit 1;
  [[ "$COMPTYPE" == 'lzma' ]] && UNCOMP='xz --format=lzma --decompress';
  [[ "$COMPTYPE" == 'xz' ]] && UNCOMP='xz --decompress';
  [[ "$COMPTYPE" == 'gzip' ]] && UNCOMP='gzip -d';

  $UNCOMP < /tmp/$NewIMG | cpio --extract --verbose --make-directories --no-absolute-filenames >>/dev/null 2>&1

  echo -e "the preseed"

  [[ "$UNZIP" == '0' ]] && PIPECMDSTR='wget2 --no-check-certificate -qO- '$DDURL' |dd of=$(list-devices disk |head -n1)';
  [[ "$UNZIP" == '1' ]] && PIPECMDSTR='wget2 --no-check-certificate -qO- '$DDURL' |tar zOx |dd of=$(list-devices disk |head -n1)';
  [[ "$UNZIP" == '2' ]] && PIPECMDSTR='wget2 --no-check-certificate -qO- '$DDURL' |gzip -dc |dd of=$(list-devices disk |head -n1)';

cat >/tmp/boot/preseed.cfg<<EOF
d-i preseed/early_command string anna-install

d-i debian-installer/locale string en_US
d-i console-setup/layoutcode string us

d-i keyboard-configuration/xkb-keymap string us

d-i hw-detect/load_firmware boolean true

d-i netcfg/choose_interface select $IFETH
d-i netcfg/disable_autoconfig boolean true
d-i netcfg/dhcp_failed note
d-i netcfg/dhcp_options select Configure network manually
# d-i netcfg/get_ipaddress string $ipAddr
d-i netcfg/get_ipaddress string $IPv4
d-i netcfg/get_netmask string $MASK
d-i netcfg/get_gateway string $GATE
d-i netcfg/get_nameservers string 8.8.8.8
d-i netcfg/no_default_route boolean true
d-i netcfg/confirm_static boolean true

d-i mirror/country string manual
#d-i mirror/http/hostname string $IPv4
d-i mirror/http/hostname string httpredir.debian.org
d-i mirror/http/directory string /debian
d-i mirror/http/proxy string

# webfsd -i $IPv4 -p 80 -r /var/log/ -l /var/log/httpd.log

d-i apt-setup/services-select multiselect
d-i debian-installer/allow_unauthenticated boolean true

d-i passwd/root-login boolean ture
d-i passwd/make-user boolean false
d-i passwd/root-password-crypted password $myPASSWORD
d-i user-setup/allow-password-weak boolean true
d-i user-setup/encrypt-home boolean false

d-i clock-setup/utc boolean true
d-i time/zone string US/Eastern
d-i clock-setup/ntp boolean true

d-i partman/early_command string $PIPECMDSTR; /sbin/reboot
EOF

  [[ "$LoaderMode" != "0" ]] && AutoNet='1'

  [[ "$setNet" == '0' ]] && [[ "$AutoNet" == '1' ]] && {
    sed -i '/netcfg\/disable_autoconfig/d' /tmp/boot/preseed.cfg
    sed -i '/netcfg\/dhcp_options/d' /tmp/boot/preseed.cfg
    sed -i '/netcfg\/get_.*/d' /tmp/boot/preseed.cfg
    sed -i '/netcfg\/confirm_static/d' /tmp/boot/preseed.cfg
  }

  #[[ "$GRUBPATCH" == '1' ]] && {
  #  sed -i 's/^d-i\ grub-installer\/bootdev\ string\ default//g' /tmp/boot/preseed.cfg
  #}
  #[[ "$GRUBPATCH" == '0' ]] && {
  #  sed -i 's/debconf-set\ grub-installer\/bootdev.*\"\;//g' /tmp/boot/preseed.cfg
  #}

  sed -i '/user-setup\/allow-password-weak/d' /tmp/boot/preseed.cfg
  sed -i '/user-setup\/encrypt-home/d' /tmp/boot/preseed.cfg
  #sed -i '/pkgsel\/update-policy/d' /tmp/boot/preseed.cfg
  #sed -i 's/umount\ \/media.*true\;\ //g' /tmp/boot/preseed.cfg

  [[ -f '/boot/firmware.cpio.gz' ]] && {
    gzip -d < /boot/firmware.cpio.gz | cpio --extract --verbose --make-directories --no-absolute-filenames >>/dev/null 2>&1
  }
```

注意上面的脚本中输出至preseed file至EOF止的内容必须顶格写，Ks.cf是installment.sh用来用同样ubuntu支持preseed支持centos的方法。已删，在文件中，我注释了原来使用内置的webfsd服务器作镜像的方案，因为preseed early command string中欠入webfsd启动命令（放anna install前），会导致PE启动时early command运行时，umount时前台却在加载webfsd卡住，需要ctrlc才能继续，但实际上webfsd已成功运行。

所以我将注释webfsd的逻辑放在preseed中某位置，这个位置是它应该启动时的时机。实际上early command string中都不比它合适。未来想办法让它以正确方式起作用。

打包
-----

```
  sleep 5s

  echo -e "\n\033[36m# Packaging \033[0m\n"

  find . | cpio -H newc --create --quiet | gzip -9 > /boot/initrd.img;
  [[ "$tmpINSTANTWITHOUTVNC" == '0' ]] &&  echo "finished, auto reboot after 30s...(if needed, you can press ctrl c to interrupt to bak the /boot/initrd.img, then manually reboot to continue)";sleep 30s

  rm -rf /tmp/boot;

}

[[ "$tmpINSTANTWITHOUTVNC" == '1' ]] && {
  sed -i '$i\\n' $GRUBDIR/$GRUBFILE
  sed -i '$r /tmp/grub.new' $GRUBDIR/$GRUBFILE
  echo -e "\n\033[33m\033[04mIt will reboot! \nPlease connect VNC! \nSelect\033[0m\033[32m Packer PE [debian jessie amd64] \033[33m\033[4mto install system.\033[04m\n\n\033[31m\033[04mThere is some information for you.\nDO NOT CLOSE THE WINDOW! \033[0m\n"
  echo -e "\033[35mIPv4\t\tNETMASK\t\tGATEWAY\033[0m"
  echo -e "\033[36m\033[04m$IPv4\033[0m\t\033[36m\033[04m$MASK\033[0m\t\033[36m\033[04m$GATE\033[0m\n\n"

  read -n 1 -p "Press Enter to reboot..." INP
  [[ "$INP" != '' ]] && echo -ne '\b \n\n';
}

chown root:root $GRUBDIR/$GRUBFILE
chmod 444 $GRUBDIR/$GRUBFILE


if [[ "$LoaderMode" == "0" ]]; then
  sleep 3 && reboot >/dev/null 2>&1
else
  rm -rf "$HOME/loader"
  mkdir -p "$HOME/loader"
  cp -rf "/boot/initrd.img" "$HOME/loader/initrd.img"
  cp -rf "/boot/vmlinuz" "$HOME/loader/vmlinuz"
  [[ -f "/boot/initrd.img" ]] && rm -rf "/boot/initrd.img"
  [[ -f "/boot/vmlinuz" ]] && rm -rf "/boot/vmlinuz"
  echo && ls -AR1 "$HOME/loader"
fi
```

脚本结束。

——


这个脚本还有其它很多需要增强的地方，比如dd progress显示。还比如改成了生成双分区。保留packerpe到第一分区。整盘其实可以preseed/partman成二个分区，一个分区200M放生成的packerpe自身（未来，还可以在这个packerpe集成dbcolinux或集成linuxbpot,coreboot，clover等），

还比如，扩展镜像到整个磁盘空间，实际上windows通过ntfs fuse是实现了的。对于linux磁盘，要处理镜像分区表适配到目标整个磁盘。这不光适用于dd方案。也适用于将磁盘打包成dd镜像（脚本中改变pipecmdstr，镜像扩容缩容）。然后nc 监听，供其它机器下载其镜像或直接dd。这是这个脚本的反向功能。

还比如，利用webfsd的cgi改dd地址。也可用grub的auto url变换dd地址。Ddurl参数保留到下一次。且读取位置不再在preseed中。

但是我不想做了，以后继续吧。这些天做windows server 2019 core的dd镜像也较累了，用了跟做osx云镜像差不多的方法：在linuxkvm中套虚拟机。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106822324/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





