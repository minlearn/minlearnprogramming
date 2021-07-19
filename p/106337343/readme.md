发布一统tinycolinux，带openvz，带pelinux,带分离目录定制（3）
=====

__本文关键字：从0开始编译tinycolinux,busybox openssh nss，改变openssh中的passwd文件引用路径__

在《发布一统tinycolinux，带openvz，带pelinux,带分离目录定制》系列的前二篇中，我们以修改BB和GLIBC的方式创建了一个/system下的rootfs并编译了诸多工具使之得以正常启动和进行开发（gcc以提供toolchain,openssh以能本地登录和ssh登录）并为它配备了一个livelinuxpe环境以调试，并在这个rootfs中安装实现了openvz，组建这样一个文件系统还是需要实践者具备相当的动手的决心的，因为你需要亲自解决N多大小问题，但所幸结果是在成功预期里的，接下的问题依然是使这个rootfs不断完善，比如将/下还需依赖原文件系统的某些文件夹消除，在本系列文章第二篇中，我们消除了/bin,/lib在那里我们谈到/etc下的passwd,shadow还在被引用，那么在这篇中，我们将要消除的就是这个/etc和/dev

来分析问题，我们现在的状况是：在/etc下和/system/etc下都存在passwd和shadow，通过su root，adduser 或 passwd root发现，实验结果修改了/system/etc/passwd但是没有修改/system/etc/shadow却修改了/etc/shadow，adduser xxx的时候提示语也表现为：没有在/etc/shadow中找到当前创建用户名的条目使用的是/system/etc/passwd，当初编译的时候，BB的include/libbbb.h中所有的#define _PATH_XXX都改变成为了/system/etc/xxx包括passwd和shadow，而glibc/sysdeps/generic/paths.h中的_PATH_SHADOW也是改变成了/system/etc/shadow的，可是我们没有找到_PATH_PASSWD的选项。glibc和bb在使用passwd,shadow方面，当bb配置为CONFIG_BB_PWD_GRP和ENABLE_BB_SHADOW时理应使用的是新定义的/system/etc/xxx下的文件。BB按理那些关于使用自身shadow,passwd的选项（在login management中）应该足于支撑让它无须任何GLIBC的NSS机制独立完成。

实现本地登录引用/system/etc/passwd,shadow
-----

继续来做测试，如果我们删掉/etc/passwd，是可以在本地登录的，但删除/etc/shadow本地不可登录，删掉/system/etc/shadow则无关紧要。于是，问题就是：1）本地登录机制引用/system/etc/passwd是我们预期的结果，但还在引用/etc/shadow而非/system/etc/shadow是意外，是不是libbb.h中那条#define _PATH_SHADOW没有起作用呢，被glibc冲突了呢，不知道是不是冲突，但BB配置SHADOW没起作用这个事实至少是真的。那句adduser xxx的提示语也说明了一切。

于是把BB中所有提到_PATH_SHADOW的都硬编码为“/system/etc/passwd”，我的方式是windows下用7z打开busybox.src.tar，里面打开那个主文件夹，然后搜索压缩包内所有含“PATH_SHADOW”的文件，发现有chpasswd.c,adduser.c,passwd.c,deluser.c,pwd_grp.c,libbb.h,,一一硬编码替换掉，然后重新编译生成BB复制到/system/bin，特别是那个pwd_grp.c，这个文件中的三处硬编码直接关乎我们最终想要的结果。

事实是，修改完的BB完全实现了我们的预期：本地登录引用/system/etc/passwd,/system/etc/shadow就能登录。

实现远程SSH登录也引用/system/etc/passwd,shadow
-----

继续来做测试，我们发现删除/etc/shadow或者/etc/passwd中任一个，远程登录都登录不了，这其实仅仅是openssh导致的问题：是否它被设置成使用系统密码验证时，使用shadow,passwd的方式是偏向GLIBC的，分析它的源码就可发现使用了getpwnam这样的nss函数。

于是追溯源头，用7z搜索glibc2.12.1.src.tar，发现nis/nss_compat/compat_pwd.c,compat_spwd.c中都有"/etc/passwd"字样，nss作为一个注册表查找程序，从这里定义查找文件位置，openssh引用是glibc的shadow,passwd机制，于是改成compat_pwd.c中将其改成“/system/etc/passwd”，compat_spwd.c中将/etc/shadow字样改成/system/etc/shadow，编译覆盖原/system下的glibc

发现远程也能直接使用/system/etc/xxx验证了。

--------------

openvz实际也是一种应用空间。。因为它的系统包可以是一个不完整的rootfs，是不是这样呢？一个设想，日后求验。

本篇过后，本系列应该不会再出新文章了。关注我。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106337343/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



