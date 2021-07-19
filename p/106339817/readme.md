v2ray-利用看网站原理模拟链路达成VPN
=====

__本文关键字：v2ray https tls, v2ray caddy__

在我们以前的文章中，我们一直谈到vpn相关课题,如《在阿里云海外windows主机上开启VPN》，《openvpn - the only access server we need,通用网络环境和访问控制虚拟器》等，在谈群晖的公网访问方式的时候我们也谈过到穿透盒子，花生壳蒲公英等，frp,cow等课题，这些与portalbox storage相对的accessbox vpn的课题其实也是属于vpn —— 一花一世界，vpn技术背后是一个代表access server的通用技术范畴，不只是用来翻墙的，它是用来打破现有复杂网络环境的限制，在这上面复合建造一层通用内网环境，来作本不可能的节点相通的节点之间互访或加速的。

以实现互访为基础目的网络活动，在各个层面都可能会达到目的，不必涉及到VPN。拿翻墙来说，远程桌面（remote app, vnc）,流量中转导向（反向代理,反向代理下的内网穿透），建立复合链路，都可以达到效果。不过真正的VPN和VPN下的翻墙，偏向指的是建立复合链路，建立网络上的网络：要从链路层面上开始去打造一个模拟网络环境去实现最终目的。这些在《openvpn - the only access server we need,通用网络环境和访问控制虚拟器》的开头处，我们说的很多。

VPN光提供好的通达性也不行，也要提供绝对的安全。抗封也是安全的一种，这些非安全因素可能来自恶意man in the middle，甚至也可能来自ISP，来自政府，其封VPN的技术手段多样，可能在VPN发生时的各个网络层面以不同技术存在，如dns过滤,封来路，封源头，封IP封端口，流量刺探，特征识别，协议分析。

那么，是否真的存在一种永远抗封、或尽可能99.99%抗封的VPN呢？使访问点之下成为绝对的公网中的暗网。甚至让来自提供网络服务的正当中间层的管理也变得无视。让安全的VPN成为任何云的绝对标配,走进accesspoint server的范畴。—— 我觉得，只要双方的网络没有去掉真正的物理层的互通，即还在同一个internet，就存在绝对的可能，因为抗封和反封本来就是一对此消彼长的矛盾，一种方式就是设置尽可能多的中间层和转发，中间层设置得越多，被破解和被识别的可能性就越低（当然效率就越低）。这些层面中，使用尽可能多的正常层面和服务（ISP不可能或不致于为了防VPN去封掉和干涉的那些业务），把风险高的层面层面降到只有1个甚至放到本地（比如shadowsocks就是这样，它把DNS放到本地不出网），需要越网的部分被封风险低的正常层面放到后面且高强度加密。

好了不重复了。我们接下来的要介绍的v2ray，是ss的增强，自带混淆和协议叠加，有流量中转导向。也可用于翻墙。它甚至可以与nginx或caddy等反向代理+流量中转+静态页面服务一起。产生更多有趣的功能组合。

V2ray：最基本的http混淆路径
-----

v2ray的效果，它有点像利用nginx反向代理做一个谷歌镜像，都是采用了一定的网站原理达到访问被代理网站的手段，不过本质却完全不一样，nginx反代google是ngx转发http的基础上，仅代理一个网站的流量。这种方式有很大局限,所有的行为和流量在ISP眼中都是透明的，使用方式上也是用浏览器访问一个特定网站。而v2ray属于走链路的真正VPN。链接建立后使用方式不限应用类型和方式，是上述所讲协议层的层层叠加，xxx over yyy(yyy作为传输协议)，你可以把它想象成一堆流量处理工具：

以我们要讲的最基本的http混淆路径来说，它先模拟正常的https流量（这对于翻墙来说就是抗封1），再在这种流量上tls加密（对于翻墙这就是抗封2），这二者组成了流量混淆。这是ISP能看到的最终的那部分流量。是越网的那部分流量。也是正常的应用流量。

在远端不需要越网的流量则是正常的caddy，它将VPS上的ws流量作转发到上述的https，在本地不需要越网的流量则是正常的DNS。

风险最大的部分，是客户端连vps时采用的vmess协议。只有一小部分。这被封就没有办法了。所以CDN在这里发挥作用。如果再加以cdn，isp端往往都看不到vps的真实IP。此时cdn成为防封的又一强大叠加层。

如何搭建v2ray：with caddy or with/without cdn
-----

这里讲解下基础的搭建方式，首先把域名A映射到VPS的IP。可以是二级域名。如果要使用CDN，去cloudflare申请个号，dns服务商要设为cloudflare中的相应条目，选择免费服务，在cloudflare中添加网站，即上面的二级域名的主域名，然后打开对应添加的网站的ssl/tls栏，设置为Your SSL/TLS encryption mode is Full（好像这个是自动设置好的？），然后点开网站的DNS页，进去添加A解析该二级域名，对应于这个二级域名的地方先把云朵点灰（不使用proxy，使用dns only）。跑完下面的脚本后再打开，否则脚本get不到ip。

再安装v2ray，source <(curl -sL https://git.io/fNgqx) –zh，这底下是一个multi-v2ray的自动化安装脚本。https://github.com/Jrohy/multi-v2ray,v3.7.3脚本中pip3挂掉需自己装，为什么选择这个呢，因为这是一个中性，不太复杂，也没有简单到什么都没有的脚本。脚本跑完后提示你可以使用v2ray命令。此时，它是按默认配置的—即第一条相对的inbound outbound配置项。

第三安装caddy，wget https://git.io/vra5C -O - -o /dev/null|sudo bash，这也是一个中性复杂度的脚本，然后caddy install安装caddy，中间会提示你启不启用php，按你需要选择。然后叫你填入默认域名-即第一条网站的域名，即前面映射的域名，填入，caddy会自带https，可以自动申请一年的免费证书，这时提示email时提供一个就好了,它就会帮你申请好。caddy start就能启动caddy了。

这二个脚本都基本你不需要再做什么，修改默认配置或添加配置即可，我们下面直接修改默认配置。

v2ray的配置：执行v2ray,5 global setting设为中文，重新执行v2ray,3更改设置，修改email为caddy中申请SSL提供那个，修改port为你喜欢的数定,修改传输方式，选择web socket,输入想伪装的域名，即caddy安装时的默认网站域名，重新执行v2ray，3更改配置，改变TLS，开启，选择生成证书，提示填入域名，这基本是caddy自动化申请ssl的v2ray版本，v2ray和caddy都需要来一次。要填入的域名就是那个映射的二级域名。如果没有事先做好，这里会提示出错，输入的域名与本机IP不符。重新执行v2ray，4查看配置，我们可以有一条ws的/path/。一直都会出现的vmess协议和客户端配置都是当前v2ray配置用于客户端配置的那些当前参数，v2ray与ss一样，都是同一份客服。

现在来配置caddy。cd /etc,打开Caddyfile,vi Caddyfile,fastcgi下添加：

```
proxy /你看到的path/ 127.0.0.1:你选择的端口 {

                websocket

                header_upstream -Origin

        }
```

这样caddy和v2ray就对接起来了。重启生效。v2ray,caddy都会开机自启。

如果使用CDN，此时进CF，把云朵点成彩色proxy模式。

现在打开二级域名。会出现一个caddy的欢迎页(caddy默认占据80)，打开二级域名:你选择的端口,出现Client sent an HTTP request to an HTTPS server.显示这个才对。如果caddy的logs中出现v2ray.com/core/transport/internet/websocket: failed to serve http for WebSocket > accept tcp [::]:端口，就把上面过程重新跑一次。如果测试通过就可以开始下面的配置客户端了。

一个静态网站有很多流量即只有少量IP这个特征较明显，你可以这里设置一个简单的静态网站。以防人工鉴别，你可以放置一个下载超大文件的链接。文件放在这个服务器上。

v2ray连接方法:手动配置客户端
-----

好了，下面我们讲解手动配置客户端参数。之所以选择手动，不使用v2mess和配置文件是为了学习目的。拿osx下的v2rayx来说。

点configure,增加一个服务器，adress填二级域名，port填你选择的端口，userid就是v2ray执行时显示的uuid,可以在v2ray上添加多个uuid，每一项uuid对应一个用户。alterid同理。security:auto, network:ws, transport点开，websocket页中，path填上面的/你选择的path/，headers，填{ "Host" : “你的二级域名“ }，tls页选择use tls,servername填那个二级域名。单击OK完成。

在v2rayx主菜单中选择刚配好的server条目，load core,模式选global模式。你可以稍后用APP代理设置。这里global可以使所有的APP都走v2ray的通道。

移动端可用Xcode连接手机或爱思助手连手机，安装shadowrocket ipa，正式版的App Store shadowrocket是下载不到实体文件的。你可以把ipa想象成开发版的安装包。安卓是v2rayng


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339817/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



