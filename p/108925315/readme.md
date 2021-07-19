panel.sh：一个nginx+docker的云函和在线IDE面板,发明你自己的paas(1)
=====

__本文关键字：Cannot connect to the Docker daemon at，containerd cannot properly do "clean-up" with shim process during start up，用标准方法实现的类群晖paas，with debugable appliance inside built__

在前面《利用openfaas faasd在你的云主机上部署function serverless面板》中我们介绍了用从https://github.com/openfaas/faasd/tree/0.9.2/cloud-config.txt提取的脚本安装openfaas（后来我们用上了0.9.5），和在云主机上使用它的方法，见《在openfaas面板上安装onemanager1，2》，如果说这3文定位主要是基本安装，排错，和调试，那么本文开始就着重于增强和提高脚本的体验了。前3文的成果和努力依旧有效。

第一个问题，脚本要能在一台干净的ubuntu1804的机器上安装，尽量一次成功，如果不能成功，那么它也要求能多次覆盖安装不致于弄坏系统。这就要求脚本中安装的组件分开，各组件包括其配置要standalone方式放置，这样可以重装时拔插和替换，覆盖。

第二个问题，虽然我前3文中从来没遇到过，但是后来的尝试中，我发现在ubuntu1804同样的安装方式和组件版本（v1.3.3containerd+cni0.4.0+cniplugins0.8.5+faasd0.9.5），居然gateway那个container开启一会之后就会停止，导致8080根本不能访问。

不废话了，直接上新的脚本：

前置
-----

更新了安装说明。集中化了全局变量，注意deps prepare部分，bridge-utils是为了控制cni制造的那个openfaas0虚拟网卡用。安装docker.io，是ubuntu上它可以同时安装containerd1.3.3和runc

>  从Docker 1.11开始，Docker容器运行已经不是简单的通过Docker daemon来启动，而是集成了containerd、runC等多个组件。如果去搜索一番，就会发现：docker-containerd 就是 containerd，而 docker-runc 就是 runc。containerd是真正管控容器的daemon，执行容器的时候用的是runc。
> 为什么 要分的七零八散呢？为了防止docker一家独大,docker当年的实现被拆分出了几个标准化的模块,,标准化的目的是模块是可被其他实现替换的,其实也是为了实现类llvm的组件化可开发效果(软件抽象上，源头如果有分才能合，如果一开始就是合的就难分)。也是为了分布式效果。docker也像git一样做分布式部件化了，分布式就是设置2个部件，cliserver，这样在本地和远程都可这样架构。

> 而为什么是dockerio而不是docker-ce:
> 事实是我还发现，有些系统上，安装了docker-ce再安装containerd。会导致系统出问题Cannot connect to the Docker daemon at 。docker与containerd不兼容，所以只好安装ubuntu维护的docker.io这种解决了containerd依赖的，它默认依赖containerd和runc(不过稍后我们会提到替换升级containerd的版本)。我们选用加入了最新cn ubuntu deb src后apt-get update得到的sudo apt install docker.io=19.03.6-0ubuntu1~18.04.1，sudo apt-cache madison docker.io出来的版本.


```
#!/bin/bash

## currently tested under ubuntu1804 64b,easy to be ported to centos(can be tested with replacing apt-get and /etc/systemd/system)

## How to use this script in a cloudhost
## su root and then: ./panel.sh -d 'your domain to be binded' -m 'email you use to pass to certbot' -p 'your inital passwords'(email and passwords are not neccessary,feed email only if you encount the "toomanyrequestofagivetype" error)
## (no prefix https/http needed,should bind to the right ip ahead for laster certbot working)


export DOMAIN_NAME=''
export EMAIL_NAME='some@some.com'
export PANEL_TYPE='0'
export PASS_INIT='5cTWUsD75ZgL3VJHdzpHLfcvJyOrUnza1jr6KXry5pXUUNmGtqmCZU4yGoc9yW4'

MIRROR_PATH="http://default-8g95m46n2bd18f80.service.tcloudbase.com/d/demos"
# the pai backend
SERVER_PATH=${MIRROR_PATH}/pai/pai-agent/stable/pai_agent_framework
PAI_MATE_SERVER_PATH=${MIRROR_PATH}/pai/pai-mate/stable/install
# the openfaas backend
OPENFAAS_PATH=${MIRROR_PATH}/faasd
# the code-server web ide
CODE_SERVER_PATH=${MIRROR_PATH}/codeserver

#install dir
INSTALL_DIR="/root/.local"
CONFIG_DIR="/root/.config"
# datadir only for pai and common data
DATA_DIR="/data"


while [[ $# -ge 1 ]]; do
    case $1 in
      -d|--domain)
        shift
        DOMAIN_NAME="$1"
        shift
        ;;
      -m|--mail)
        shift
        EMAIL_NAME="$1"
        shift
        ;;
      -t|--paneltype)
        shift
        PANEL_TYPE="$1"
        shift
        ;;
      -p|--passinit)
        shift
        PASS_INIT="$1"
        shift
        ;;
      *)
        if [[ "$1" != 'error' ]]; then echo -ne "\nInvaild option: '$1'\n\n"; fi
        echo -ne " Usage(args are self explained):\n\tbash $(basename $0)\t-d/--domain\n\t\t\t\t\-m/--mail\n\t\t\t\t\-t/--paneltype\n\t\t\t\t-p/--passinit\n\t\t\t\t\n"
        exit 1;
        ;;
      esac
    done

[[ "$EUID" -ne '0' ]] && echo "Error:This script must be run as root!" && exit 1;

beginTime=$(date +%s)

# write log with time
writeProgressLog() {
    echo "[`date '+%Y-%m-%d %H:%M:%S'`][$1][$2]"
    echo "[`date '+%Y-%m-%d %H:%M:%S'`][$1][$2]" >> ${DATA_DIR}/h5/access.log
}

# update install progress
updateProgress() {
    progress=$1
    message=$2
    status=$3
    installType=$4

    # echo "=====================$installType progress======================="
    echo "=======================$installType progress=======================" >> ${DATA_DIR}/h5/access.log
    writeProgressLog "installType" $installType
    writeProgressLog "progress" $progress
    writeProgressLog "status" $status
    echo $message >> ${DATA_DIR}/h5/access.log

    if [ $status == "0" ]; then
      code=0
      message="success"
    else
      code=1
      message="$installType error"
      # exit 1
    fi

    cat << EOF > ${DATA_DIR}/h5/progress.json
{
    "code": $code,
    "message": "$message",
    "data": {
        "installType": "$installType",
        "progress": $progress
    }
}
EOF

    if [ $status == "0" ]; then
      code=0
      message="success"
    else
      code=1
      message="$installType error"
      # exit 1
    fi

    if [ $status != "0" ]; then
      echo $message >> ${DATA_DIR}/h5/installErr.log
    fi
}

echo "=====================begin .....====================="
echo "PANEL_TYPE: ${PANEL_TYPE}"
echo "DOMAIN_NAME: ${DOMAIN_NAME}"
echo "SERVER_PATH: ${MIRROR_PATH}"
echo "OPENFAAS_PATH: ${OPENFAAS_PATH}"
echo "PAI_MATE_SERVER_PATH: ${PAI_MATE_SERVER_PATH}"
echo "CODE_SERVER_PATH: ${CODE_SERVER_PATH}"
echo "INSTALL_DIR: ${INSTALL_DIR}"


rm -rf ${DATA_DIR}/h5
mkdir -p ${DATA_DIR}/h5
rm -rf ${DATA_DIR}/h5/index.json

rm -rf ${DATA_DIR}/logs
mkdir -p ${DATA_DIR}/logs

mkdir -p ${INSTALL_DIR}/bin
mkdir -p ${CONFIG_DIR}

echo "=====================deps prepare progress(this may take long...)======================="
msg=$( #begin
    if [ $PANEL_TYPE == "0" ]; then

    	apt-key adv --recv-keys --keyserver keyserver.Ubuntu.com 3B4FE6ACC0B21F32
    	echo deb http://cn.archive.ubuntu.com/ubuntu/ bionic main restricted universe multiverse >> /etc/apt/sources.list
    	echo deb http://cn.archive.ubuntu.com/ubuntu/ bionic-security main restricted universe multiverse >> /etc/apt/sources.list
    	echo deb http://cn.archive.ubuntu.com/ubuntu/ bionic-updates main restricted universe multiverse >> /etc/apt/sources.list
    	echo deb http://cn.archive.ubuntu.com/ubuntu/ bionic-proposed main restricted universe multiverse >> /etc/apt/sources.list
    	echo deb http://cn.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse >> /etc/apt/sources.list
    	apt-get update
    	apt-get install docker.io=19.03.6-0ubuntu1~18.04.1 --no-install-recommends bridge-utils -y
    	apt-get install nginx python git python-certbot-nginx -y
    	# sed '1{:a;N;5!b a};$d;N;P;D' -i /etc/apt/sources.list
    	# apt-get update

    else
    	apt-get update && apt-get install git nginx gcc python3.6 python3-pip python3-virtualenv python-certbot-nginx golang -y
    fi 2>&1)
status=$?
updateProgress 30 "$msg" "$status" "deps prepare"
```


基础组件代码:nginx front and docker backend
------

这部分虽然写死了各条转发。但重点在于如何根据具体的转发需要布置代码。这里的理论在于：如果代理服务器地址（proxy_pass后面那个）中是带有URI的，此URI会替换掉 location 所匹配的URI部分。 而如果代理服务器地址中是不带有URI的，则会用完整的请求URL来转发到代理服务器。

```
confignginx() {

    echo "=====================certbot renew+start+init progress======================="
    systemctl enable nginx.service
    systemctl start nginx

    #cp -f /lib/systemd/system/certbot.service /etc/systemd/system/certbot-renew.service
    #echo '[Install]' >> /etc/systemd/system/certbot-renew.service
    #echo 'WantedBy=multi-user.target' >> /etc/systemd/system/certbot-renew.service
    #cp -f /lib/systemd/system/certbot.timer /etc/systemd/system/certbot-renew.timer

    # sed -i "s/renew/renew --nginx/g" /etc/systemd/system/certbot-renew.service


    rm -rf /etc/systemd/system/certbot-renew.service
    cat << 'EOF' > /etc/systemd/system/certbot-renew.service

[Unit]
Description=Certbot
Documentation=file:///usr/share/doc/python-certbot-doc/html/index.html
Documentation=https://letsencrypt.readthedocs.io/en/latest/
[Service]
Type=oneshot
ExecStart=/usr/bin/certbot -q renew
PrivateTmp=true
[Install]
WantedBy=multi-user.target
EOF

    rm -rf /etc/systemd/system/certbot-renew.timer
    cat << 'EOF' > /etc/systemd/system/certbot-renew.timer

[Unit]
Description=Run certbot twice daily

[Timer]
OnCalendar=*-*-* 00,12:00:00
RandomizedDelaySec=43200
Persistent=true

[Install]
WantedBy=timers.target
EOF

    msg=$(
    #first time renew
    certbot certonly --quiet --standalone --agree-tos --non-interactive -m ${EMAIL_NAME} -d ${DOMAIN_NAME} --pre-hook "systemctl stop nginx"

    systemctl daemon-reload 
    systemctl enable certbot-renew.service
    systemctl start certbot-renew.service
    systemctl start certbot-renrew.timer 2>&1)
    status=$?
    updateProgress 40 "$msg" "$status" "certbot renew+start+init"


    echo "=====================nginx reconfig progress======================="
    # add nginx conf
    rm -rf /etc/nginx/conf.d/default.conf
    cat << 'EOF' > /etc/nginx/conf.d/default.conf

server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name DOMAIN_NAME;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/DOMAIN_NAME/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/DOMAIN_NAME/privkey.pem;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    location / {
      proxy_pass http://localhost:PORT;

      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

    location /pai/ {
      proxy_pass http://localhost:5523;

      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

    location /faasd/ {
      proxy_pass http://localhost:8080/ui/;

      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

    location /codeserver/ {
      proxy_pass http://localhost:5000/;
      proxy_redirect http:// https://;
      proxy_set_header Host $host:443/codeserver;

      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

}

server {
    listen 80;
    server_name DOMAIN_NAME;

    if ($host = DOMAIN_NAME) {
        return 301 https://$host$request_uri;
    }

    return 404;
}
EOF

    sed -i "s#DOMAIN_NAME#${DOMAIN_NAME}#g" /etc/nginx/conf.d/default.conf

    if [ $PANEL_TYPE == "0" ]; then
    	sed -i "s#PORT#8080/function/#g" /etc/nginx/conf.d/default.conf
    else
    	sed -i "s#PORT#3000#g" /etc/nginx/conf.d/default.conf
    fi



    # restart nginx
    msg=$( #begin
    [[ $(systemctl is-active nginx.service) == "activating" ]] && systemctl reload nginx.service
    systemctl restart nginx 2>&1)
    status=$?
    updateProgress 50 "$msg" "$status" "nginx reconfig"

}

confignginx
```

为了让docker能覆盖安装，接下来脚本开头处逻辑的清空了配置，这里的重点问题是containerd与cni，与openfaasd的复杂关系：

Container Network Interface (CNI) 最早是由CoreOS发起的容器网络规范，是Kubernetes网络插件的基础。其基本思想为：Container Runtime在创建容器时，先创建好network namespace，然后调用CNI插件为这个netns配置网络，其后再启动容器内的进程。现已加入CNCF，成为CNCF主推的网络模型。CNI负责了在容器创建或删除期间的所有与网络相关的操作，它将创建所有规则以确保从容器进和出的网络连接正常，但它并不负责设置网络介质，例如创建网桥或分发路由以连接位于不同主机中的容器。

这个工作由openfaasd等完成。docker的这些组件->containerd+cni+ctr+runc，是由faasd来配置运行的。单独启动第一次安装完的containerd+cni+ctr+runc并不会启动cni和开启网卡(单独启动containerd提示cni conf not found没关系它依然会启动)，需要openfaasd中的动作给后者带来cni和网卡配置。但这种结合很紧密，使得接下来容器的完全清理工作有难度。

对于容器的清除，用ctr tasks kill &&  ctr tasks delete && ctr container delete可以看到ps aux|grep manual看到主机空间的shim任务和/proc/id号/ns都被删掉了，但还是某些地方有残留。这是因为这二者很难分开,shim开启的task关联容器和/var/run/containerd无法清理，导致前者很难单独拔插/进行配置卸载，也难于在下一次覆盖安装时能从0全新开始。

而这其实是一个bug导致的,containerd cannot properly do "clean-up" with shim process during start up? #3971（https://github.com/containerd/containerd/issues/3971），直到1.4.0beta才被解决（https://github.com/containerd/containerd/pull/4100/commits/488d6194f2080709d9667e00ff244fbdc7ff95b2），但我测试了(cd /var/lib/faasd/ faasd up)，只是效果好点，1.3.3是提示id exists不能重建container，1.40是提示/run/container下的files exists，同样没解决完全清理以全新覆盖安装containerd的需求，所以我脚本中提示了“containerd install+start progress(this may hang long and if you over install the script you may encount /run/containerd device busy error,for this case you need to reboot to fix after scripts finished”，这个基本如果你遇到了var/run删不掉错误，等安装程序跑完，重启即可。

因此我选择了1.40的containerd，它也同时解决了我开头提到的，gateway失效的问题。用的cni plugins还是0.8.5，本来想用那个cri-containerd-cni-1.4.0-linux-amd64.tar.gz,但里面的cni是0.7.1，与faasd要求的0.4.0不符。

对于cni的卸载和清除，则不属于ctr的能控制范畴，cni没有宿主机上的控制，除非将进程网络命名空间恢复到主机目录，或在在容器网络空间内运行IP命令来检查网络接口是否已正确设置，都挺麻烦，用上面删容器的ctr tasks kill &&  ctr tasks delete && ctr container delete三部曲删可以看到ifconfig中五个task对应的虚拟网卡也被干掉了，所以我也就没有再深入研究cni的卸载逻辑。

```
configdocker() {



    [[ $(systemctl is-active faasd-provider) == "activating" ]] && systemctl stop faasd-provider
    [[ $(systemctl is-active faasd) == "activating" ]] && systemctl stop faasd
    [[ $(systemctl is-active containerd) == "activating" ]] && ctr image remove docker.io/openfaas/basic-auth-plugin:0.18.18 docker.io/library/nats-streaming:0.11.2 docker.io/prom/prometheus:v2.14.0 docker.io/openfaas/gateway:0.18.18 docker.io/openfaas/queue-worker:0.11.2  && for i in basic-auth-plugin nats prometheus gateway queue-worker; do ctr tasks kill -s SIGKILL $i;ctr tasks delete $i;ctr container delete $i; done && systemctl stop containerd && sleep 10
    ps -ef|grep containerd|awk '{print $2}'|xargs kill -9
    rm -rf /var/run/containerd /run/containerd

    [[ ! -z "$(brctl show|grep openfaas0)" ]] && ifconfig openfaas0 down && brctl delbr openfaas0
    rm -rf /etc/cni

    echo "===============================cniplugins installonly================================="
    msg=$( #begin

    if [ ! -f "/tmp/cni-plugins-linux-amd64-v0.8.5.tar.gz" ]; then
        wget --no-check-certificate -qO- ${MIRROR_PATH}/docker/containernetworking/plugins/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tar.gz > /tmp/cni-plugins-linux-amd64-v0.8.5.tar.gz
    fi

    mkdir -p /opt/cni/bin
    tar -xf /tmp/cni-plugins-linux-amd64-v0.8.5.tar.gz -C /opt/cni/bin

    /sbin/sysctl -w net.ipv4.conf.all.forwarding=1 2>&1)
    status=$?
    updateProgress 50 "$msg" "$status" "cniplugins installonly"


    echo "======containerd install+start progress(this may hang long and if you over install the script you may encount /run/containerd device busy error,for this case you need to reboot to fix after scripts finished)====="
    msg=$( #begin
    # del original deb by docker.io
    rm -rf /usr/bin/containerd* /usr/bin/ctr

    # replace with new bins
    if [ ! -f "/tmp/containerd-1.4.0-linux-amd64.tar.gz" ]; then
        wget --no-check-certificate -qO- ${MIRROR_PATH}/docker/containerd/v1.4.0/containerd-1.4.0-linux-amd64.tar.gz > /tmp/containerd-1.4.0-linux-amd64.tar.gz
    fi

    tar -xf /tmp/containerd-1.4.0-linux-amd64.tar.gz -C ${INSTALL_DIR}/bin/ --strip-components=1 && ln -sf ${INSTALL_DIR}/bin/containerd* /usr/local/bin/ && ln -sf ${INSTALL_DIR}/bin/ctr /usr/local/bin/ctr

    rm -rf /etc/systemd/system/containerd.service
    cat << 'EOF' > /etc/systemd/system/containerd.service

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target
#After=network.target containerd.socket containerd.service
#Requires=containerd.socket containerd.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process 
#changed to mixed to let systemctl stop containerd kill shims
#KillMode=mixed
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
    systemctl start containerd --no-pager 2>&1)
    status=$?
    updateProgress 50 "$msg" "$status" "containerd install+start"



}

configdocker
```


-----

未来等containerd的这个bug彻底解决或许有可能让containerd的shim task实现彻底停止和移除。来说点别的，还记得我在《enginx》中对openresty可以脚本编程转发连结游戏服务器的多组件集群，形成demo based programming的能力的设想吗(类似组成openfaas的五个containers，是组建一个单节点集群分布式的典型职责单位。有验证有网关，有业务)。还有基于jupyter的engitor，那么现在，我们用openfaas+vscodeonline来实现它们。我们知道openfaas这种就是构建一个分布式函数的“城市”，让城市组成的世界在二进制级，相互调用分布式API，进行demo组合，构成应用。是真正的demo积木编程。因为它可以Turn Any CLI into a Function，甚至是本地native cli。比如它能使shell完全变成分布式语言。直接在二进制上编程。


-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108925315/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




