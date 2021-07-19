一键pebuilder，实现云主机在线装deepin20beta
=====

__本文关键字：deepin云主机版本，在onedrive里装机__

国产系统deepin现在已经很精美了，国内很多软硬厂商在慢慢联手uos，。手机os华为方面2020.9月份会发布第一个可用版鸿蒙，这也是一个"uniform os"，最近听说美控台积电又要给华为断供了，但其实技术和生态也是俗物，只要钱慢慢到位，时间都是第二位的，也就没有不可克服的问题，毕竟华为芯片都能自研了，这种从1到2的事情其实反而好办。

我们《minlearnprogrammingv2:选型与实践》作为《minlearnprogramming:理论与实践》的替代与延续，也力在打造一个统一os：mateos,matecloud os，最初定位于从tinycorelinux开始深度定制，最近决定转为基于deepin深度定制，这是一个集成云booter,面向云应用，由客服一体的netdisk backend apps组成的生态。---- 这种集成虚拟机和linux的云booter和kernel exec，真正让rootfs as os container content,使得app和os,subos一个性质都是rootfs。从最开始和源头就自带强烈的虚拟化特性，它类似qubeos,不过qubeos的用处在于利用上述设施制造安全系统，它把联网代码转置入非安全域所以用到vtd，据说还有集成quubeos的librem安全笔记本。而我们的目的在于容器和app container定义和建设。这样deepin os被转换成pure rootfs，放置在rootfs的位置。一个虚拟机上可以运行多份大小os。---- 况且为了这种os下的开发，我们还准备了可视开发和网盘内devops支持（这是后话）

好了不废话了。先在云主机上装上deepinos20beta，经实践，deepin20的界面（非特效）在gd5446下也非常流畅，它的界面效果也是四角圆白色风格，类web还有darkmode。所以放到云主机也是很和谐的效果。

首先我们要创建可用的镜像，还要增强一下《利用onedrive加packerpebuilder实现本地网络统一装机》以来的在od里装机的方案：

创建云主机镜像
-----

利用的是云主机上装黑果系列文章中制造镜像的方法(而非packer结合oss,cos)，在pd中装deepin，开kvm，再启动qemu制造镜像：

```
qemu-img create -f raw deepin20 20G

qemu-system-x86_64 -enable-kvm \ -machine pc-i440fx-2.8 \ -cpu Penryn,kvm=off,vendor=GenuineIntel \ -m 5120 \ -device cirrus-vga,bus=pci.0,addr=0x2 \ -usb -device usb-kbd -device usb-mouse \ -device ide-drive,bus=ide.0,drive=MacDVD \ -drive id=MacDVD,if=none,snapshot=on,media=cdrom,file=./20.iso \
 -device ide-drive,bus=ide.1,drive=MacDVD1 \ -drive id=MacDVD1,if=none,snapshot=on,media=cdrom,file=./3.iso \ -device virtio-blk-pci,bus=pci.0,addr=0x5,drive=MacHDD \ -drive id=MacHDD,if=none,cache=writeback,format=raw,file=./deepin20.raw \ -device virtio-net-pci,bus=pci.0,addr=0x3,mac='52:54:00:c9:18:27',netdev=MacNET \ -netdev bridge,id=MacNET,br=virbr0,"helper=/usr/lib/qemu/qemu-bridge-helper" \
 -boot order=dc,menu=on
```

手动全盘挂载到/不要自动全盘否则小于64g安装程序不让你过去。
以上脚本方案当然也适合windows，我用类似的方案制造了一个winsrvcore2019.gz,对于windows你还需要一个iso
virtio-win-0.1.171.iso，上面的3.iso就是。如果是for deepin可以去掉。当然windows需要定制，比如防止ping localhost出现ipv6，disablectlaltdel登录等等。

这样装好的镜像在还原到云主机上或在pd kvm/qemu中运行，都是可以正常联网的。

强化pebuilder.sh
-----

加了一个export tmpGENMIRRORBAK='0'全局变量，即-g开关，直接pebuilder.sh -dd '你的镜像文件地址'，使用-g开关可以生成本地镜像仓库以供上传到你自己的onedrive用

加入了当Downloading basic kernel and rootfs files时：

```
if [[ "$tmpGENMIRRORBAK" == '1' ]]; then
  bakdir='/tmp/boot/var/log/debian/dists/jessie/main/installer-amd64/current/images'
  mkdir -p "$bakdir/netboot/debian-installer/amd64"
  cp "/boot/initrd.img" "${bakdir}/netboot/debian-installer/amd64/initrd.gz"
  cp "/boot/vmlinuz" "${bakdir}/netboot/debian-installer/amd64/linux"
  wget --no-check-certificate -qO $bakdir/udeb.list $MIRROR/dists/jessie/main/installer-amd64/current/images/udeb.list
fi
```

接下来的downloading full udeb pkg当然也用这个开关控制。

新的自动化prepare ddessentials：

```
UNZIP=''
IMGSIZE=''

function PrepareDDessentials(){

  if [[ -n "$tmpDDURL" ]]; then
    echo "$tmpDDURL" |grep -q '^http://\|^ftp://\|^https://';
    [[ $? -ne '0' ]] && echo 'No valid URL in the DD argument,Only support http://, ftp:// and https:// !' && exit 1;

    IMGHEADERCHECK="$(curl -Is "$tmpDDURL")";
    IMGTYPECHECK="$(echo "$IMGHEADERCHECK"|grep -E -o '200|302'|head -n 1)" || IMGTYPECHECK='0';

    #directurl style,no addon headcheckpass,1 addon typecheckpass to the final
    [[ "$IMGTYPECHECK" == '200' ]] && \
    IMGTYPECHECKPASS_DRT="$(echo "$IMGHEADERCHECK"|grep -E -o 'raw|qcow2|gzip|x-gzip'|head -n 1)" && {
      # IMGSIZE
      [[ "$IMGTYPECHECKPASS_DRT" == 'raw' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
      [[ "$IMGTYPECHECKPASS_DRT" == 'qcow2' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
      [[ "$IMGTYPECHECKPASS_DRT" == 'gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
      [[ "$IMGTYPECHECKPASS_DRT" == 'x-gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
      [[ "$IMGTYPECHECKPASS_DRT" == 'gunzip' ]] && UNZIP='2' && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
    }

    # refurl style,(added 1 more imgheadcheck and 1 more imgtypecheck pass in the middle)
    [[ "$IMGTYPECHECK" == '302' ]] && \
    IMGHEADERCHECKPASS2="$(echo "$IMGHEADERCHECK"|grep 'Location: http'|sed 's/Location: //g')" && IMGHEADERCHECKPASS2=${IMGHEADERCHECKPASS2%$'\r'} && IMGHEADERCHECKPASS2="$(curl -Is "$IMGHEADERCHECKPASS2")" && \
    IMGTYPECHECKPASS2="$(echo "$IMGHEADERCHECKPASS2"|grep -E -o '200|302'|head -n 1)" && {

      #sharepoint style,1 addon typecheck pass to the final
      [[ "$IMGTYPECHECKPASS2" == '200' ]] && \
      IMGTYPECHECKPASS_SPT="$(echo "$IMGHEADERCHECKPASS2"|grep -E -o 'raw|qcow2|gzip|x-gzip'|head -n 1)" && {
        # IMGSIZE
        [[ "$IMGTYPECHECKPASS_SPT" == 'raw' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS_SPT" == 'qcow2' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS_SPT" == 'gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS_SPT" == 'x-gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS_SPT" == 'gunzip' ]] && UNZIP='2' && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
      }

      # office365 style,1 addon headercheck and 1 addon typecheck pass to the final
      [[ "$IMGTYPECHECKPASS2" == '302' ]] && \
      IMGHEADERCHECKPASS3="$(echo "$IMGHEADERCHECK"|grep 'Location: http'|sed 's/Location: //g')" && IMGHEADERCHECKPASS3=${IMGHEADERCHECKPASS3%$'\r'} && IMGHEADERCHECKPASS3="$(curl -Is "$IMGHEADERCHECKPASS3")" && \
      IMGHEADERCHECKPASS4="$(echo "$IMGHEADERCHECK3"|grep 'content-location: http'|sed 's/content-location: //g')" && IMGHEADERCHECKPASS4=${IMGHEADERCHECKPASS4%$'\r'} && IMGHEADERCHECKPASS4="$(curl -Is "$IMGHEADERCHECKPASS4")" && \
      IMGTYPECHECKPASS_OFE="$(echo "$IMGHEADERCHECKPASS4"|grep -E -o 'raw|qcow2|gzip|x-gzip|gunzip'|head -n 1)" && {
        # IMGSIZE
        [[ "$IMGTYPECHECKPASS_OFE" == 'raw' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS_OFE" == 'qcow2' ]] && UNZIP='0' && sleep 3s && echo -en "[\033[32m raw \033[0m]";
        [[ "$IMGTYPECHECKPASS_OFE" == 'gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS_OFE" == 'x-gzip' ]] && UNZIP='1' && sleep 3s && echo -en "[\033[32m x-gzip \033[0m]";
        [[ "$IMGTYPECHECKPASS_OFE" == 'gunzip' ]] && UNZIP='2' && sleep 3s && echo -en "[\033[32m gunzip \033[0m]";
      }

    }


    [[ "$UNZIP" == '' ]] && echo 'didnt got a unzip mode, you may input a incorrect url,or the bad network traffic caused it,exit ... !' && exit 1;
    #[[ "$IMGSIZE" -le '10' ]] && echo 'img too small,is there sth wrong? exit ... !' && exit 1;
    [[ "$IMGTYPECHECK" == '0' ]] && echo 'not a raw,tar,gunzip or 301/302 ref file, exit ... !' && exit 1;

  else
    echo 'Please input vaild image URL! ';
    exit 1;
  fi

}
echo -en '\n\nprepare DDessentials ......';
PrepareDDessentials;
```

加入了定制grub，以提供给用户手动执行pebuilder的机会：

```
sed -i 's/timeout_style=hidden/timeout_style=menu/g' $GRUBDIR/$GRUBFILE;
sed -i 's/timeout=[0-9]*/timeout=30/g' $GRUBDIR/$GRUBFILE;
```

重要的prepare others->the preseed部分：

```
[[ "$UNZIP" == '0' ]] && PIPECMDSTR='wget -qO- '$tmpDDURL' |dd of=$(list-devices disk |head -n1)';
[[ "$UNZIP" == '1' ]] && PIPECMDSTR='wget -qO- '$tmpDDURL' |tar zOx |dd of=$(list-devices disk |head -n1)';
[[ "$UNZIP" == '2' ]] && PIPECMDSTR='wget -qO- '$tmpDDURL' |gzip -dc |dd of=$(list-devices disk |head -n1)';
```

和packaging部分：

```
  echo -e "\n\033[36m# Packaging \033[0m\n"

  sleep 2s &&   echo -en "make a safe wget wrapper to inc --no-check-certificate\n"

  rm -rf /tmp/boot/usr/bin/wget
cat >/tmp/boot/usr/bin/wget<<EOF
#!/bin/sh
rdlkf() { [ -L "\$1" ] && (local lk="\$(readlink "\$1")"; local d="\$(dirname "$1")"; cd "\$d"; local l="\$(rdlkf "\$lk")"; ([[ "\$l" = /* ]] && echo "\$l" || echo "\$d/\$l")) || echo "\$1"; }
DIR="\$(dirname "\$(rdlkf "\$0")")"
exec /usr/bin/env wget2 --no-check-certificate "\$@"
EOF
  chmod +x /tmp/boot/usr/bin/wget

  sleep 2s && echo -en "packaging\n"
  find . | cpio -H newc --create --quiet | gzip -9 > /boot/initrd.img;

  [[ "$tmpGENMIRRORBAK" == '1' ]] &&  echo -en "packaging finished,and all done! auto reboot after 9999s...(if needed, you can press ctrl c to interrupt to bak the repodir under tmp/boot/, then manually reboot to continue)" && sleep 9999s

  rm -rf /tmp/boot;
```
还有一些小的修补。

------

给pe配不死booter的工作还在设想和进行中，可能开始会提供一个较简易的方案，而非与制造开头所提到的云booter一起完成。如果能，当然最好


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107448089/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



