利用openfaas faasd在你的云主机上部署function serverless面板
=====

__本文关键字：自建云函数后端。self build serverless function as service,single node serverless__

在前面《云主机上手动安装PAI面板》中我们讲到了在云主机上安装某种“类似baota xx语言项目管理器”的虚拟主机管理面板，也提到它并不是cloudbase版的云函数面板，后者这种方案要重得多：

function serverless最初也是由一个专家一篇文章给的思路，然后业界觉得好用就流行起来了。vs 传统虚拟主机管理面板和language backend as service，它至少有下面几个显著的不同特点：1），它将服务托管细粒化到了语言单位，即函数调用，故名faas，2），它与流行的API分离前后端结合，对这种webappdev有支持。3），它利用了devops docker, 可scaleable集群的部署，还记得我们《利用colinux打造serverfarm》一文吗？。4）运营上它支持按需按调用计费，将语言按调用次数收费。5) 它面向来自内部外部多种不同服务交互的混合云，构成的API调用环境。

> 它自动化了好多部署和开发级的东西以devops，以容器为后端，Triggers是一个重要组件，从GATEWAY代理中提取函数。根据触发从容器中fork一个process出来（因此与那些纯k8s和swarm的管理面板直接提供docker级别的服务粒度不同）。这个process就是watchdog 它是一种similar to fastCGI/HTTP的轻量web服务器，提供函数服务。由于支持多种环境多种不同服务交互，因此main_handler()中总有event指定事件来源，支持event,content为参数的async函数书写方式（而这，是nodejs的语言支持精髓）。。

综上，它是某种更倾向于“云网站管理面板”的思路。and more ...开源界的对应产品就是openfaas这类。

openfaas一般使用到k8s这种比较重的多节点docker管理器。注重集群可伸缩的云函数商用服务。那么对于个人，只是拿来装个云主机搭个博客，不想用到服务端的云函数（虽然有免费额度，不过总担心超）的用户，有没有更轻量的方案呢？

这就是faas containerd serverless without kubernetes:faasd，它其实也是一种openfaas的后端，只不过它使用containerd代替后端容器管理，因此它也可以To deploy embedded apps in IoT and edge use-cases，项目地址，http://github.com/openfaas/faasd/
，作者甚至在树莓派上运行了它。

好了，下面在一台1h2g的云主机上来安装它，测试在ubuntu18.04下进行。

基础
-----

以下脚本从项目的cloudinit.txt提取，有改正和修补。注意使用说明：外网访问云主机需开8080，如果提示Get http://faasd-provider:8081/ namespace=: dial tcp: i/o timeout之前，把你的云主机对外的8081打开，最好都打开。

一些变量

```
MIRROR_PATH="http://default-8g95m46n2bd18f80.service.tcloudbase.com/d/demos"
# the openfaas backend
OPENFAAS_PATH=${MIRROR_PATH}/faasd
```

安装依赖

```
apt-get install nginx golang python git runc python-certbot-nginx -qq -y
不安装runc会导致containerd可能出现oci runtime error，导致启不动faasd
```

安装faasd
-----

1.3.5有个link错误，所以换用1.3.3。

```
# install faasd
installOpenfaasd() {

    echo "=====================containerd install progress======================="
    msg=$(wget -qO- ${OPENFAAS_PATH}/containerd/v1.3.3/containerd-1.3.3-linux-amd64.tar.gz > /tmp/containerd.tar.gz && tar -xvf /tmp/containerd.tar.gz -C /usr/local/bin/ --strip-components=1

    cat << 'EOF' > /etc/systemd/system/containerd.service

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload && systemctl enable containerd
    systemctl start containerd 2>&1)
    status=$?
    updateProgress 50 "$msg" "$status" "containerd install"

    echo "=====================cni install progress======================="
    msg=$(/sbin/sysctl -w net.ipv4.conf.all.forwarding=1
    mkdir -p /opt/cni/bin
    wget -qO- ${OPENFAAS_PATH}/containernetworking/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tgz > /tmp/cni-plugins-linux-amd64-v0.8.5.tgz && tar -xvf /tmp/cni-plugins-linux-amd64-v0.8.5.tgz -C /opt/cni/bin 2>&1)
    status=$?
    updateProgress 60 "$msg" "$status" "cni install"

    echo "=====================faasd install progress(this may take long and finally fail due to network issues,you can manual fix later)======================="
    msg=$(wget -qO- ${OPENFAAS_PATH}/openfaas/faasd/0.9.2/faasd > /usr/local/bin/faasd && chmod a+x /usr/local/bin/faasd

    export GOPATH=$HOME
    rm -rf /var/lib/faasd/secrets/basic-auth-password
    rm -rf /var/lib/faasd/secrets/basic-auth-user
    rm -rf $GOPATH/go/src/github.com/openfaas/faasd

    mkdir -p $GOPATH/go/src/github.com/openfaas/
    cd $GOPATH/go/src/github.com/openfaas/ && git clone https://github.com/openfaas/faasd && cd faasd && git checkout 0.9.2
    cd $GOPATH/go/src/github.com/openfaas/faasd/ && /usr/local/bin/faasd install
    sleep 60 && systemctl status -l containerd --no-pager
    journalctl -u faasd-provider --no-pager
    systemctl status -l faasd-provider --no-pager
    systemctl status -l faasd --no-pager 2>&1)
    status=$?
    updateProgress 90 "$msg" "$status" "faasd install"


    echo "=====================faas-cli install progress======================="
    msg=$(wget -qO- ${OPENFAAS_PATH}/openfaas/faas-cli/0.12.9/faas-cli > /usr/local/bin/faas-cli && chmod a+x /usr/local/bin/faas-cli && ln -sf /usr/local/bin/faas-cli /usr/local/bin/faas
    sleep 5 && journalctl -u faasd --no-pager
    cat /var/lib/faasd/secrets/basic-auth-password | /usr/local/bin/faas-cli login --password-stdin 2>&1)
    status=$?
    updateProgress 100 "$msg" "$status" "faas-cli install"
}
```

整个脚本跟pai安装脚本的风格很类似。可以像pai一样把nginx也整合起来作为总前端(openfaas+faasd也是前后端的一种说法)，把8080转发到nginx，要知道，nginx是通用协议转发器不只http，见《基于openresty前后端统一，生态共享的webstack实现》。

以上这些如果无误完成。在云主机上可以打开8080(faasd),8081(faasd-provider)等。打开8080需要登录。

如果打不开8080，可能是脚本faasd up时从docker.io下载的几个必要小images时timeout了。cd /var/lib/faasd/ && /usr/local/bin/faasd up（一定要观察看到几个小images下完，可能会提示8080已被占用）。重启即可访问8080。

在云主机上sudo cat /var/lib/faasd/secrets/basic-auth-password得到网关密码。用户名是admin，然后部署云函数：faas-cli store deploy figlet --env write_timeout=1s。系统可能依然会开二个实例，设成仅1个也可以。由于faas-cli都是一样的，其它相关适用的高级用法可以继续关注faasd相关文档得到。

然后，，，就是把运行在cloudbase的云函数移过来，可能需要一些补正，跑在自己的服务器上，好处是不用再担心额度了，省心省事。

-----

不过说真的我对于这种docker做的虚拟化不放心，最好不要存数据。所以还是选择pai，未来整合pai,faas试试？

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108606218/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



