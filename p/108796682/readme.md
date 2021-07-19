在云主机上安装vscodeonline
=====

__本文关键字：teamviewer保持在线的替代品把vscode terminal当虚拟主机管理面板。remote-openfaas,打造vscodeos:用插件打通openfaas界面到vscodeonline，实现ide.sh与云函数容器对接__

在前面《云主机上部署pai》,《云主机上部署openfaas》中，我们用同样风格的脚本写出了在云主机上部署的二个paas面板，pai类似虚拟主机管理器，而openfaas是paas->faas，综合这二者都是部署和devops面板，它们在透出的界面5523,8080处用web操作。然后我们在《戒掉PC，免pc开发，cloud ide and debug设想》又遇到了vscodeonline，这三者都是云主机构造paas APP以“在线开发和部署”的OS扩展，体验良好度又都有前后分离十分接近，因此，我们这次也把vscodeonline集成在这里，组成成为minstackos的主要部分。未来，我们把所有讲到的paas app用这些面板串联起来，使它们成为可一键在线开发和部署的APP，形成minstackos的appstore来源。

> 在《在openfaas面板上安装onemanager》中我们讲过云主机的ssh往往是很容易断的，如果是graphic linux下，需要tv这样的这样的远程桌面方案来保持长时间在线，但实际上，vscodeonline也有linux终端面板。remote-ssh连接的vscodeonline可以保持长时间ws在线而且可以点上面的一个图标扩展到整个IDE的编辑区。因此体验上，后者可成为前者的良好替代。

好了不废话了。下面依然在一台ubuntu 1h2g上进行。


基础
-----

一些变量

```
MIRROR_PATH="http://default-8g95m46n2bd18f80.service.tcloudbase.com/d/demos"
# the code-server web ide
CODE_SERVER_PATH=${MIRROR_PATH}/codeserver
```

安装codeserver
-----

脚本被做成融合成安装pai和openfaas的风格，按standalone方式安装,以root身份运行。你可以集成自己需要的语言和插件服务到这个IDE，以做到尽量开箱即用。

```
# install codeserver
installCodeserver() {

    echo "=====================codeserver install progress======================="
    msg=$(mkdir -p ~/.local/lib/code-server-3.5.0
    wget --no-check-certificate -qO- ${CODE_SERVER_PATH}/v3.5.0/code-server-3.5.0-linux-amd64.tar.gz > /tmp/code-server-3.5.0-linux-amd64.tar.gz && tar  -xvf /tmp/code-server-3.5.0-linux-amd64.tar.gz -C ~/.local/lib/code-server-3.5.0 --strip-components=1
    rm -rf /tmp/code-server-3.5.0-linux-amd64.tar.gz
    ln -s ~/.local/lib/code-server-3.5.0/bin/code-server ~/.local/bin/code-server
    PATH="~/.local/bin:$PATH"

   # systemd service start
    rm -rf ~/.config/code-server/config.yaml
    cat << 'EOF' > ~/.config/code-server/config.yaml

bind-addr: 0.0.0.0:5000
auth: password
password: pleasecorrectme
cert: false
EOF

   # systemd service start
    rm -rf /etc/systemd/system/code-server.service
    cat << 'EOF' > /etc/systemd/system/code-server.service

[Unit]
Description=code-server
After=network.target

[Service]
Type=exec
ExecStart=~/.local/bin/code-server
Restart=always
User=root

[Install]
WantedBy=default.target
EOF

    systemctl daemon-reload && systemctl enable code-server
    systemctl start code-server 2>&1)
    status=$?
    updateProgress 95 "$msg" "$status" "code-server install"
}
```

安装完成后记得修改~/.config/code-server/config.yaml下的密码，端口为5000。如果你要用上证书，就最好搭配脚本中的nginx+certbot申请的那个。cert: false也可以用假的localhost的那个，但是基本没有什么用。

-----

其实，利用那个remote-container，可以把openfaas-cli跟vscode连起来，利用工程源文件下的yml模板（.pai.yml,openfaas-cli.yml,etc...）文件打造一个带开发部署的工程资源组织文件，形成remote-openfaas效果：多环境多语言下，需要频繁切换环境，一次开发总是跟一次塔环境开始的，这也是vagrant和docker对于开发的意义（以前是vm，没有模板机制），而docker用于开发也用于部署。以后一套APP天然就有一个online webide守护，自带开发环境了。



-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108796682/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


