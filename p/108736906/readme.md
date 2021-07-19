在openfaas面板上安装onemanager（2）
======

__本文关键字：openfaas onemanager,nginx仅监听ipv4，disabling IPv6 name/address support: Address family not supported by protocol__

在前面《在openfaas面板上安装onemanager》中我们讲到自建云函数将om放到自己服务器的方法，但是遗留了一些问题，1，经常会有“error finding function onemanagerforopenfaas.: Get http://faasd-provider:8081/system/function/onemanagerforopenfaas?namespace=: dial tcp: i/o timeout”，2，整个应用访问延时感比较明显（这些在腾讯scf也有,容器是多层虚拟化有抽象成本），但在faasd中，还有其它一些导致延时的问题。3，程序中有一些路径不对，readme.md可以显示，但压缩文件下载不了显示页面返回空白，4，/function/xxx/的形式也长。

第一个问题，我们得去了解faasd的机制，在安装步骤的cd faasd src,git checkout 0.9.2,faasd install时，它利用一个faasd bin启动了二个过程，faasd up和faasd provider，ps aux|grep faasd可看到二进程的路径，netstat -tlp | grep -i faasd或lsof -i:8080可看到端口（注意到它们都绑定在了ipv6），这里的逻辑如下：

> faasd deploys OpenFaaS along with its core logic faasd-provider, which uses containerd to start, stop, and manage functions.The faasd process will proxy any incoming requests on HTTP port 8080 to the OpenFaaS gateway container。

> 解析一下：云函数首先是个容器集群，faasd维护一个单机容器集群和一个内部网络(这种东西在《利用colinux制作tinycolinx，在ecs上打造server farm和vps iaas环境代替docker》一文中我们提到单机构造集群的例子，多个vm要重用80，再想象一下，虚拟机用得多吧，也要接确到与宿主的多种网络出入逻辑和子网逻辑。在《openaccess》一文中也讲到过,openaccess虚拟局域网，这些都是虚拟网络的例子。只不过这次是管理单机上（或集群）上的容器集。）多个容器pod也要network，这个用linux上的iptables转发就可以办到，但容器为了功能强大往大了做，它使用了cni。体现在openfaas端（/etc/cni/net.d），就是它在本地网卡和虚拟的cni openfaas网卡之间转发实体间占用虚拟网卡的IP空间，faasd自己维护一个这样带自我解析的机制从而做到不依赖于dns，它们都cat /var/lib/faasd/hosts和cat /var/lib/faasd/docker-composer.yml中可以看到定义。

> 我们发现，到最终应用，整个serverless机制，它就是faasd+cni+一堆容器堆出来的，gateway是容器，在10.62.0.1网关(这个由cni做出来的openfaas0网卡)，watchdog也是容器。app也是容器。只是hosts中没有app的10.62.0.xx的ip地址。ps aux | grep onemanager可以看到它是一个containerd-shim-runc-v2容器进程，每次重启后这个docker本身有一个ip 10.62的ip( journalctl -u faasd --lines 40，出现onemanagerforopenfaas has IP: 10.62.0.xxx获得)，每次deploy后也会换，你会发现每打开一个链接，lsof -i:8080都会增加几个进程，这就是openfaas的云函数机制，它是按需为访问活动中的函数（从后端容器进程中提取,我们知道，容器运行时即容器的实现方法，现在有很多种，不光docker。containerd也是）生成进程的，等到这些访问活动没了，它就自动消失，没有一个像httpd daemon的守护（这是延时的由来原因之一），faasd负责管理所有这一切。

无论如何，作为serverless，faasd这样做的好处是：（这样做的意义，在于最大利用单机上的资源，而且以可伸缩的方式，所以同样可以有集群方案），且你可以用为语言贡献库的方式去开发远程服务性APP。因为它们是直接php index.php或go index.go弄起来的 Any binary can become a function through the use of watchdog.，开发接上了部署都在API级（带容器集群开发/部署级别的faas给devops带来的功效相当于另一种带visual ide as devops的集成appstack环境）。而且这种非http方式提高了利用率。不过单机上的应用集群局限于单机的最大能力，而集群上的应用则不会。

解决问题1，2
-----

来看第一个问题，我们猜它是发现不了docker-composer.yml中http://faasd-provider:8081中的faasd-provider对应的ip，于是我们把yml中的地址替换成10.62.01。问题1，解决。但随后经常出现“service not reachable for onemanagerforopenfaas”，（但这个其实访问很流畅：http://m.shalol.com:8081/system/function/onemanagerforopenfaas?namespace=-），于是我们查看日志：sudo journalctl -u faasd-provider --lines 40，发现error with proxy request to: http://10.62.0.xxx:8080/，这就是containerd的原始进程经过gateway代理的过程中出现的。我们要把它弄成ipv4的listen,如何在ipv4上启动faasd试试：

在ubuntu上关掉ipv6，我尝试过sysctl关掉所有网卡，禁掉/etc/hosts所有[::]:等方案都不行，最后etc/default/grub中修改GRUB_CMDLINE_LINUX_DEFAULT="quiet spalsh"为GRUB_CMDLINE_LINUX_DEFAULT="ipv6.disable=1 quiet splash"然后update-grub重启成功。再次netstat发现faasd进程都被forced to use ipv4了(不过听说faasd升到0.9.5，源码中有ip修改的地方，如果不是默认的空或者0或者0.0.0.0，那么都可以成功在IPv4下建立侦听。)。问题2延时减少了很多，（无论是直接访问原始容器还是代理到gateway的8080）“service not reachable for onemanagerforopenfaas”出现的机率都减少了很多(进入系统多等，或要访问应用至少一次，faasd-provider才会启动,这是40秒docker自定义网络延时？)。

但是nginx却启动不了了，考虑到禁用了ipv6,于是我把/etc/nginx/conf.d/default.conf中listen [::]:都注释掉了，但还是启动不了（网上说的那些ipv6 on的方法也不管用），我的是ubuntu18.04上的nginx 10.14.2，这是因为跟faasd一样，Nginx is doing DNS resolution at startup by default，在ubuntu上，With current default vhost listening on IPV6, nginx won't start on fully IPV6 disabled system, expecting to connect to a IPV6 socket on port 80.但是在其它发行版Other distributions does that (not including ipv6 listen by default) already (e.g. Centos and Upstream)，不过This bug was fixed in the package nginx - 1.17.5-0ubuntu1，于是

```
sudo touch /etc/apt/sources.list.d/nginx.list
deb [arch=amd64] http://nginx.org/packages/mainline/ubuntu/ bionic nginx
deb-src http://nginx.org/packages/mainline/ubuntu/ bionic nginx
wget http://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key
sudo apt update
sudo apt remove nginx nginx-common nginx-full nginx-core
如果apt包管理器询问你是否要安装新版本的/etc/nginx/nginx.conf文件，则可以回答否，以保留原来的注释掉了[::]:的config.
```

成功。

关于延时，watch-dog有一个特化版，可以将一般进程转为长驻留的http，这种方式下Keep function process warm for lower latency / caching / persistent connections through using HTTP，延时不会因为频繁经过网关开进程导致开销(否则你需要定时访问触发以触发网关warm up这个function)，而且它支持webframework，因为里面可以内嵌一个webserver(虚拟机管理面板中，为什么php cgi很容易跟nginx组合，配置成熟，很多细节可以配置，而python项目管理就相对少一些。因为php cgi自带了服务进程管理相当于这里的watchdog作为外部，框架放在内部源码部分仅涉及到业务逻辑不涉及到服务管理，内外分开，而python这样的没有cgi这样的东西，框架管理进程，直接在里面开服务。这种细节在外部与nginx组合的时候显得难于配置)，webframework可以允许你定制一次请求响应和url router，允许你用类似js的event,content方式开发。使用这种watch-dog，需要：

```
FROM openfaas/of-watchdog:0.7.7 as watchdog
最下面ENV fprocess="php index.php"之后加：
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000" (框架服务器透出的转发地址，转到watchdog:8080)
```
由于上述方案适合这种框架中自带服务器的，onemanager并不适用，所以我没尝试。它还支持一种mode=static，想到pai中那个static:public了吗？在支持statcisite方面，上次说到pai需要手动一次，因为它使用的是devops的一个部件git，而没有使用到devops的主要部件docker。这里用了云函数这种方案，就纯自动了。


解决问题3
-----

你会发现地址不对了。这时在common.php中作以下替换：
$_SERVER['base_disk_path'] . '/' . $path
$_SERVER['base_disk_path'] . '/' . 'function/onemanagerforopenfaas/' . $path
有大约七处。

对于那个文件下载不了了的问题显示空白，curl_setopt是common.php中onemanager向上面的api要求结果的函数。影响着向下面的客户返回的结果。你可以在本地模拟curl -I 正常的om的下载返回结果，修改common.php以让整个程序也返回同样的结果。

另外发现一个可在common.php中markdown替换网址，以实现在域名子目录里放md网站效果的功能，(如果不这样，则你需要在md中定制带此目录的域名根路径)。这个其实跟domainforproxy一样可以做成后台配置项。

```
因为我使用了parsedown，你也可以在原始的$headmd输入中替换：
            $headmd = str_replace('<!--HeadmdContent-->', $parser->text(str_replace('/p/','/minlearnprogramming/p/',fetch_files(spurlencode(path_format(urldecode($path) . '/readme.md'),'/'))['content']['body'])), $tmp[0]);
```

---------------

被一个腾讯cvm的问题折腾了好久，别人能正常ssh我的服务器，我在本机却不能(没有任何响应)。web ssh下，别人和我都不能wget，提示no response。解决办法是去管理台切换一下172.16.0.xxx形式的私网地址。


-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108736906/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



