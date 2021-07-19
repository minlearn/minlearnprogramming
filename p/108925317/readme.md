panel.sh：一个nginx+docker的云函和在线IDE面板,发明你自己的paas(2)
=====

继前文，这里介绍faasd和pai二个后端安装启动逻辑，也直接放代码：

faasd
=====

这里主要是将原来cloud-config.txt中cd git source root,faasd install替换成，经分析source/cmd/install.go后得到的几个静态文件，并将它们直接放进代码中，这样可以免除编译代码直接全部静态资源加脚本逻辑处理。

注意几处，1，/var/lib/faasd的权限一定要设成0755，而不是cmd/install.go中指定的0644。否则containerd启动不了，它要写hosts文件，以调用docker cni。2，结尾服务启动时，须留足充足的时间让images都下载完，只有下载完镜像才会启动网卡的。加了一些关于镜像和虚拟网卡是否准备完毕和唤起的判断，重点是离线镜像。

由于影响脚本失败的地方和可变因素主要来自镜像的拉取（及之后cni网络唤起，faasd up建立cni有一定机率让openfaas0网络出现很慢。），故设置了预拉取离线镜像并判断（本来之前计划是预配置cni，但与镜像部分结合紧密，且后来考虑到镜像拉取如果OK，cni过程应该无须再考虑）。使得onedrive+om可代替docker register仓库（github也有一个git bundle create offline-repos master，这对应我在《在openfaas面板上安装onemanager》文尾把git和docker仓库静态化的设想）。

```
# install faasd
installOpenfaasd() {

    [[ $(systemctl is-active faasd-provider) == "activating" ]] && systemctl stop faasd-provider
    [[ $(systemctl is-active faasd) == "activating" ]] && systemctl stop faasd

    ps -ef|grep faasd|awk '{print $2}'|xargs kill -9
    rm -rf ${INSTALL_DIR}/bin/faas*
    rm -rf /var/lib/faasd /var/lib/faasd-provider

    echo "=====faasd install+start progress(this may finally fail due to incorrect folder permissions or network issues,you can maunal fix it later with systemctl restart faasd)====="
    msg=$( # begin

    if [ ! -f "/tmp/faas-cli" ]; then
        wget --no-check-certificate -qO- ${OPENFAAS_PATH}/faasd/0.9.5/faasd > /tmp/faasd
    fi

    cp /tmp/faasd ${INSTALL_DIR}/bin/faasd && chmod a+x ${INSTALL_DIR}/bin/faasd && ln -sf ${INSTALL_DIR}/bin/faasd /usr/local/bin/faasd


    #export GOPATH=$HOME
    # ......
    #systemctl status -l faasd-provider --no-pager
    #systemctl status -l faasd --no-pager 

    mkdir -p /var/lib/faasd && chmod 0755 /var/lib/faasd && mkdir -p /var/lib/faasd-provider && chmod 0755 /var/lib/faasd-provider && mkdir -p /var/lib/faasd/secrets


    rm -rf /var/lib/faasd/docker-compose.yaml
    cat << 'EOF' > /var/lib/faasd/docker-compose.yaml

version: "3.7"
services:
  basic-auth-plugin:
    image: "docker.io/openfaas/basic-auth-plugin:0.18.18ARCH_SUFFIX"
    environment:
      - port=8080
      - secret_mount_path=/run/secrets
      - user_filename=basic-auth-user
      - pass_filename=basic-auth-password
    volumes:
      # we assume cwd == /var/lib/faasd
      - type: bind
        source: ./secrets/basic-auth-password
        target: /run/secrets/basic-auth-password
      - type: bind
        source: ./secrets/basic-auth-user
        target: /run/secrets/basic-auth-user
    cap_add:
      - CAP_NET_RAW

  nats:
    image: docker.io/library/nats-streaming:0.11.2
    command:
      - "/nats-streaming-server"
      - "-m"
      - "8222"
      - "--store=memory"
      - "--cluster_id=faas-cluster"
    # ports:
    #    - "127.0.0.1:8222:8222"

  prometheus:
    image: docker.io/prom/prometheus:v2.14.0
    volumes:
      - type: bind
        source: ./prometheus.yml
        target: /etc/prometheus/prometheus.yml
    cap_add:
      - CAP_NET_RAW
    ports:
       - "127.0.0.1:9090:9090"

  gateway:
    image: "docker.io/openfaas/gateway:0.18.18ARCH_SUFFIX"
    environment:
      - basic_auth=true
      - functions_provider_url=http://faasd-provider:8081/
      - direct_functions=false
      - read_timeout=60s
      - write_timeout=60s
      - upstream_timeout=65s
      - faas_nats_address=nats
      - faas_nats_port=4222
      - auth_proxy_url=http://basic-auth-plugin:8080/validate
      - auth_proxy_pass_body=false
      - secret_mount_path=/run/secrets
      - scale_from_zero=true
    volumes:
      # we assume cwd == /var/lib/faasd
      - type: bind
        source: ./secrets/basic-auth-password
        target: /run/secrets/basic-auth-password
      - type: bind
        source: ./secrets/basic-auth-user
        target: /run/secrets/basic-auth-user
    cap_add:
      - CAP_NET_RAW
    depends_on:
      - basic-auth-plugin
      - nats
      - prometheus
    ports:
       - "8080:8080"

  queue-worker:
    image: docker.io/openfaas/queue-worker:0.11.2
    environment:
      - faas_nats_address=nats
      - faas_nats_port=4222
      - gateway_invoke=true
      - faas_gateway_address=gateway
      - ack_wait=5m5s
      - max_inflight=1
      - write_debug=false
      - basic_auth=true
      - secret_mount_path=/run/secrets
    volumes:
      # we assume cwd == /var/lib/faasd
      - type: bind
        source: ./secrets/basic-auth-password
        target: /run/secrets/basic-auth-password
      - type: bind
        source: ./secrets/basic-auth-user
        target: /run/secrets/basic-auth-user
    cap_add:
      - CAP_NET_RAW
    depends_on:
      - nats
EOF

    sed -i "s#ARCH_SUFFIX##g" /var/lib/faasd/docker-compose.yaml

    rm -rf /var/lib/faasd/prometheus.yml
    cat << 'EOF' > /var/lib/faasd/prometheus.yml

# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'gateway'
    static_configs:
    - targets: ['gateway:8082']
EOF

    rm -rf /var/lib/faasd/resolv.conf
    cat << 'EOF' > /var/lib/faasd/resolv.conf
nameserver 8.8.8.8
EOF

    rm -rf /var/lib/faasd-provider/resolv.conf
    cat << 'EOF' > /var/lib/faasd-provider/resolv.conf
nameserver 8.8.8.8
EOF

    rm -rf /var/lib/faasd/secrets/basic-auth-user
    cat << 'EOF' > /var/lib/faasd/secrets/basic-auth-user
admin
EOF

    rm -rf /var/lib/faasd/secrets/basic-auth-password
    cat << 'EOF' > /var/lib/faasd/secrets/basic-auth-password
PLEASECORRECTME
EOF

    sed -i "s#PLEASECORRECTME#${PASS_INIT}#g" /var/lib/faasd/secrets/basic-auth-password

    rm -rf /etc/systemd/system/faasd-provider.service
    cat << 'EOF' > /etc/systemd/system/faasd-provider.service

[Unit]
Description=faasd-provider

[Service]
MemoryLimit=500M
Environment="secret_mount_path=/var/lib/faasd/secrets"
Environment="basic_auth=true"
ExecStart=/usr/local/bin/faasd provider
Restart=on-failure
RestartSec=10s
WorkingDirectory=/var/lib/faasd-provider
# add
# User=root

[Install]
WantedBy=multi-user.target
EOF

    rm -rf /etc/systemd/system/faasd.service
    cat << 'EOF' > /etc/systemd/system/faasd.service

[Unit]
Description=faasd
After=faasd-provider.service

[Service]
MemoryLimit=500M
ExecStart=/usr/local/bin/faasd up
Restart=on-failure
RestartSec=10s
WorkingDirectory=/var/lib/faasd
# add
# User=root

[Install]
WantedBy=multi-user.target
EOF

    # in saftey,first downloads image and import them offline by advance
    if [ ! -f "/tmp/faasd-containers.tar.gz" ]; then
        wget --no-check-certificate -qO- ${OPENFAAS_PATH}/faasd/0.9.5/faasd-containers.tar.gz > /tmp/faasd-containers.tar.gz && tar -xf /tmp/faasd-containers.tar.gz  -C /tmp
    fi
    ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/basic-auth-plugin-0.18.18.tar
    ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/nats-streaming-0.11.2.tar
    ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/gateway-0.18.18.tar
    ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/queue-worker-0.11.2.tar
    ctr --address=/run/containerd/containerd.sock image import /tmp/faasd-containers/prometheus-v2.14.0.tar

    systemctl daemon-reload && systemctl enable faasd-provider faasd
    systemctl start faasd-provider --no-pager
    systemctl start faasd --no-pager  2>&1)
    status=$?
    updateProgress 65 "$msg" "$status" "faasd install+start"

    # then, wait extra 30 for cni taking effort
    sleep 30

    # finally check to ensure every container services and cni network being ready
    for i in 1 2 3; do [[ ! -z "$(ctr image list|grep basic-auth-plugin)" ]] && break;sleep 3;echo "checking basic-auth ($i),if failed at 3,it may require a reboot"; done
    for i in 1 2 3; do [[ ! -z "$(ctr image list|grep nats)" ]] && break;sleep 3;echo "checking nats ($i),if failed at 3,it may require a reboot"; done
    for i in 1 2 3; do [[ ! -z "$(ctr image list|grep prometheus)" ]] && break;sleep 3;echo "checking prometheus ($i),if failed at 3,it may require a reboot"; done
    for i in 1 2 3; do [[ ! -z "$(ctr image list|grep gateway)" ]] && break;sleep 3;echo "checking gateway ($i),failed at 3,it may require a reboot"; done
    for i in 1 2 3; do [[ ! -z "$(ctr image list|grep queue-worker)" ]] && break;sleep 3;echo "checking queueworker ($i),if failed at 3,it may require a reboot"; done
    for i in 1 2 3 4 5; do [[ ! -z "$(brctl show|grep openfaas0)" ]] && break;sleep 3;echo "checking openfaas0 ($i),if failed at 5,it may require a reboot"; done


    echo "=====================faas-cli install+login progress======================="
    msg=$( #begin

    if [ ! -f "/tmp/faas-cli" ]; then
        wget --no-check-certificate -qO- ${OPENFAAS_PATH}/faas-cli/0.12.9/faas-cli > /tmp/faas-cli
    fi

    cp /tmp/faas-cli ${INSTALL_DIR}/bin/faas-cli && chmod a+x ${INSTALL_DIR}/bin/faas-cli && ln -sf ${INSTALL_DIR}/bin/faas-cli /usr/local/bin/faas && ln -sf ${INSTALL_DIR}/bin/faas-cli /usr/local/bin/faas-cli

    mkdir -p /var/lib/faasd/.docker
    ln -sf ~/.docker/config.json /var/lib/faasd/.docker/config.json

    sleep 5 && journalctl -u faasd --no-pager
    cat /var/lib/faasd/secrets/basic-auth-password | /usr/local/bin/faas-cli login --password-stdin 2>&1)
    status=$?
    updateProgress 70 "$msg" "$status" "faas-cli install+login"
}
```

pai
=======

然后是pai的了,感觉没什么好说，就是把node跟vscode一样处理，将语言做进应用。因为node是带modules发布的。node是一个bootstrap，不像go的containerd，cni这些，是单个exe。我把pai整合了一下：把node-v10.16.2-linux-x64和pm2-3.5.1.tgz,serve-handler,sqlite3-4.1.1.tgz整合了，再把pai-mate-latest.tar.xz和node_modules.tar.xz整合了。最后把二者顶级文件夹整合的结果形成新的pai-mate-latest.tar.xz。这样也方便简化pai的脚本逻辑

```
# install paiserver
installPai() {

    [[ $(systemctl is-active tencentcloud-pai-mate) == "activating" ]] && systemctl stop tencentcloud-pai-mate
    [[ $(systemctl is-active tencentcloud-pai-mate-update) == "activating" ]] && systemctl stop tencentcloud-pai-mate-update
    [[ $(systemctl is-active tencentcloud-pai-agent) == "activating" ]] && systemctl stop tencentcloud-pai-agent
    rm -rf ${INSTALL_DIR}/pai ${INSTALL_DIR}/pai-mate

    echo "=====================paimate install+start progress======================="
    msg=$(
    # prepare directory
    mkdir -p ${INSTALL_DIR}/pai-mate

    # prepare
    mkdir -p ${INSTALL_DIR}/pai
    echo "export PATH=/root/.local/pai-mate/bin:$PATH" > ${INSTALL_DIR}/pai/pai-mate-env
    source ${INSTALL_DIR}/pai/pai-mate-env # get node/npm binary path

    # prepare workspace
    mkdir -p ${DATA_DIR}/pai_mate_workspaces

    # download package
    if [ ! -f "/tmp/pai-mate-latest.tar.xz" ]; then
        wget --no-check-certificate -qO- ${PAI_MATE_SERVER_PATH}/pai-mate-latest.tar.xz > /tmp/pai-mate-latest.tar.xz
    fi

    # unzip
    tar -Jxf /tmp/pai-mate-latest.tar.xz  -C ${INSTALL_DIR}/pai-mate
    mv /tmp/pai-mate-latest.tar.xz /tmp/pai-mate-latest.tar.xz.old

    cd ${INSTALL_DIR}/pai-mate

    # config
    echo "UPDATE_PATH: ${PAI_MATE_SERVER_PATH}" > config.yml
    echo "DOMAIN_NAME: ${DOMAIN_NAME}" >> config.yml
    echo "CERT_PATH: /etc/letsencrypt/live/${DOMAIN_NAME}/fullchain.pem" >> config.yml
    echo "KEY_PATH: /etc/letsencrypt/live/${DOMAIN_NAME}/privkey.pem" >> config.yml

    #npm install --production --unsafe-perm=true --allow-root
    npm run migrate:latest

    # systemd service start
    rm -rf /etc/systemd/system/tencentcloud-pai-mate.service
    cat << 'EOF' > /etc/systemd/system/tencentcloud-pai-mate.service

[Unit]
Description=Tencent Cloud Pai Mate
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
Environment=PATH=/root/.local/pai-mate/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WorkingDirectory=/root/.local/pai-mate
ExecStart=/root/.local/pai-mate/bin/start.sh

[Install]
WantedBy=multi-user.target
EOF

    rm -rf /etc/systemd/system/tencentcloud-pai-mate-update.service
    cat << 'EOF' > /etc/systemd/system/tencentcloud-pai-mate-update.service

[Unit]
Description=Tencent Cloud Pai Mate Update
After=network.target

[Service]
Type=oneshot
User=root
Environment=PATH=/root/.local/pai-mate/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WorkingDirectory=/root/.local/pai-mate
ExecStart=/root/.local/pai-mate/bin/update.sh
EOF

    rm -rf /etc/systemd/system/tencentcloud-pai-mate-update.timer
    cat << 'EOF' > /etc/systemd/system/tencentcloud-pai-mate-update.timer

[Unit]
Description=Tencent Cloud Pai Mate Update

[Timer]
OnCalendar=daily
RandomizedDelaySec=5minutes
Persistent=true

[Install]
WantedBy=timers.target
EOF

    chmod +x ${INSTALL_DIR}/pai-mate/bin/*
    systemctl daemon-reload
    systemctl enable tencentcloud-pai-mate.service
    systemctl start tencentcloud-pai-mate.service
    systemctl start tencentcloud-pai-mate-update.timer 2>&1)

    status=$?
    updateProgress 90 "$msg" "$status" "paimate install+start"


    echo "=====================pai install+start progress======================="


    msg=$(#begin
    mkdir -p ${INSTALL_DIR}/pai/etc
    mkdir -p ${INSTALL_DIR}/pai/bin

    echo "server_path: ${SERVER_PATH}" > ${INSTALL_DIR}/pai/etc/pai.yml
    echo "domain_name: ${DOMAIN_NAME}" >> ${INSTALL_DIR}/pai/etc/pai.yml

    # Note: `agent` binary will update and run this time. `baker` binay will be run next time.
    # cannot overwrite binay, error: text busy
    # mv -f "${INSTALL_DIR}/pai/bin/pai_agent" "${INSTALL_DIR}/pai/bin/pai_agent.old"
    # mv -f "${INSTALL_DIR}/pai/bin/pai_baker" "${INSTALL_DIR}/pai/bin/pai_baker.old"
    wget --no-check-certificate -qO- "${SERVER_PATH}/bin/pai_agent" > "${INSTALL_DIR}/pai/bin/pai_agent"
    # curl "${SERVER_PATH}/bin/pai_baker" -sSf > "${INSTALL_DIR}/pai/bin/pai_baker"
    chmod +x "${INSTALL_DIR}/pai/bin/pai_agent"
    # chmod +x "${INSTALL_DIR}/pai/bin/pai_baker"

    rm -rf /etc/systemd/system/tencentcloud-pai-agent.service
    cat << 'EOF' > /etc/systemd/system/tencentcloud-pai-agent.service

[Unit]
Description=Tencent Cloud Pai Agent
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/root/.local/pai/bin/pai_agent

[Install]
WantedBy=multi-user.target
EOF

    rm -rf /etc/systemd/system/tencentcloud-pai-baker.timer
    cat << 'EOF' > /etc/systemd/system/tencentcloud-pai-baker.timer

[Unit]
Description=Tencent Cloud Pai Baker

[Timer]
OnCalendar=daily
RandomizedDelaySec=5minutes
#OnCalendar=*-*-* *:*:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

    systemctl daemon-reload
    systemctl enable tencentcloud-pai-agent.service
    systemctl start tencentcloud-pai-agent.service 2>&1)
    # systemctl restart tencentcloud-pai-baker.timer 
    status=$?
    updateProgress 100 "$msg" "$status" "pai install+start"


}

if [ $PANEL_TYPE == "0" ]; then
	installOpenfaasd
else
	installPai
fi
```

vscodeonline和结束部分
----

```
# install codeserver
installCodeserver() {

    [[ $(systemctl is-active code-server) == "activating" ]] && systemctl stop code-server
    rm -rf ${INSTALL_DIR}/bin/code-server ${INSTALL_DIR}/lib/code-server-3.5.0 ${CONFIG_DIR}/code-server

    echo "=====================codeserver install+start progress======================="
    msg=$(# begin
    
    mkdir -p ${INSTALL_DIR}/lib/code-server-3.5.0 ${INSTALL_DIR}/bin ${CONFIG_DIR}/code-server

    if [ ! -f "/tmp/code-server-3.5.0-linux-amd64.tar.gz" ]; then
        wget --no-check-certificate -qO- ${CODE_SERVER_PATH}/v3.5.0/code-server-3.5.0-linux-amd64.tar.gz > /tmp/code-server-3.5.0-linux-amd64.tar.gz
    fi

    tar  -xf /tmp/code-server-3.5.0-linux-amd64.tar.gz -C ${INSTALL_DIR}/lib/code-server-3.5.0 --strip-components=1
    ln -s ${INSTALL_DIR}/lib/code-server-3.5.0/bin/code-server /usr/local/bin/code-server

    # systemd service start
    rm -rf ${CONFIG_DIR}/code-server/config.yaml
    cat << 'EOF' > ${CONFIG_DIR}/code-server/config.yaml

bind-addr: 127.0.0.1:5000
auth: password
password: PLEASECORRECTME
cert: false
EOF

    sed -i "s#PLEASECORRECTME#${PASS_INIT}#g" ${CONFIG_DIR}/code-server/config.yaml

    # systemd service start
    rm -rf /etc/systemd/system/code-server.service
    cat << 'EOF' > /etc/systemd/system/code-server.service

[Unit]
Description=code-server
After=network.target

[Service]
Type=exec
ExecStart=/usr/local/bin/code-server
Restart=always
User=root

[Install]
WantedBy=default.target
EOF

    systemctl daemon-reload && systemctl enable code-server
    systemctl start code-server 2>&1)
    status=$?
    updateProgress 100 "$msg" "$status" "code-server install+start"
}

installCodeserver

echo "=====================finished .....====================="
if [ $PANEL_TYPE == "0" ]; then
	echo "the final admin panel url you can access(login with user name admin and passwd given): https://${DOMAIN_NAME}/faasd/,password is '$(cat /var/lib/faasd/secrets/basic-auth-password)',thevscodewebide server at:https://${DOMAIN_NAME}/codeserver/,password is ${PASS_INIT},thank you!!"
else
	echo "the final admin panel url you can access(login with your valid cvm account): https://${DOMAIN_NAME}/pai/,datadir is ${DATA_DIR},thevscodewebide server at:https://${DOMAIN_NAME}/codeserver/,password is ${PASS_INIT},thank you!!"
fi


# count time
endTime=$(date +%s)
echo 'Total Time Spent: '$((endTime-beginTime))'s'
```

-------

如果安装后你更改了密码，可以systemctl restart faasd faasd-provider containerd使之生效。刚刚收到短信20201012,腾讯scf免费要改为一年之后收费了，还是自备云函的好

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108925317/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




