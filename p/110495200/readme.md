利用增强tinycorelinux remaster tool打造你的硬盘镜像及一种让tinycorelinux变成Debian install体的设想
=====

__本文关键字:增强tinycorelinux remaster tool，tinycorelinux 开机加载module,x509: certificate signed by unknown authority__

在前面很多云主机装机相关的文章中，我们都讲到debian的netinstall实现云主机装机，它并不利用pxe这种cs结构和另外的装机服务器之类的东西，而是debian固有装机方式中的一种，即简单利用软件包仓库和chroot机制在线操作硬盘provision出一个ramos pe化os的原理，---- 这在《一个fully retryable的rootbuild packer脚本,从0打造matecloudos》和《把DI当online packer用:利用installnet制作一个云装机packerpe》都讲过。那么它在其它linux dists上有实现吗？

这种替代类似方案之一就是tinycorelinux，它追求小跟di一样，而且它本身就是一个ramos，（tinycorelinux内存os是什么意思呢？其实整个tc也可以通过把initrd.gz cpio -idmv < 到硬盘中运行。但是默认情况下，如果不提供tce=sda1之类的bootcode 及after bootinto system then tce-setup重配置，那么它的包是下载到/tmp这个内存fs和/挂载点的。如果指定硬盘上的tce目录加载，除了一些极端情况，如docker工作在ramfs会现privot_root异常，它实际上跟普通硬盘linux无异，这就是tc的stateless和cloud mount属性），而且tinycorelinux也有在线仓库，它的remaster tool和bootlocal.sh也可以成为简单的preseed和provision机制，可以简单地模拟出debian的netinstall环境。

ezremaster工作在gui下，通过读取仓库下的info.lst.gz形成软件列表（find 11.x/x86_64/tcz/*.tcz -print | sed -e 's/11.x\/x86_64\/tcz\///' > 11.x/x86_64/tcz/info.lst
gzip -cf 11.x/x86_64/tcz/info.lst > 11.x/x86_64/tcz/info.lst.gz），但是也可以让它工作在命令行下，提取其中的/usr/local/bin/remaster.sh，然后手动装好依赖tce-load -iw mkisofs-tools advcomp，调用remaster+ezremaster.cfg(这个必须为绝对路径)+命令进行工作，工作原理见脚本本身，过程是给一个模板iso，然后blalalala....最后生成一个新iso。

！！重点来了：我们想通过定制/增强tc的ezremaster tool来讲解tc的增强能力。比如我们想使之生成硬盘镜像呢（以后考虑进一步弄成pebuilder.sh之类的东西，呆会你就会看到，整remaster.sh的逻辑跟pebuilder.sh共享同样的原理和流程,这种可组装能力有点让tc等同metalinux的意思了）？该如何进行。不废话，（我们使用前面《一种设想：利用tinycorelinux+chrome模拟chromeos并集成vscodeonline》和《一种混合包管理和容器管理方案，及在tinycorelinux上安装containerd和openfaas》的实践案例，顺便在这里增强一下上述二文），我们使用的测试环境是上述二文中经过了打包openssh.tcz的corepure64-11.1-remasterd.iso，用它开的一台虚拟机，因为要手动安装ezremaster tool的依赖，所以还要：tce-load -iw mkisofs-tools advcomp parted grub2-multi util-linux ca-certificates（其实其它linux发行也可以运行remaster.sh，只要它们有cpio, tar, gzip, advdef, and mkisofs. Advdef is used to re-compress the image with a slightly better
implementation, producing a smaller image that is faster to boot,后四个tcz是新增的，供生成硬盘镜像用，没ca-certificates会出现ctr/faas-cli拉镜像时x509: certificate signed by unknown authority）

不废话了，直接提供脚本：

1,ezremaster.cfg文件
-----

一般用CorePure64-11.1.iso作模板，corepure64中的vmlinuz64，并没有一些像它的http://mirrors.163.com/tinycorelinux/11.x/x86_64/release/src/kernel/config-5.4.3-tinycore64一样，corepure64/lib/modules中为追求小也删了大量modules，我们用集成origmod.tcz的方法代替前面仅集成graphics modules.tcz等部分模块的方法以后，好多驱动认到了，鼠标跟指针分离的现象也貌似解决了？。但还是遗留了一些问题，比如virtio_blk实际上没有做成builtin，在云主机上工作时，需要寻求另外编译vmlinuz或寻求启动时加载modules的相关方案。

其它注意点：1，如果在gui中，loglevel=3和cde（如果你在cfg中写过了extract outside intro）是默认的，不需再写，2，因为要集成的tce比较多，最终的iso比较大，进程耗时，所以在/mnt/sda1/ezremaster(sudo chown tc:staff)上进行。3，faasd.tcz是使用了自己的镜像，（因为用od托管的tce镜像必须要用到wget ssl，因此临时先把/opt/tcemirror切换到mirrors.163.com/tinycorelinux下载openssl-1.1.1，然后切换到od主镜像），4，勾选产生的那个copy2fs.flg目前还考虑不到有什么用。,tc有多个藏tcz的地方，如tce,cde,initrd的集成包，且可以同时起作用，那么它是如何处有多个tcz目录的依赖项和防止冲突的？因为在tce=sda1且root=/dev/sda1硬盘模式下，/usrlocal/tce.installed会导致混乱.

其实我是把cfg文件当字串内置到remaster.sh中供grep处理的（本来是grep "^cd_location = " $input文件 | awk '{print $3}'），如下：

```
input="cd_location = /mnt/sda1/CorePure64-11.1.iso
temp_dir = /mnt/sda1/ezremaster/
cc = home=sda1 opt=sda1 tce=sda1 restore=sda1
app_outside_initrd_onboot = chromium-browser.tcz
app_outside_initrd_onboot = iptables.tcz
app_outside_initrd_onboot = faasd.tcz
app_outside_initrd_onboot = nginx.tcz
app_extract_initrd = openssh.tcz
app_extract_initrd = original-modules-5.4.3-tinycore64.tcz
app_extract_initrd = Xorg-7.7.tcz
extract_tcz_script = ignore
copy2fs.flg"

if [ ! -n $input ]; then
	echo "input is empty"
	exit 2
fi

......

cd_location=`echo "$input" | grep "^cd_location = " | awk '{print $3}'`
temp_dir=`echo "$input" | grep "^temp_dir = " | awk '{print $3}'`
```

脚本中其它的cat $input,都要改一下为上述形式，参数判断逻辑也改一下，使得一个参数也能通过。

2，改造脚本，使之生成硬盘镜像
-----

把下面这段加在package(){...}后，这部分主要的逻辑是硬盘镜像的挂载初始化与析构。

注意到，remaster.sh rebuild function本身是可以覆盖执行的，extract目录中的initrd在经过第一次iso打包后就已经变化，集成了app_extract_initrd指定的三个tczs，除非extract必变以后它的体积大小不会改变。（但保险起见，你依然可以选择在第一次脚本完成后打包那个/mnt/sda1/ezremaster为ezremaster-inital.gz备用），

我们加入了一些fixandmodinitrd，因此，我们必须保证extract目录也是依然可以覆盖使用的。因此我们把这个函数变成了fixandmodinitrdforonly，用了一个 if [ ! -f $temp_dir/extractalreadyprocessed ]; then判断，这种手法在《一个fully retryable的rootbuild packer脚本,从0打造matecloudos》系列中很常见。

 （前面tce-load中这个util-linux主要是因为busybox中的那个losetup太旧，没法用）

```
export BUILD_PACKAGE_MNT_PT=/tmp/buildpackage

fixandmodinitrdforonlyonce() {

    if [ ! -f $temp_dir/extractalreadyprocessed ]; then

        sudo cp -f /etc/passwd etc/passwd
        sudo cp -f /etc/shadow etc/shadow

        sudo cp -f /usr/local/etc/ssh/sshd_config.orig usr/local/etc/ssh/sshd_config
        sudo cp -f -R /usr/local/etc/ssl usr/local/etc/
        sudo cp -f -R /usr/local/etc/ssh usr/local/etc/
        sudo mkdir -p var/lib/sshd/

        sudo cp -av /usr/local/etc/ssl/ etc/ssl

        sudo sh -c "echo Xorg > etc/sysconfig/Xserver"
        sudo sh -c "echo /usr/local/bin/chromium-browser --start-maximized http://127.0.0.1 > etc/skel/.X.d/chromium-browser"
        sudo chmod +x etc/skel/.X.d/chromium-browser

        sudo sh -c "echo /usr/local/etc/init.d/openssh start >> opt/bootlocal.sh"
	sudo sh -c "echo http://d.shalol.com/mirrors/tinycorelinux/ >> opt/tcemirror"

	sudo sh -c "find . | cpio -o -H newc | gzip -2 > $temp_dir/image/boot/corepure64.gz" || exit 22
	sudo advdef -z4 $temp_dir/image/boot/corepure64.gz || exit 23
        
        sudo touch $temp_dir/extractalreadyprocessed

    else
        echo fixandmodinitrdforonlyonce is already processed!
    fi
}

packagehd() {
    dev_buildpackage=$(mount | grep "$BUILD_PACKAGE_MNT_PT" | awk '{print $1}')
    if [ -z "$dev_buildpackage" ];then
        echo "- Creating new dev_buildpackage disk"
        sudo rm -rf $temp_dir/buildpackage.raw
        sudo dd if=/dev/zero of=$temp_dir/buildpackage.raw bs=1024 count=20971520
        dev_buildpackage=`sudo /usr/local/sbin/losetup -fP --show $temp_dir/buildpackage.raw | awk '{print $1}'`
        echo

        [ -n "$dev_buildpackage" ] && { 
                sudo parted -s "$dev_buildpackage" mktable msdos
                sudo parted -s "$dev_buildpackage" mkpart primary ext3 2048s 100%
                sudo parted -s "$dev_buildpackage" set 1 boot on
                sudo mkfs.ext3 "$dev_buildpackage"p1 
        }

        [ ! -d "$BUILD_PACKAGE_MNT_PT" ] && sudo mkdir "$BUILD_PACKAGE_MNT_PT"
        sudo mount "$dev_buildpackage"p1 "$BUILD_PACKAGE_MNT_PT"

        sudo grub-install --boot-directory="$BUILD_PACKAGE_MNT_PT"/boot "$dev_buildpackage"
        sudo grub-mkconfig -o "$BUILD_PACKAGE_MNT_PT"/boot/grub/grub.cfg
        sudo sh -c "echo set timeout=3 >> $BUILD_PACKAGE_MNT_PT/boot/grub/grub.cfg"
        sudo sh -c "echo menuentry \"this is a raw hd\" { >> $BUILD_PACKAGE_MNT_PT/boot/grub/grub.cfg"
        sudo sh -c "echo linux /boot/vmlinuz64 loglevel=3 tce=sda1 opt=sda1 home=sda1 restore=sda1 cde >> $BUILD_PACKAGE_MNT_PT/boot/grub/grub.cfg"
        sudo sh -c "echo initrd /boot/corepure64.gz >> $BUILD_PACKAGE_MNT_PT/boot/grub/grub.cfg"
        sudo sh -c "echo } >> $BUILD_PACKAGE_MNT_PT/boot/grub/grub.cfg"


	sudo depmod -a -b $temp_dir/extract `uname -r`
	sudo ldconfig -r $temp_dir/extract
	
	cd $temp_dir/extract
	umount $temp_dir/extract/proc >/dev/null 2>&1
        fixandmodinitrdforonlyonce


	cd $temp_dir/image
        sudo cp -av boot/ "$BUILD_PACKAGE_MNT_PT"/
        sudo cp -av cde/ "$BUILD_PACKAGE_MNT_PT"/tce


    fi
    # Automatically remove DISK on exit
    trap 'echo; echo "- Ejecting dev_buildpackage disk"; cd "$HOME"; sudo umount "$BUILD_PACKAGE_MNT_PT" && sudo /usr/local/sbin/losetup -d "$dev_buildpackage"' EXIT
}
```
最后，参数处加一条	packagehd) packagehd ;;作调用

测试，修改镜像为od主镜像，运行remaster.sh /mnt/sda1/ezremaster.cfg rebuild生成iso，然后remaster.sh /mnt/sda1/ezremaster.cfg packagehd，生成硬盘镜像，成功，脚本是自带tail -f ezremaster.log效果的，里面会出现一些wget 404，应该是一些无关紧要的info文件等的下载。无妨。


最后你可以你可以tar这个镜像用tce-load -iw python，sudo python -m SimpleHTTPServer 80下载这个镜像，默认0.0.0.0。如果发现卡住，下载不了ctrl c一下python进程。


-----

最后，这个脚本可以保存为remaster.sh，但是因为tc没有arm64，所以准备转debian，这样也可以与pebuilder.sh针对的debian netinstall接上，但是默认提供的最小的netinstall.iso也要几百M，网上看到一篇《基于 debootstrap 和 busybox 构建 mini ubuntu》https://www.cnblogs.com/fengyc/p/6114648.html，据称可以将debian或ubuntu精简到40-50m，或更小，它的思路主要是将启动脚本依赖的占用替换为简单bb和内核模块精简一下，所以以后有机会了尝试一下，还看到一个slax的live系统，我们要尝试用来代替tc的这个新系统一定要像slax和tc一样live加载扩展，而且要/ext模块目录和/os，用来保存数据的/data单独三个文件夹layout到/下。


