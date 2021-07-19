在tc上安装buildkit.tcz，vscode.tcz，打通vscodeonline与openfaas模拟cloudbase打造碎片化编程开发部署环境
======

__本文关键字：rebuild kernel invalid magic number,failed to create diff tar stream: failed to get xattr for : operation not supported__

在《一种用buildkit打造免registry的local cd/ci工具,打通vscodeonline与openfaas模拟cloudbase打造碎片化编程开发部署环境的设想》中，我们介绍了方案和设想，本文将用测试说话，在tc11上实践上文谈到的内容：

> 这里提个上文遗漏的内容，为什么我们要说vscodeonline+openfaas这种环境是碎片化呢？因为这一切发生在云上，我们可以随时用碎片化时间断续编程，而且buildkit commit方案直接快速本地形成“修改->发布”的容器循环，而且如果你在云函数中用的是js这种带universal web/desktop app支持的云原生语言（说实话，只有js是云和web原生的而且实现全栈，因为它是内置语言自带环境browser,v8 nodejs，其它都是走通用化语言->支持webframework，更多关注仅后端的路子出来的，js像是一种编辑器和编辑器脚本语言更具DSL特性，）当然，这些都是后端微服务化，持续集成化，那么前端碎片化呢，全碎片化前端+碎片化后端才是降低程序规模以降低难度的标配，而Electron这样的产品（nodejs+chrome core形成的app front），可以把前端文件托管在后端，动态加载这些”前端资源“，达成wx小程序这样的快速原型，这也是“小”，“碎片”程序的意义表现之一。

buildkit.tcz+instant commit
-----

首先，从下载buildkit最新版，它跟docker,openfaas,containerd一样，都是清一色的go binary，采用《一种混合包管理和容器管理方案，及在tinycorelinux上安装containerd和openfaas》同样的tcz构建方法，将下载到的v0.8.1的buildkit包的所有bin放到一个squashfs-root/usr/local/bin中(不用加chmod + x因为下载包里自带)，然后新建一个squashfs-root/usr/local/etc/init.d/，里面放二个chmod +x ./buildkit ./makedockerconfig:

这是makedockerconfig中的内容，buildkit+containerd代替了整个docker作构建工具和运行时(containerd不能build,ctr image build没有这个命令),但配置文件用的还是对接dockerhub的那一套。

```
read -p "enter your dockerhub username:" DOCKERHUB_USERNAME
read -p "enter your dockerhub personal access token:" DOCKERHUB_TOKEN
rm -rf ~/.docker/config.json
mkdir -p ~/.docker/
cat > ~/.docker/config.json << EOF
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "$(echo -n $DOCKERHUB_USERNAME:$DOCKERHUB_TOKENT | base64)"
        }
    }
}
EOF
cp -f ~/docker/config.json /var/lib/faasd/config.json
```

解释一下，所谓access token就是一种能代替密码，实现有限权限子账户的机制。dockerhub后台可以得到。由于tc11是没有base64的，这个工具在coreutils.tcz中。稍后建成的buildkit.tcz将依赖一条coreutils.tcz

这是buildkit中的内容，/usr/local/bin/buildkitd &，（你可以按上文添加额外参数设置为runc或containerd后端，默认为runc），为应用包在/init.d/写随着系统启动的启动文件，特别要注意，在命令后必要处加&，否则前台命令会block启动流程。在前文中，faasd和containerd都是这样处理的。

写好dep（依赖coreutils和containerd,attr.tcz:buildkit使用xattr相关命令）和md5.txt，打包成tcz,安装在tc11中，现在来测试一下，建立一个测试仓库并：sudo buildctl build --frontend dockerfile.v0 --local context=./ --local dockerfile=./ --output type=image,name=docker.io/minlearn/dafsdf:latest，出错了：

```
......
=> ERROR exporting to image 0.1s
error: failed to solve: rpc error: code = Unknown desc = mount callback failed on /tmp/containerd-mount267804283: mount callback failed on /tmp/containerd-mount706848158: failed to write compressed diff: failed to create diff tar stream: failed to get xattr for /tmp/containerd-mount267804283/bin: operation not supported
```

这是因为我在tc11中使用的是ext3，构建tc11用的config-5.4.3-tinycore64中并没有开启CONFIG_EXT3_FS,也没有开启CONFIG_EXT3_FS_XATTR，导致buildkit调用xattr相关命令时不成功。因此重新编译内核。注意要sudo make install，不能仅sudo make在arch/boot/x86/compressed下得到vmlinux，而要在install过后的/boot下得到vmlinuz。否则虽然编译成功，但kernel image运行不了，会提示invalid magic number

安装好kernel image,重启，问题解决。你就可以实现在local容器中免registry commit了。最后在命令中按是否需要上传到dockerhub，加个push=true一下。

当然这一切现在只是命令行方式进行，并没有上升到整合为openfaas 8080/ui那个后台的界面功能。但作为上文提到的方案2，接下来的vscode mount就好多了。

vscode.tcz + mount
------

我们下载的是cdr的code-server-3.8.0-amd64，按 《panel.sh：一个nginx+docker的云函和在线IDE面板,发明你自己的paas(2)》的路子将所有文件解压到squashfs-root的/usr/local/lib/中。然后新建squashfs-root/usr/local/bin,squashfs-root/usr/local/etc/init.d/并依次：

cd squashfs-root/usr/local/bin
sudo ln -s ../lib/code-server-3.8.0/bin/code-server code-server
sudo chmod +x ./code-server

cd squashfs-root/usr/local/etc/init.d/
sudo touch vscodeonline makevscodeconfig
sudo chmod +x ./vscodeonline makevscodeconfig

makevscodeconfig里面放：

```
read -p "enter your desired access token(plain strings ok):" VSCODE_TOKEN
rm -rf ~/.config/code-server/config.yaml
mkdir -p ~/.config/code-server/
cat > ~/.config/code-server/config.yaml << EOF
bind-addr: 127.0.0.1:5000
auth: password
password: "$(echo $VSCODE_TOKEN)"
cert: false
EOF
mkdir -p /home/tc/.config/code-server/
cp -f /root/.config/code-server/config.yaml /home/tc/.config/code-server/config.yaml
```

然后是启动vscodeonline的：/usr/local/bin/code-server --config /root/.config/code-server/config.yaml &

直接打包，直接md5,没有dep引用。
--config ~/.config/code-server/config.yaml是需要的，因为code-server似乎存在一个bug，它启动的时候会在~.config下找配置文件，而不是~/.config少了一个/(多了一个/?没仔细看)，且它识别不了~，找不到会自动在8080启动vscode服务，导致发生异常，故强行指定/root/.config/。

然后你就可以按https://code.visualstudio.com/docs/remote/containers-advanced尝试vscode.tcz + mount方案了

未来我们将code-server中绑定的nodejs发行版本独立出来用独立tcz代替，这才符合tcz一个软件一个包的适当粒度划分。

-----

buildkit commit方案已经可以即时将构建修改循环的时间压缩到很短了，但第一次拉取和构建镜像依然是从dockerhub，耗时巨大，我们在后来的文章中将探索用onedrive配合onemanager等程序，打造将OS发行的软件源和程序语言模块仓库，在线商店等，统统托管成平坦结构，放进od的方法。将OD发展为程序员专用，人手一套的私人registry网盘，配合ctr images offline tar import。这样，以后的时间将只耗费在我们本文提到的免registry commit的ci/cd过程中。


