基于虚拟机的devops套件及把dbcolinux导出为虚拟机和docker格式
=====

__本文关键字：hashi packer,devops backend cloudcomputing,制作dbsubcolinux的docker模板,零配置，自动化开发件__

在前面我们多次谈到cloud ide & liveeditor,用nas+docker作devops等集成思想，所谓devops，它是一个集成语言系统，开发件，运维件，IDE tools，容器环境的所有自动化开发相关的东西,Devops是一个很巨大的工程，它越来越作为IT的基础存在，比如甚至这个过程和生态整合进了系统和运维 —— 这里的需求是：统一开发，多节点分布式部署，持续发布，大型网站的负载均衡架构，有点接近devops backend computing，—— 但更显然，它最开始主要服务于开发devops backend appdev。

我在bcxszy选型中，与之对应的概念就是engitor （这个命名带有点使cloud回归native的概念），vs devops 我的思想要更超前一点，尽量零配置，自动化，使得用户仅依赖它，就可以动手开发不管其它的东西。这个东西要像linux rootfs一样可以配置出来,作为某种系统基础件built into a os就好了,有点类似devops ci backend os的意思。。除了xaas的dbcolinux，在《选型》中devops的engitor它就是大头了。

>在现实生活中，devops虽然出现较晚，但对比物很多。这些都在我们以前与devops和cloud ide相关的文章中提到过，比如，gitlab是一个集成化的ci,cd平台。它是基于git的devops，这里git只是分布版本提交器与devops没有太大关系，gitlab runner才开始有devops，我们以前也提到docker可用devops，单纯的docker只是一个容器还没有持续的概念存在docker composer才刚有持续构建的概念存在。另外一个例子，就是mac osx的xcode ide,它是ide的devops可启用build bot不过它也是对其它工具的调用,还有vagrant,Jenkins,chef这些。———  所以，往往devops是对所有这些工具的结合运用，最终达到一种称为CD，CI的过程，注意这个持续C —— 这其实本质就是一种代码化加自动化思想，CI的思想本质就是就是把一切代码化。就像脚本语言一切化一样，碎片化了就可以做任何集成层的持续过程事情,比如docker有composer这是语义化了。实际上是可编程部署的容器（写yal语法）和自动化部署（这就有了devops），vagrant这种用命令行vagrant up,etc..和可脚本配置方式控制虚拟机，所以也有devops可能。

>我们还谈到terralang是一种devops语言，其实所有脚本语言都可源码文档化，可集成式发布和构建应用。比如实现terralang，使编译期和运行期分离的那个macro系统，它就是一个命令行IDE。就是一种devops的运用。当然，还有很多，很多。。。。。。

基于虚拟机的devops:hashicorp tools
-----

最近我用上了osx也使用了parallels desk，主要是冲着business和pro版有devops支持去的，它主要用的是vagrant，上面说到它使一切有了devops可能，vagrant有for parallels desk插件，不过它自己也有虚拟机部件，Vagrant就相当命令行代替了parallels desk的图形环境，vagrant 甚至还可以与 docker 结合来用，parallels desk在osx上也就变成了一个基于虚拟机的full devops软件。—— 这也是《聪明的osx云》那文我们讲到的它自带devops，而实际上它的xcode ide也是.

说到这个vagrant，它是hashi corp的东西，它自己也有一个七件套，涵盖虚拟机为中心展开devops的方方面面。以前我们主要谈到docker,git为中心的devops，docker作为轻量虚拟机性能固然好，但是它不能管从0开始，内核定制方面的事情，而vagrant可以。我们这里只讲构建，即与vagrant联系紧密的一个叫packer的工具的使用，它可以制造供vagrant使用的镜像。—— 这里只讲构建，而且用virtualbox代替vagrant，因为packer支持广泛 ——— 支持docker,支持各种虚拟机，甚至云主机，而且vb又有一个图形界面。

我们采用的例子是《将tinycolinux以硬盘模式安装到主机》，在那文中，我们以前手工构建是先发明一个liveos，现在我们不需要了，因为借助packer我们可以直接在虚拟机上构建，试错，调试。因为packer工具就一套以虚拟机为中心的devops语法支持的，持续构建器。

了解原理，准备基础环境
-----

我使用的是osx上的packer_1.4.1_darwin_amd64.zip，VirtualBox-6.0.8-130520-OSX.dmg，Packer的文档在官网上都有，原理大约是先提供一个host os - 一个在其中构建的宿主，我们是在tinycolinux live os上定制新的tinycolinux hd，所以要先准备一个ISO，和各种它的tczs，生成的硬盘镜像就是我们需要的。

在启动构建过程中，packer会自动生成硬盘和虚拟机配置信息（下述基础脚本已写明），然后就是准备builders，和执行provisioners脚本了。然后将这些写入硬盘。

* 安装virtualbox
* 下载packer放入本地os的/usr/local/bin
* 按下面基础脚本要求准备microcore_3.8.4.iso和各种tczs材料。


准备基本脚本
-----

```
{

  "_comment": “整个builders段，是定制这个iso出来的live os， 直到产生一个ssh登录服务，然后 provisioners接手。此时并未重启，除非你在脚本或inline中指定reboot”,
  "builders":
	[{
	"type": "virtualbox-iso",
	"_comment": “Linux_64这个virtualbox的默认模板，里面有默认的配置，当然你可以定制加入其它参数，具体翻packer文档”,
	"guest_os_type": "Linux_64",
	"iso_url": "./microcore_3.8.4.iso",
	"iso_checksum": "41cbc86443cc12bfbf7b03c4965e4a171ac1aa993017aeef8d04c78db73c6afb",
	"iso_checksum_type": "sha256",
	"ssh_username": "tc",
	"ssh_password": "tc",
	"boot_wait": "4s",
	"shutdown_command": "sudo poweroff",
	"_comment": "http_directory会产生一个主机上的http服务器，下面pkgs里会有3.x,tcz这样的文件夹结构,tce-load -w才能正常下载到”,
	"http_directory": "pkgs/",

	"boot_command":
		[	
	  	"<enter><wait10>",
	  	"ifconfig",
	  	"<return>",
		"sudo rm /opt/tcemirror && sudo touch /opt/tcemirror<return>",
		"_comment": “这个10.0.2.2是主机相对于虚拟机nat网卡能访问到的地址”,
		"sudo sh -c 'echo http://10.0.2.2:{{ .HTTPPort }}/ > /opt/tcemirror'",
		"<return>",
		"_comment": “每个tcz要从官网下好dep,md5.txt文件”,
	  	"tce-load -iw openssh.tcz<return><wait10>",
	  	"sudo passwd tc<return>",
	  	"tc<return>",
	  	"tc<return>",
		"sudo cp /usr/local/etc/ssh/sshd_config.example /usr/local/etc/ssh/sshd_config<return><wait>",
	  	"sudo /usr/local/etc/init.d/openssh start<return><wait>"
	  	],

	  "export_opts": 
	  	[
	  	"--manifest",
	  	"--vsys", "0",
	  	"--description","{{user `vm_description`}}",
	  	"--version", "{{user `vm_version`}}"
	  	],
	  "format":"ova",
	  "vm_name":"dbcolinux"
	}],


  "_comment": “整个provisioners段，开始为这个guestos作各种写入硬盘（那个linux_64自动生成的一块40g的硬盘）作上述export前的过程，当然，如何写入，脚本里要写明”,
  "provisioners": 
	[{
    	"type": "shell",
    	"pause_before":"1s",
    	"execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'",
    	"inline":
		[
		"cp -R /tmp/tce ~/"
		]
	},

	{
    	"type": "shell",
    	"pause_before":"1s",
    	"inline":
		[
		"tce-load -iw parted.tcz",
		"tce-load -iw grub2.tcz"
		]
	},

	{
    	"type": "shell",
    	"pause_before":"1s",
    	"execute_command": "echo '' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'",
    	"scripts":
		[
		"./scripts/phase1.sh"
		]
    	}]

}
```

假设上面构建文件保存为dbcolinux-pe.packer,调试和启动构建的方法：

进入正确的目录，执行packer build ./dbcolinux-pe.packer(for test and debug, you can use -debug -on-error=ask after build command)，用了ask，它会问，你调试完后选择clean即可。不要立即作答。如果用了-debug会走一步卡一步让你确认。

Scripts中就是如何写入硬盘，写入哪些东西形成硬盘系统的逻辑，这是一个字符数组，目前只有phase1一个文件

scripts->phase1中的内容
-----

```
export PATH=$PATH:/usr/local/sbin:/usr/local/bin

echo PREPARE HD
parted /dev/hda mktable msdos
parted /dev/hda mkpart primary ext3 1% 99%
parted /dev/hda set 1 boot on
mkfs.ext3 /dev/hda1
parted /dev/hda print
rebuildfstab
mount /mnt/hda1

echo COPY SOFT
echo /usr/local/etc/init.d/openssh start >> /opt/bootlocal.sh
echo usr/local/etc/ssh > /opt/.filetool.lst
echo etc/passwd>> /opt/.filetool.lst
echo etc/shadow>> /opt/.filetool.lst
/bin/tar -C / -T /opt/.filetool.lst -cvzf /mnt/hda1/mydata.tgz
mv ~/tce /mnt/hda1/
cp -R /opt /mnt/hda1

echo INSTALLING GRUB
grub-install --boot-directory=/mnt/hda1/boot /dev/hda
mkdir /mnt/cdrom/
mount /dev/cdrom /mnt/cdrom
cp /mnt/cdrom/boot/microcore.gz /mnt/hda1/boot/microcore.gz
cp /mnt/cdrom/boot/bzImage /mnt/hda1/boot/bzImage
echo set timeout=3 > /mnt/hda1/boot/grub/grub.cfg
echo menuentry \\\"dbcolinux\\\" { >> /mnt/hda1/boot/grub/grub.cfg
echo  linux /boot/bzImage com1=9600,8n1 loglevel=3 user=tc console=ttyS0 console=tty0 noembed nomodeset tce=hda1 opt=hda1 home=hda1 restore=hda1 >> /mnt/hda1/boot/grub/grub.cfg
echo  initrd /boot/microcore.gz >> /mnt/hda1/boot/grub/grub.cfg
echo } >> /mnt/hda1/boot/grub/grub.cfg

#reboot
```

——————

懂得了原理，基本够用的语句，或许以后，我们会把所有的xaas->dbcolinux上写过的文章中的构建全部按lesson实例写一次。

关注我


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106336778/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





