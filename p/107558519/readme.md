Dsm as deepin mate(2)：在阿里云上真正实现单盘安装运行skynas
=====

__本文关键字：单盘群晖，本验证码版黑群__

现在是2019新冠疫情时期，记得关于sars的描述的一句话吗，这种病在早期人类历史上，可以杀死一个大洲的人都不止，所以，对这种病有解的医生，等同于神。

在《使用群晖作mineportalbox》文3和《利用整块化自启镜像实现黑群在单盘位实机与云主机上的安装启动》，及《dsm as deepin mate》文1中，我们着重提到使skynas在云主机上单机安装使用的方法，这些文章中，研究点经历了从“黑群webassit，从0开始安装的方法”到“从镜像开始，能启动就行”的路径变化，但这二者我们都没有完全完成，达成最终目的。下面继续这个课题，实现最终在阿里云上单盘安装运行skynas。


制造并导出镜像的方法
-----

我们重新开始，首先去阿里云开一台系统盘20G，数据盘20G，镜像为skynas615-15254的按量机器，等进去了（因为我们不尝试导出纯净可安装镜像，所以这里不必跟以前一样等一启动就急停，做快照…，或者使用阿里云最新的卸载系统盘功能,恢复系统盘初始状态不重启…，其实这个时候导出的镜像也没用，会失效，稍后会讲到），新建一个admin，密码自设，这个admin会是管理员。我们发现应用只有系统区的二个,filestation和universal search，没有关系，打开控制面板，外部访问中的ssh，我们可以ssh admin@yourip从命令行用admin登录，然后sudo synouser --setpw root xxx，然后exit或su root切换到root。

其它的应用去http://synology.cn/zh-cn/support/download/VirtualDSM#packages复制其spk地址。通过命令行cd到/tmp，wget这个spk，通过synopkg install xxx.spk来安装，这种方式下你需要自己探寻app的依赖关系。启动失败一般是依赖没安好。

但是我们这里并不急着安装APP，因为我们优先考虑尽早把系统设置保存下来，这些将来用于作为模板初始化新volume， ——— 系统启动，安装APP会在数据盘volume1中产生文件，分析产生的文件有volume1/@appstore,volume1/@databases,/tmp，volume1/@tmp，这些文件与APP与系统本身都绑定，如果不打包打走，等volume1被释放，这些APP或系统本身，在新volume上启动，运行都会产生设定丢失或权限隐患等问题。比如tmp中包含有系统关于volume1的空间设定（这个设定会让没有任何volume的系统在启动时误认为volume1损毁，但这正是我们需要的,在以前的文章中，我们那个tmpdata,tmproot只是安装/升级时解包的逻辑。并不是进入系统后判断用作数据盘的逻辑。），相应的，@appstore应该就是app的数据。—— 为了不使这个模板文件过大，也是为了不带入因集成APP产生的不确定因素，我们不必事先在这个20G的数据盘volume上装太多APP，后面在增强部分会讲到APP。

>> 我们打包这些初始设置文件并替换volume所在盘：
>> /volume1/@database/pgsql(它实际上/var/services/pgsql,app和系统都会使用它)
>> /volume1/@tmp（它实际上是/var/services/tmp,app和系统都会使用它）
>> /tmp（这个目录极为重要，系统使用）
>> cd到/，tar cvpzf template.gz tmp volume1/@tmp volume1/@databases，带原用户权限打包，这样这个模板文件就有了。

事实上，在这里我们就可以把镜像导出来了。因为此时群晖是视数据盘为volume1的，我们盘和模板都有了，所以将在下一次重启时永久卸载这个volume1，让它用上新volume就行了。

来测试一下：1，制造新volume。因为群晖中有parted，比起fdisk它是即时生效的，所以sudo parted /dev/sda mkpart primary 4 100%，然后格式化sudo mkfs.btrfs /dev/sda4，完成。2，sudo reboot 重启，在阿里云ECS的存储管理中，卸载volume1，重启进来，登录web端。空间管理器提示没有任何存储，这是意料之中的，那么没有什么也没有提示空间损毁呢，这是因为tmp是每次启动都会清空的，保存在其中的空间信息会丢掉。3，我们把template.gz恢复到tmp，web端存储管理器立刻提示空间损毁，这是与原来数据盘volume1的挂钩的tmp/space数据，这个时候我们只需要把/dev/sda4挂载到/volume1就可以了。我们在命令行中尝试root登录，sudo mount /dev/sda4 /volume1挂载，web端已显示存储空间可用，这里其实格式sda4为Ext4也可以直接挂载。

然后你就可以导出这个自动镜像为别的机器所用。导出时可以在线nc导出(需配合virtio tinycorelinux，因为群晖没有nc也不能在其中直接nc):1，dd if=/dev/vda | gzip -c | nc -v -l -p 5000 , 恢复处用nc -v xxx.xxx.xxx.xxx(这里最好使用阿里云内网地址) 5000 | gzip -dc | dd of=/dev/vda，也可以2:保存为一个本地gz的方式然后使用ossutil+oss内部endpoint地址导出或者scp xx.gz root@xxx.com:/var/www上传到一个远程空间，oss最快。

恢复到不同的机器时，需要手动产生sda4及重复上述测试过程（使用非custom linux镜像如果是custom linux，需要配合virtio tinycorelinux修改/boot/aliyun_image/os.conf），1，wget -qO- http://xxx/vdaimg.gz（配合oss内网endpoint和virtio tinycorelinux） | dd of=/dev/vda 或者2用InstallNET.sh -dd，重新测试前，记得把网络改了，你需要改，etc/sysconfig/network，etc/sysconfig/network-scripts，/etc/resolv.conf，并增加一条gateway：route add default gw xxx.xxx.xxx.xxx。

增强：在镜像中直接封装sda4 as volume1，及提前封装app
-----

事实上，我们可以对镜像进行一些强化，因为上述过程在每一次换机或重启都要进行一次，为了追求制造一个自动的镜像。我们需要自动产生坏盘和自动挂载sda4，但又不能直接把这个sda4封装进去(因为这样在不同机器上不能自动扩展空间)，所以还需要一些内容

>> 自动产生损毁盘。自动产生并挂载sda4
>> 要echo 自动挂载规则到fstab。这就需要绕开syno的自动cfgen，重命名/usr/syno/cfgen/s00_synocheckfstab to k00_synocheckfstab这样etc/fstab就不会改了。
>> 如果不存在sda4,sudo parted /dev/sda mkpart primary 4 100%，然后格式化sudo mkfs.btrfs /dev/sda4
>> 然后sudo vi /etc/rc,在开头加上：
>> echo “/dev/sdb4 /volume1 btrfs nospace_cache,synoacl,relatime 0 0” > /etc/fstab
>> mount -a
>> 如果不存在/tmp/space,volume1/@databases,volume1/@tmp，则tar zxvf /template.gz -C /
>> 如果是ext4，直接在/etc/rc开头挂载mount /dev/sda4 /volume1，即可,mount -a不用加

这是代码：

```
mkdir -p /tmp/space
cat >/tmp/space/space_mapping.xml<<EOF
<?xml version="1.0" encoding="UTF-8"?>
<spaces>
	<space path="/dev/sda4" reference="/volume1" uuid="/dev/sda4" device_type="3" drive_type="0" container_type="2" limited_raidgroup_num="0" space_id="" >
		<device>
		</device>
		<reference>
			<volume path="/volume1" dev_path="/dev/sda4" uuid="/dev/vda6" type="ext4">
			</volume>
		</reference>
	</space>
</spaces>
EOF
```

进一步封装App，你可以手动将APP移到系统区：先停止app，这里以TextEdtior为例，先sudo synopkg stop TextEditor，然后sudo mv /volume1/@appstore/TextEditor /usr/local/packages/@appstore，再然后sudo ln -sfn /usr/local/packages/@appstore/TextEditor /var/packages/TextEditor/target

但最好是直接安装app到volume1做app封装，因为app启动与volume1/@tmp,volume1/@databases相关,直接转移到系统分区的APP绝大多数不依赖有没有一个volume1，但有些会要求有volume1，这时即使有原来的volume1/@tmp,@databases，启动也会出错。另外注意，把所有的依赖和核心安装全：

>> 这里我总结了spk的依赖规律。
>> 一些语言类的：
>> PHP5.6-x86_64-5.6.40-0059.spk(webstation需要，nginx是系统自带的)
>> PHP7.0-x86_64-7.0.33-0028.spk(photostation需要)
>> Perl-x86_64-5.24.0-0074.spk,PythonModule-x86_64-0114.spk(perl和pymodule cardavserver需要,py2是系统内置的)，Node.js_v8-x86_64-8.9.4-0005.spk(这个就是chat server缺失的v8,calendar也需要)
>> PHP7.2-x86_64-7.2.24-0006.spk(这个calendar也需要)
>> 和基础类的：
>> SynologyApplicationService-x86_64-1.6.2-0431.spk(calendar也需要)的都是最好要安装的。
>> 然后就是各种其它APP了。


本地验证版
-----

还记得开头说的“其实这个时候导出的镜像也没用，会失效”吗？上一步导出的镜像有可能失效。在另外的机器上启动时会出现仅加载到webassit的情况，这应该是群晖的验证过程。但是我测试时从一台新阿里云+skynas的按量组合产生的镜像，在24小时之内是不会的。所以要赶在你做好镜像的24小时内”食用“这个镜像，很沮丧是吗?我们可以把黑群弄成纯粹的本地验证。

验证的过程肯定存在于rd.gz，rd.gz相当于usb pe(它会把绑定的网卡或sn与官方比对？异或是上面的tmp?)。而且我发现，普通群晖与skynas的rd.gz中的逻辑脚本组织等，都很接近。skynas的rd.gz并没有经过深度定制。只是没找到vda变sda的地方。是否可以直接相互置换制造云上的黑群？测试安装普通黑群到阿里云，

但是安装好了的，绝对没事(这分二个过程,验证的过程永远在sda1中，而不是sda5中)，不然，安装好了的syno会重启后boot不动，这是毁灭性的。没有出现过黑群被删volume1。


>> 一些本质上的问题,任何黑群晖都不是绝对的本地验证版

>> 就像微软假sn一样，你固然可以在淘宝上买到，但如果涉及到使用某些在线服务时（比如黑苹果之于icloud，黑群晖之于quickconnect，windows之于office，共同的在线升级等），注册在官方的库中，还是可以被辨别并删掉中止服务的，一切只>> 是官方出于某些目的并没有去针对反破解，这些反破解措施中有没有包括删掉你的数据不可得知。技术上可以绕过机器和本地，但是你没法绕过远程和人 — 除非你满足于使用封闭的本地服务，所以，任何使用破解OS这类大件都是有风险的，黑群也是。


——

读到这里的小伙伴都赢了。让我在这里看到你的头像

接下来还会有一篇文章。Dsm as deepin mate(3)：更安心全面地使用syno as icloud,我们将讲解app中那些关于个人办公及日常使用的portal,erp性质的app（email,cardav,calendar,etc..），探讨其可以无缝代替iCloud的方式，如何利用cloudstation+cloudsync作三地异地，探讨其同步算法与原理。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107558519/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



