在云主机上手动安装腾讯PAI面板
=====

__本文关键字：云主机上装管理面板__

在前面，我们介绍过lnmp,sandstorm paas，还有黑群晖，docker管理面板，这些都是云OS上的面板扩展和APPSTACK扩展，分散在不同级别被实现，（像群晖这种是OS和面板一体的），包括这里要介绍的pai和未来可能要介绍的openfaas云函数面板，基本可以分为二类，一带无devops无隔离，没有明显的虚拟化APP打包特征，像宝塔，lnmp，pai，APP直接在baremetal上运行，一带devops，基于容器。像docker管理面板，openfaas。都可以做所有通用服务应用不限web，里面基于容器的技术也都类同，不过vs sandstorm paas，openfaas是用标准容器方法达成的(vs真正的用统一语言和统一微内核做的那套，容器是我们当代伪applvl的virtualappliance做得最完善的了。)，更开放更devops。

所以谁说openfaas,sandstorm,lamp,dsm这样的规模的东西不是个os?openfaas cloud还有整合存储Minio，类似自建云函数+云存储的方案只不过个人难于负担存储部分只能寻求OD这样的替代。

好了不废话。

上次我们搞好了云主机装机的pebuilder.sh。这次来介绍云主机装机常用的服务器套件，一般这类产品有宝塔，wdcp,lnmp等，但是鉴于我们近期在研究云函数和serverless，这次我们找到了PAI，https://cloud.tencent.com/solution/pai，它在一台云主机上自动绑定一个cloudbase域名，并做了对小程序的自动鉴权（大约小程序对xxx.pai.cloudbase.com的域名自动鉴权，否则需要去小程序后台填自定义域名），集成了git拉取pai项目，自动certbot作ssl验证，当然，tx的servless产品主要有cloudbase（里面有云函数云存储云数据库）和wx ide。这个PAI并不能达到官方cloudbase提供的服务那么完整(自建云函数机制，支持云函数的event,context写法)，也不能做到让wx ide完全无缝对接（比如管理PAI上的云函数），这货吧有点像nodejs做的容器和devops，目前它只是自动鉴权方面有点强而已。其它只是一个通用服务器和不使用云函等的小程序后端，没发现什么亮点。

这个PAI它不是一个镜像也不是一个软件，而是需要购买时绑定的。下面我们把它安装在任意云主机上，甚至不是tx cvm也可以。这样我们就失去了那个免费xxx.pai.cloudbase.com三级域名和自动鉴权的好处，但是实际上用自己的域名和自动鉴权也不费事。关键是我们想看看pai有哪些程序可用。直接给脚本：

基础
-----

注意使用说明：云主机事先开5523，并域名绑好到这个云主机上。以便程序内自动申请证书等工作。

一些变量：

```
MIRROR_PATH="http://default-8g95m46n2bd18f80.service.tcloudbase.com/d/demos"
# the pai backend
SERVER_PATH=${MIRROR_PATH}/pai/agent/stable/pai_agent_framework
PAI_MATE_SERVER_ROOT_PATH=${MIRROR_PATH}/pai/mate
PAI_MATE_SERVER_PATH=${MIRROR_PATH}/pai/mate/stable/install
TOOLS_PATH=${MIRROR_PATH}/pai/tools
```

安装依赖

apt-get install git nginx gcc python3.6 python3-pip python3-virtualenv python-certbot-nginx golang -y

单独安装node语言件：

```
# install node.js
installNodejs() {

    echo "=====================node.js progress======================="
    msg=$(wget -q ${TOOLS_PATH}/node-v10.16.2-linux-x64.tar.xz
    tar -Jxvf node-v10.16.2-linux-x64.tar.xz -C /usr/local/
    ln -sf /usr/local/node-v10.16.2-linux-x64 /usr/local/node
    rm node-v10.16.2-linux-x64.tar.xz -f
    # for manual launch node in shell maybe in the later
    echo "export PATH=/usr/local/node/bin:$PATH" >> ${HOME}/.bashrc

    wget -q ${TOOLS_PATH}/pm2-3.5.1.tgz
    PATH=/usr/local/node/bin:$PATH npm install -g pm2-3.5.1.tgz
    PATH=/usr/local/node/bin:$PATH npm install -g serve-handler
    rm pm2-3.5.1.tgz -f

    wget -q ${TOOLS_PATH}/sqlite3-4.1.1.tgz
    PATH=/usr/local/node/bin:$PATH npm config set user 0
    PATH=/usr/local/node/bin:$PATH npm config set unsafe-perm true
    PATH=/usr/local/node/bin:$PATH npm install -g sqlite3-4.1.1.tgz
    rm sqlite3-4.1.1.tgz -f 2>&1)
    status=$?
    updateProgress 10 "$msg" "$status" "node.js"
}

installNodejs
```

pai前后端基础支持
-----

后端5523会透出管理页面，/data/pai-mate-workspace中的应用代理到nginx 3000，first time renew也是为了生成一个/etc/letsencrypt/renewal/下的模板文件供certbot-renew.service服务使用。
安装中，请保证certbot renew务必成功。否则后面的二个pai服务绝对启动不了。但如果成功，基本安装就能很好完成。

```
confignginx() {

    echo "=====================certbot renew progress======================="
    systemctl enable nginx.service
    systemctl start nginx

    cp -f /lib/systemd/system/certbot.service /etc/systemd/system/certbot-renew.service
    cp -f /lib/systemd/system/certbot.timer /etc/systemd/system/certbot-renew.timer

    # sed -i "s/renew/renew --nginx/g" /etc/systemd/system/certbot-renew.service

    msg=$(
    #first time renew
    certbot certonly --standalone --agree-tos --non-interactive -m ${EMAIL_NAME} -d ${DOMAIN_NAME} --pre-hook "systemctl stop nginx"

    systemctl daemon-reload 
    systemctl enable certbot-renew.service
    systemctl start certbot-renew.service
    systemctl start certbot-renrew.timer 2>&1)
    status=$?
    updateProgress 40 "$msg" "$status" "certbot renew"


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
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_pass http://localhost:3000;
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

    # restart nginx
    msg=$(systemctl reload nginx.service 
    systemctl restart nginx 2>&1)
    status=$?
    updateProgress 50 "$msg" "$status" "nginx reconfig"

}

confignginx
```

安装pai,paimate

```
installPai() {

    echo "=====================paimate install progress======================="
    mkdir -p ${HOME}/pai
    echo "export PATH=/usr/local/node/bin:$PATH" > ${HOME}/pai/pai-mate-env
    
    rm -rf /data/logs
    sudo mkdir /data/logs

    echo "Start installing PAI Mate!"
    echo ${PAI_MATE_SERVER_PATH}
    echo ${DOMAIN_NAME}

    INSTALL_DIR="${HOME}/pai-mate"

    # prepare directory
    mkdir -p ${INSTALL_DIR}

    msg=$(# download package
    wget -qO- ${PAI_MATE_SERVER_PATH}/pai-mate-latest.tar.xz > ${INSTALL_DIR}/pai-mate-latest.tar.xz

    # unzip
    tar -Jxvf ${INSTALL_DIR}/pai-mate-latest.tar.xz  -C ${INSTALL_DIR}
    mv ${INSTALL_DIR}/pai-mate-latest.tar.xz ${INSTALL_DIR}/pai-mate-latest.tar.xz.old

    cd ${INSTALL_DIR}

    # config
    echo "UPDATE_PATH: ${PAI_MATE_SERVER_PATH}" > config.yml
    echo "DOMAIN_NAME: ${DOMAIN_NAME}" >> config.yml
    echo "CERT_PATH: /etc/letsencrypt/live/${DOMAIN_NAME}/fullchain.pem" >> config.yml
    echo "KEY_PATH: /etc/letsencrypt/live/${DOMAIN_NAME}/privkey.pem" >> config.yml

    # prepare
    source ${HOME}/pai/pai-mate-env # get node/npm binary path
    #npm install --production --unsafe-perm=true --allow-root
    # download from cos
    wget -qO- ${PAI_MATE_SERVER_ROOT_PATH}/libs/node_modules.tar.xz | tar -Jxf -
    npm run migrate:latest

    # prepare workspace
    mkdir -p /data/pai_mate_workspaces

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
Environment=PATH=/usr/local/node/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WorkingDirectory=/root/pai-mate
ExecStart=/root/pai-mate/bin/start.sh

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
Environment=PATH=/usr/local/node/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WorkingDirectory=/root/pai-mate
ExecStart=/root/pai-mate/bin/update.sh
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

    chmod +x ${INSTALL_DIR}/bin/*
    systemctl daemon-reload
    systemctl enable tencentcloud-pai-mate.service
    systemctl start tencentcloud-pai-mate.service
    systemctl start tencentcloud-pai-mate-update.timer 2>&1)

    status=$?
    updateProgress 90 "$msg" "$status" "paimate install"


    echo "=====================pai install progress======================="
    CONFIG_INSTALL_DIR=${HOME}/pai/etc
    BINARY_INSTALL_DIR=${HOME}/pai/bin

    mkdir -p ${CONFIG_INSTALL_DIR}
    mkdir -p ${BINARY_INSTALL_DIR}

    echo "server_path: ${SERVER_PATH}" > ${CONFIG_INSTALL_DIR}/pai.yml
    echo "domain_name: ${DOMAIN_NAME}" >> ${CONFIG_INSTALL_DIR}/pai.yml

    msg=$(# Note: `agent` binary will update and run this time. `baker` binay will be run next time.
    # cannot overwrite binay, error: text busy
    # mv -f "${BINARY_INSTALL_DIR}/pai_agent" "${BINARY_INSTALL_DIR}/pai_agent.old"
    # mv -f "${BINARY_INSTALL_DIR}/pai_baker" "${BINARY_INSTALL_DIR}/pai_baker.old"
    wget -q "${SERVER_PATH}/bin/pai_agent" > "${BINARY_INSTALL_DIR}/pai_agent"
    # curl "${SERVER_PATH}/bin/pai_baker" -sSf > "${BINARY_INSTALL_DIR}/pai_baker"
    chmod +x "${BINARY_INSTALL_DIR}/pai_agent"
    # chmod +x "${BINARY_INSTALL_DIR}/pai_baker"

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
ExecStart=/root/pai/bin/pai_agent

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
    updateProgress 100 "$msg" "$status" "pai install"


}

installPai
```
安装完成后，打开域名:5523，用你的云主机帐号，最好root登录。其它就没有什么了，/root/pai,/root/pai-mate是程序目录 /data是数据，，测试了下，只有一个当前应用能起作用（鸡肋？）。，，并没有太深入去了解这个工程的细节。只是追求能做到可用即可。恩恩

---------

我们的下一文，打造yet another cloudbase:在云主机上安装cloudide(jupyter)为pai面板所用

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108544605/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



