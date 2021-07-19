一种混合包管理和容器管理方案，及在tinycorelinux上安装containerd和openfaas
=====

__本文关键字：在tinycorelinux上装docker，virtual appliance vs virtual appstack，no cgroup mount found in mountinfo: unknown，jailing process inside rootfs caused: pivot_root invalid argument: unknown__

在《利用openfaas faasd在你的云主机上部署function serverless面板》 和《panel.sh：一个nginx+docker的云函和在线IDE面板,发明你自己的paas(1),(2)》文中，我们谈到openfaas是一种基于containerd(docker官方经过对旧docker进行模块化重构后得到的二进制级的可复用结构，作为容器的运行时实现存在，containerd+runc+cni是默认的代替原docker的选型，https://github.com/containerd/containerd/releases也是把cni,containerd,runc这样打包一起的)的云函数面板，《tinycorelinux上编译安装lxc,lxd》《tinycorelinux上装ovz》这些文章也都是在linux上容器选型的例子。

> 容器化为什么这么重要，因为容器是现在最流行的原生virtual cloud appliance（cloud appliance化是app部署级别的融合，代表着“为云APP造一种包结构”，k8s这些被称为云原生所以你可以将其简单理解为云原生软件包，cloud appliance要与app开发用的内部cloud appstack融合化区别开）代表，作为统一部署方案，它主要关注解决集群和云上那些“软件发行资源配额的隔离”问题，软件的资源配额从来都是一个复杂的关联问题，可巧这些问题在本地和原生的包管理软件中（包主要关注解决依赖问题）也存在，linux上提供了统一的整套内核级支持方案（比如liblxc,libcgroups，基于它们+brtfs可以完全用shell发明一套简单的docker运行时，当然这跟我们需要的，最终完备的容器和容器管理系统containerd,openfaas是没法比的），另一方面，“资源隔离”稍微更进一步，就很容易与“软件怎么样启动”这些问题相关联，变成systemd这类软件要解决的问题（systemd-nspawn可以创建最轻量级的容器），进而变成k8s这类软件要解决的问题。所以，这三大问题融合和关联发生在方方面面，，很容易成为某种“混合包管理和容器管理”融合体系要解决的中心问题，进而需要这样一种软件，core os的https://github.com/rkt/rkt就是这一类软件的代表并为此构建出一个基于容器作为包管理的OS（虽然2020年中期它们准备dreprecate了）。

这就是说，容器化的实现可以简单也可以复杂，不同OS也有集成不同复杂容器管理的方案，除了core os的rkt这种，tc的tce pkg本身就是一种沙盒环境，不过它与上述提到的lxc,ovz,containerd这些真正意义的容器化没有关系。前述与openfaas相关的这些文章都是在流行的linux发行版中实践装openfaas的例子，接下来的本文将介绍尝试在tinycorelinux11中装真正的容器，即containerd和openfaas的安装实践过程。

> 这里的主要问题是，tc本身是一种raw linux发行版，追求小和简单，一般地，跟alpine一样tc往往作为容器guest os如boot2docker，鲜少在tc上装containerd作为容器服务器环境，因此，在tc中装容器可能会因为没有现成参考方案而显得繁琐。比如，tc也没有使用systemd这类复杂的init启动管理系统而是简单的sysv init（虽然systemd提出了一个巨大的init pid 1，但是它只关注“启动”，这点上，它还是符合kiss的）。一般linux发行都是依赖systemd处理容器设置的一些至关重要的基础问题,比如稍后我们会谈到systemd自动管理cgroups，，而这些在tc中都没有,需要手动还不见得能解决，

不管了，下面开始尝试实践，我们的测试环境是tc11。


1，制造一个containerd.tcz和faasd.tcz
-----

准备一个集成了openssh,sudo passwd tc,做好了bootcode tce=sda1，echo过 /opt/tcemirror,/etc/passwd,/etc/shadow > /mnt/sda1/opt/.filetool.lst，tar 进restore=sda1 mydata.gz的基本ezremasterd tc11 iso，即《一个fully retryable的rootbuild packer脚本,从0打造matecloudos(2)》中第一小节那样的iso。我们开启二个引导了这个iso的虚拟机,都用parted格好sda1，这二虚拟机准备一个生成containerd.tcz和faasd.tcz用（生成后在其目录下python -m SimpleHTTPServer 80供以后下载，事先ifconfig看好ip），另一个测试生成的tcz(/opt/tcemirror设成第一台地址+11.x/x86_64/tcz正确的结构)，在第一台虚拟机的/mnt/sda1根目录中，安排这些文件：

```
准备文件夹结构和binaries:
docker采用前述文章的版本组合而成（做二文件夹一个containerd-root，其中cni放在/opt/cin/bin,runc放在/usr/local/sbin,containerd放在/usr/local/bin，一个faasd-root，其中放/usr/local/bin/faasd,faas-cli，最后，前文提到的几个offline docker image也集进来放在faasd-root/tmp/*.tar）。这些exe设好chmod +x，由于这些exe都是go的，都是静态链接的，在tc11上可直接运行(lib64一定要ln -s 一下到lib，否则ctr不能起作用)。

准备几个配置文件：
一些containerd和faasd启动时的动态文件，不用创建。否则会导致只读文件系统无法写入错误
/etc/cni/net.d/10-openfaas.conflist
/var/lib/faasd/secrets/*
/var/lib/faasd/resolv.conf
/var/lib/faasd-provider/resolv.conf
这些文件必须要
/var/lib/faasd/docker-compose.yaml
/var/lib/faasd/prometheus.yml

准备overlay module:
在第一台中tce-load -iw bc compiletc perl5重新编译kernel，在config中把config_overy_fs打开为m，得到overlay module,因为它在tc11中被关闭了。下载tc11中所需的kernel编译文件http://mirrors.163.com/tinycorelinux/11.x/x86_64/release/src/kernel/到/mnt/sda1
解压并cp config-5.4.3-tinycore64 linux-5.4.3/.config
sudo make oldconfig，提示几个交互项直接回车
sudo make install
sudo make modules_install
把得到的overlay module file(在/lib/modules/fs中)放到准备打包的containerd.tcz文件夹结构中。

准备2个服务文件并分别chmod +x：
containerd-root/usr/local/init.d/start-containerd:
/sbin/modprobe overlay
/usr/local/bin/containerd
containerd-root/usr/local/init.d/start-faasd:
for i in 1 2 3; do [[ ! -z "$(ctr image list|grep basic-auth-plugin)" ]] && break;ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/basic-auth-plugin-0.18.18.tar;echo "checking basic-auth ($i),if failed at 3,it may require a reboot"; sleep 3;done         
for i in 1 2 3; do [[ ! -z "$(ctr image list|grep nats)" ]] && break;ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/nats-streaming-0.11.2.tar;echo "checking nats ($i),if failed at 3,it may require a reboot"; sleep 3;done                                
for i in 1 2 3; do [[ ! -z "$(ctr image list|grep prometheus)" ]] && break;ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/prometheus-v2.14.0.tar;echo "checking prometheus ($i),if failed at 3,it may require a reboot"; sleep 3;done                       
for i in 1 2 3; do [[ ! -z "$(ctr image list|grep gateway)" ]] && break;ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/gateway-0.18.18.tar;echo "checking gateway ($i),failed at 3,it may require a reboot"; sleep 3;done                                   
for i in 1 2 3; do [[ ! -z "$(ctr image list|grep queue-worker)" ]] && break;ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/queue-worker-0.11.2.tar;echo "checking queueworker ($i),if failed at 3,it may require a reboot"; sleep 3;done
cd /var/lib/faasd
/usr/local/bin/faasd provider
/usr/local/bin/faasd up

准备mkall.sh放到/mnt/sda1根下chmod +x起来：
mkall.sh的内容(注意到统一作为tc:staff存放到tcz中):
rm -rf containerd.tcz containerd.tcz.md5.txt faasd.tcz faasd.tcz.md5.txt faasd.tcz
mksquashfs containerd-root containerd.tcz -noappend -no-fragments -force-uid tc
md5sum containerd.tcz > containerd.tcz.md5.txt
mksquashfs faasd-root faasd.tcz -noappend -no-fragments -force-uid tc -force-gid staff
md5sum faasd.tcz > faasd.tcz.md5.txt
echo containerd.tcz > faasd.tcz.dep
```

sudo ./mkall.sh打包好的tcz各80多m，准备好后我们就可以在第二台测试了，tce-load -iw faasd后经过测试不满意可删除/mnt/sda1/tce/optional下的tcz重启重来，我们需要不断mkall并测试生成的二个tcz。

2，测试
-----

第一次测试启动start-containerd,start-faasd，出现:no cgroup mount found in mountinfo: unknown这就是上面谈到tc11不具有自动处理cgroups的逻辑。而containerd依赖它们。tc11中的kernel config中提供了linux对容器的内核支持基础，只是没有更进一步。

> CGroup 提供了一个 CGroup 虚拟文件系统，作为进行分组管理和各子系统设置的用户接口。要使用 CGroup，必须挂载 CGroup 文件系统。这时通过挂载选项指定使用哪个子系统。
> 需要注意的是，在使用 systemd 系统的操作系统中，/sys/fs/cgroup 目录都是由 systemd 在系统启动的过程中挂载的，并且挂载为只读的类型。换句话说，系统是不建议我们在 /sys/fs/cgroup 目录下创建新的目录并挂载其它子系统的。这一点与之前的操作系统不太一样。

针对于此，很幸运我们找到了https://gitee.com/binave/tiny4containerd/blob/master/src/rootfs/usr/local/etc/init.d/cgroupfs.sh,它使用old docker，并且基于https://github.com/tianon/cgroupfs-mount/（这个工程https://gitee.com/binave/tiny4containerd/src/rootfs/usr/local/这里的lvm动态扩展分区脚本和docker服务，cert处理等函数也不错，可为未来所用），里面有几句。

```
mount -t tmpfs -o uid=0,gid=0,mode=0755 cgroup /sys/fs/cgroup

如果你没有上面这句mount 接下来会mkdir: can't create directory 'cpu': No such file or directory，因为/sys/fs/cgroup只是内核给的fake fs

cd /sys/fs/cgroup;

# get/mount list of enabled cgroup controllers
for sys in $(awk '!/^#/ { if ($4 == 1) print $1 }' /proc/cgroups); do
    mkdir -p $sys
    if ! _mountpoint -q $sys; then
        if ! mount -n -t cgroup -o $sys cgroup $sys; then
            rmdir $sys || true
        fi
    fi
done
```
我们把cgroupfs放跟containerd-root/usr/local/etc/init.d/containerd并排，在containerd脚本中启动containerd前加入/usr/local/etc/init.d/cgroupfs.sh mount这句。打包再测试：出现jailing process inside rootfs caused: pivot_root invalid argument: unknown(我也一直没有测试https://gitee.com/binave/tiny4containerd/中的docker会不会出现这错误，不过听说有遇到了https://forums.docker.com/t/tinycore-8-0-x86-pivot-root-invalid-argument/32633),

> In a system running entirely in memory, after an upgrade from 17.09.1-ce to 17.12.0-ce, docker stopped creating containers, failing with message like docker: Error response from daemon: OCI runtime create failed: container_linux.go:296: starting container process caused "process_linux.go:398: container init caused \"rootfs_linux.go:107: jailing process inside rootfs caused \\\"pivot_root invalid argument\\\"\"": unknown..

查网上说要使用DOCKER_RAMFS=true环变，我试了没用。

在其它非tc上相似的容器产品也有人遇到了：https://engineeringjobs4u.co.uk/how-we-use-hashicorp-nomad，针对于此它们做了一个内核补丁：https://lore.kernel.org/linux-fsdevel/20200305193511.28621-1-ignat@cloudflare.com/

> The main need for this is to support container runtimes on stateless Linux system (pivot_root system call from initramfs). Normally, the task of initramfs is to mount and switch to a "real" root filesystem. However, on stateless systems (booting over the network) it is just convenient to have your "real" filesystem as initramfs from the start.

对linux543/fs/namespace.c进行手动patch,mnt_init()定义前增加，和中间增加新加的代码。但是没用。因此按《利用hashicorp packer把dbcolinux导出为虚拟机和docker格式(3)》的方法转为传统硬盘安装方式。问题解决。

然后又出现了：
Error creating CNI for basic-auth-plugin: Failed to setup network for task "basic-auth-plugin-1210": failed to create bridge "openfaas0": could not add "openfaas0": operation not supported: failed to create bridge "openfaas0": could not add "openfaas0": operation not supported

这个问题其实在意料之中，因为从前面文章的经验来看，我们一直对containerd中的那个cni必须要起作用留了个心，可是faasd up产生了10openfaas.conflist后，我一直尝试ifconfig，都没看到第三个网卡。

网上有人提示说是CONFIG_BRIDGE_VLAN_FILTERING，看tc11的kernel，config_bridge被作为模块了，它的file应该是bridge.ko之类。但modprobe bridge没用，tce-load -iw original-modules-5.4.3-tinycore64，这下成功了。（安装了这个包之后，控制台显示好多设备都认到了）

再测试：
Error: Failed to setup network for task "basic-auth-plugin-3894": failed to locate iptables: exec: "iptables": executable file not found in $PATH: failed to locate iptables: exec: "iptables": executable file not found in $PATH，需要tce-load -iw iptables，

至此，faad up启动成功。containerd控制台显示warning,memory cgroup not supported,应该是kernel config，又没设好。

我们接下来要做的，就是把usr/local/etc/init.d下的几个服务文件做完善点。参考https://github.com/MSumulong/vmware-tools-on-tiny-core-linux/blob/tiny-core-6.3/additional-files/etc/init.d/open-vm-tools和https://gitee.com/binave/tiny4containerd/blob/master/src/rootfs/usr/local/etc/init.d/





