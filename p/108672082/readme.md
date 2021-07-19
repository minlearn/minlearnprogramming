在openfaas面板上安装onemanager
=====

__本文关键字：openfaas onemanager__

在前面《在云主机上手动安装腾讯PAI面板》《利用openfaas faasd在你的云主机上部署function serverless面板》中，我们介绍了二种虚拟主机/容器后端，它们都可以搭配nginx作为前端，形成webstack，和通用服务器应用栈，而且都支持多语言，支持devops部署。其中，前者用git作devops，后者用容器（faasd中用containerd），后者的devops更全面：要知道，现在的devops构建基本是用docker完成的，无容器不devops。况且，openfaas是对我《最小编程学习集与开发栈选型：1，语言选型与开发融合，2，云系统，云服务器，虚拟boot，语言运行时和应用容器，及平台融合，3，云APPSTACK云调试云DEVOPS及开发融合》中的2，3的整个暂代实现（替代目前还无法做到的统一语言，统一内核）,它与我的cloverboot+黑群晖黑苹果等cloud os，组成了一个暂代可用的minstackos:一个企图将包括上面1，2，3的最小开发栈实现做入boot的方案。

其实老早就接确过docker和k8s,swarm这种容器管理面板，但是一直没怎么深入使用(有这功夫为何不研究coreos?)，一是对docker aufs不放心(二是docker本身虽然利用了内核功能，但是本体属于在应用级用虚拟化重新构建网络和文件系统，带来较高负担和延时)，二是swarm,k8s这种比较重，直到接确云函数，再到这里的轻量级faasd，才发现基于容器的面板也可以很轻量（当然，pure k8s和云函化的容器面板还是有点不一样的）。二是云函数本身就关注用它运行功能，而不用涉及到存放数据，因此docker aufs这种可能导致文件系统污染的缺陷也就不足于当回事了。最后这种容器用于生产环境的缺陷慢慢也正在基本被彻底修复。

> faasd用的是containerd，但使用docker构建应用，注意这里的docker-ce只是离线构建openfaas/faasd用的docker镜像之用，并不会加大faasd的运行时，这里提到openfaasd用docker image as app meta(dockfile vs pai.yml)，没错，openfaas用docker作devops。所以可以封装构建流程。而不仅仅像pai面板那样停留于用git拉取。

下面，接上文《利用openfaas faasd在你的云主机上部署function serverless面板》，我们将onemanager移过来。----- 为什么不是fodi,fodi毕竟没有url,不利于运营。

好了，不费话。


调试与基础
----

调试，前调试后调试，都是运维开发的必要过程，甚至是主要过程。因为它是搞清问题的必要实验过程。

在faasd所在主机上apt-get install docker-ce，我在faasd所在主机是一台装上了deepin20a的云主机，因为这样结合teamviewer可以远程桌面(linux上远程桌面大部分都是位图截取算法的不要用。tv效率高，支持本地远程剪贴板交互，而且tv还有一个图形scp功能。deepin+1h2g开2g swap完全可以驾驭，开机只占1G内存)可以长时间保持在线，而不必总是用本地的ssh(十分易断)，

还有，faasd构建应用涉及到大量与docker.io的交互，这个在国内云主机网络环境下速度是很弱的，虽然有一些方案支持自建docker registry给faasd用，但是太折腾，不如在云主机上装v工具，然后命令行下export http/https_proxy。而且，dokcer-cli是支持应用级proxy的，

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo touch /etc/systemd/system/docker.service.d/http-proxy.conf
编辑http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8888/"
然后重启服务：
sudo systemctl daemon-reload
sudo systemctl restart docker
测试是否设置成功：
systemctl show --property=Environment docker
```

然后是登录docker login (registry name，对于官方dockerhub这里可省)，按提示输入用户名和密码。

当然市面上也有很多基于docker的免费或商用docker devops，如果上一步你选择了阿里开发中心的这些私有库，需要修改~/.docker/config.json到 /var/lib/faasd/.docker/config.json供接下来使用。

解决这二个问题后。基本可以往下了：mkdir  test && cd test && faas-cli new onemanagerforopenfaas --lang dockerfile

vi ./onemanagerforopenfaas.yml 在image路径前加上你的dockerhub用户名，或者把它改成完整地址（带registry。比如国内的阿里云docker仓库,记住版本最好要写上，不能用latest???）注意官方dockerpush支持private镜像，但是pull时会提示denied，所以不要把这个镜像设为private。(如果你使用官方dockerhub，且是private仓库，那么也需要~/.docker/config.json到 /var/lib/faasd/.docker/config.json，否则接下来通过faas-cli调用docker pull的时候会提示认证失败，push没问题，默认gateway127.0.0.1:8080需要一次faas-cli login --password xxx,自定义gateway的方法：dockerfile中gateway改且faas-cli login --password xxx --gateway配合改)

上面我们不用带语言的--lang php7（它是php72），是因为：虽然--lang dockerfile和--lang php7都会拉取一大堆template，但后者文件夹混乱。涉及到的修改需要在templates里进行，而前者涉及到的所有改动仅发生在一个文件夹，即生成的onemanagerforopenfaas这里。我们只需从template里复制一些关于php72的脚手架文件即可(index.php和functions文件夹)。不必修改template(也不要删掉，如果调用faas-cli xx -f ./xx.yml，它会再次自动下载)。

但我们这里不复制index.php和functions，把onemanager复制到生成的onemanagerforopenfaas根目录下，我这样做是为了不涉及太多的文件夹结构，也方便接下来的onemanager source（我使用的om版本依然是20200607-1856.19），因为它本身并没有使用类似如下的文件夹结构：

然后，对应修改onemanagerforopenfaas下的dockerfile（如果使用--lang php7要到template/php7下修改，十分混乱）：

```
# Import function
WORKDIR /home/app
COPY ./ ./
WORKDIR /home/app
```

dockerfile中有一处composer install由于被q也很费时。可修改dockerfile，禁掉composer install行，因为onemanager在vender中自带依赖，如果不禁掉，我们也可以在composer.json中自己手工添加下面chunks以提速(其实你也可以删掉composer.json，因为我们并没有使用到它)：

```
"repositories": {
    "packagist": {
        "type": "composer",
        "url": "https://packagist.phpcomposer.com"
    }
}
```

这样调试环境就建好了。调试过程就是修改源码（当然放在第二节中讲，这里是最主要的调试工作,这里仅关注流程），然后sudo faas-cli build  -f ./onemanagerforopenfaas.yml &&  sudo faas-cli push -f ./onemanagerforopenfaas.yml && sudo faas-cli deploy -f ./onemanagerforopenfaas.yml。

下面仅给出结果，不给出具体的调试过程，但是要知道下面得出的结果是这里的不断调试过程得到的,所以这里的流程必不可少。输出信息你可以在源码index.php中写echo，可以观察8080 invoke按钮出来的信息，或网页直接展示。也可以journalctl查看。


源码修改：
-----

faas的那个web环境类似虚拟主机空间，所以主文件index.php中(这也是程序的入口)，我们去掉for tencent的main_handler和for aliyun的handler，直接：

```
<?php

error_reporting(E_ALL & ~E_NOTICE);

include 'vendor/autoload.php';
include 'conststr.php';
include 'common.php';
include 'parsedown.php';


include 'platform/Normal.php';
$path = getpath();
$_GET = getGET();

$re = main($path);
$sendHeaders = array();
foreach ($re['headers'] as $headerName => $headerVal) {
        header($headerName . ': ' . $headerVal, true);
}
http_response_code($re['statusCode']);
echo $re['body'];
```

大部分需要调试的地方跟《利用onemanager配合公有云做站和nas（2）:在tcb上装om并使它变身实用做站版》一文一样，属于环境参数和path获取不到。

platform/normal.php中：

```
function getpath()
{
    $_SERVER['firstacceptlanguage'] = strtolower(splitfirst(splitfirst($_SERVER['HTTP_ACCEPT_LANGUAGE'],';')[0],',')[0]);
    if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_X_FORWARDED_FOR'];
    $_SERVER['base_path'] = path_format(substr($_SERVER['SCRIPT_NAME'], 0, -10) . '/');
    $p = strpos($_SERVER["Http_Path"],'?');
    if ($p>0) $path = substr($_SERVER["Http_Path"], 0, $p);
    else $path = $_SERVER["Http_Path"];
    $path = path_format( substr($path, strlen($_SERVER['base_path'])) );
    return substr($path, 1);
    //    return spurlencode($path, '/');
             
}
```
以上注意到，faas使用的php-cli index.php唤起的服务。php72的$_SERVER数组中，并没有request_uri，但是完全可以用base_path代替。否则index.php中获取到的$path永远为空。这是由不同的web环境导致的问题。

另外一个需要修改的地方：

```
function getConfig($str, $disktag = '')
{
    global $InnerEnv;
    global $Base64Env;
    //include 'config.php';
    //$s = file_get_contents('config.php');
    $configs = ''
    ......
}
```

以上，我们把原本存在于config.php中的'{}'放到上面代替''，因为正常逻辑下程序不能获取到config.php，我们需要手动用《利用onemanager配合公有云做站和nas（2）:在tcb上装om并使它变身实用做站版》获到的envVariables并把参数给它。这样getConfig('timezone')这样的调用才能发挥作用。具体为什么获取不到config.php。。我也没有深入调试。应该也是openfaas所属特殊web环境导致的问题。

你可能还需要禁掉getGET()中的这段:

```
if (!ConfigWriteable()) {
            $html .= getconstStr('MakesuerWriteable');
            $title = 'Error';
            return message($html, $title, 201);
        }
```

还有common.php中一些涉及到html输出的也给禁掉，以适应特殊的openfaas web环境。


-------

突然想把git和docker弄成netdisk backend的静态仓库，不用做到git pull能git clone就好，因为它们的原理上，也是用curl来下载的。

发现一个v使用终端代理命令只在当前窗口生效的问题,如果执行./xx.sh是不能生效的此时开全局也无用，方法是要把该命令也写一次到./xx.sh中。

还有panel.sh缺少多租户环境的功能，对于openfaas是linkerd，我们知道，pai这种也是不支持多租户的，一次只能一个应用。多租对于个人用户也是有用的，那就是可以为每一个函数指定一个子域名并代理出去，目前openfaas使用/function/应用名的形式，可以寻求直接在主域上展示的方法。

还有那个顽固的dail tcp failed faasd-provider问题，日后也需要寻求解决

-------


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108672082/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


