利用fodi给onemanager前后端分离(1)：将fodi py后端安装在腾讯免费cloudbase
=====

__本文关键字：将fodi py后端安装在腾讯cloudbase__

在前面《利用onemanager配合公有云做站和nas》中，我们提到onemanager是个可以装在腾讯云函数API网关或cloudbase上作为云函数的产品，onemanager php backend它本身也是调用onedrive的api，在前面《owncloud微博式记事本》《wp2oc fileshare》中我们也提到这种web api的修改定制。最近的《打造小程序版本公号和自托管的公号:miniblog》也是这种例子,后端cloudfuntion化，前端miniprogram则被部署到微信，，前后端分离和web api化调用是早已流行的趋势。

在《利用onemanager配合公有云做站和nas》中我们也提到要继续为onemanager找提高浏览体验用的缓存方案静态化方案，onemanager拉取文件列表延迟高体验不佳（这种延迟一部分就是它本身前后端不分离，一次返回结果混合了渲染html的客端逻辑，客户端也没有 ajax,另一部分是onemanager这类程序本身也是使用api的，返回结果来自OD本身是外部的，其只有有限的doctrine缓存技术，没有像混合云那样的多样化混合本地和远程云结果构建高效网站体验的能力），这次我们找到了fodi。其前后分离和客户端ajax技术，体验比单纯的onemanger要好得多，fodi分离的好处是，相当大部分逻辑都被放在了客户端。如浏览文件，服务器仅负责一次返回一些json数据，现在的浏览器本身一般也有本地缓存存储。，整个可以带来类似pwa,js based react native app的效果，这也是fodi名字中的fast的理念技术基础。

我们的目标是使od像fodi那样前后端分离，并研究尝试让它接上fodi前端的方法。----- 等等，直接用fodi前后端一套流不香吗？fodi有二个后端，一个backend_cf for cloudflare,一个py for 云函数，fodi py后端输出的是加密的json对我们接下来的研究不够直观化（除非chrome f12），而nodejs端直接输出直观的json结果，但nodejs runtime(<=10.15)又不支持addeventlistener()，它是cloudflare的webwoker api用的(这是一种客户端API，然而CF把它ported到了服务端)，所以我们打算用改造onemanager的结果来适配fodi前端。使之成为fodi的php backend。

注意：Fodi有部分预拉取列表功能，所以产生的云函数调用会比较多（所以推荐将fodi装到vps）,但是om的缺点是云函数调用流量方面也相对较多。

这里还是介绍将fodi后端部署到cloudbase的方法

上传和修改后端逻辑:
-----

用cloudbase-cli提交backend-py：

```
{
  "envId": "default-4gpm7vnrb911600e",
  "functionRoot": "functions",
  "functions": 
  [{
    "name": "fodi",    //fodi就是backend-py
    "timeout": 6,
    "runtime": "Python3.6",
    "memorySize": 128,
    "installDependency": true,
    "handler": "index.main_handler"
  }]
}
```

以下保证要使用https://xxxx.service.tcloudbase.com/fodi形式的调用路径(后台的接入路径定义)：

```
def router(event):
    """对多个 api 路径分发
    """
    door = 'https://' + event['headers']['host']
    print('door:'+door)
    func_path = '' #event['requestContext']['path'] //置空
    print('func_path:'+func_path)
    
    api = event['path'].replace(func_path, '').strip('/') 
    api_url = door + '/fodi' + event['path']  //加一个fodi串

    queryString = event['queryStringParameters']  //这里改
    body = None
    ......
```
弄好后，访问上面的接入路径，仅/fodi，输出path error和后面一长串东西就代表服务器搭建正常，


获取refresh token
-----

然后就是那个refreshtoken的获取，http://scfonedrive.github.io已经挂掉了，我们可以自建，先在某网站下建一个get.html:

```
<html><meta charset=utf-8><body><h1>Error</h1><p>Please set the <code>refresh_token</code> in environments<br>
    <a href="" id="a1">Get a refresh_token</a>
    <br><code>allow javascript</code>
    <script>
        url=window.location.href;
        if (url.substr(-1)!="/") url+="/";
        url="https://login.partner.microsoftonline.cn/common/oauth2/v2.0/authorize?scope=https%3A%2F%2Fmicrosoftgraph.chinacloudapi.cn%2FFiles.ReadWrite.All+offline_access&response_type=code&client_id=04c3ca0b-8d07-4773-85ad-98b037d25631&redirect_uri=https://scfonedrive.github.io&state="+encodeURIComponent(url);
        document.getElementById('a1').href=url;
        //window.open(url,"_blank");
    </script>
    </p></body></html>

如果是国际版url换成:
 url="https://login.microsoftonline.com/common/oauth2/v2.0/authorize?scope=https%3A%2F%2Fgraph.microsoft.com%2FFiles.ReadWrite.All+offline_access&response_type=code&client_id=4da3e7f2-bf6d-467c-aaf0-578078f0bf7c&redirect_uri=https://scfonedrive.github.io&state="+encodeURIComponent(url);

```

再在根下建一个index.html
```
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
	<head>
		<title>OneManager jump page</title>
	</head>
	<body>
		<a id="direct">No link here!</a><br>
		If not auto jump, click the link to jump.<br>
		如果长时间未跳转，请点击上方链接继续安装。<br>
		<label id='test1'></label>
		<script>
      			var q = new Array();
			var query = window.location.search.substring(1);
			//document.getElementById('test1').innerHTML=query;
			var vars = query.split("&");
			for (var i=0;i<vars.length;i++) {
				var pair = vars[i].split("=");
				q[pair[0]]=pair[1];
			}
			var url = q['state'];
			var code = q['code'];
			if (!!url && !!code) {
				url = decodeURIComponent(url);
				if (url.substr(-1)!="/") url+="/";
				var lasturl = url+"?authorization_code&code="+code;
				document.getElementById('direct').innerText = lasturl;
				document.getElementById('direct').href = lasturl;
				window.location = lasturl;
			} else {
				var str='Error! 有误！';
				if (!url) str+='No url! url参数为空！';
				if (!code) str+='No code from MS! 微软code为空！';
				document.getElementById('test1').innerHTML=str+'<br>'+decodeURIComponent(query);
				alert(str);
			}
		</script>
	</body>
</html>
```

把get中scfonedrive.github.io换成你的index.html所在的网站地址，（之后保证把py后端中用于认证的地址和那个clientid,clientsecret替换用你自己新建的一个，具体方法见我前面的一些文章）

调用后结果显示在url中（整个页面显示404是没有处理结果的php后端，除非你把https://github.com/qkqpttgf/OneDrive_SCF部署在index.html所在的网站），分辨复制即可。

安排好后端和refreshtoken后，调用接入路径/fodi/fodi/，输出看到其输出的加密的json结果，就代表refreshtoken也正常了。
开始部署前端，可以另外一个网站，能托管html的就行。也可以在后端另起一函数，部署如下index.py:

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-


def main_handler(event, context):
    f = open("./front.html", encoding='utf-8')
    html = f.read()
    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "headers": {'Content-Type': 'text/html; charset=utf-8'},
        "body": html
    }
```

front.html当然是配置好的那个前端文件。

如果你fodi前端调用发生for each,length之类的提示错误，往往是refresh token没获取对。如果发生跨域错误（chrome f12可看到），则在后端面板中需要配置一条客户端网站的安全域名。

------

整个代码也较大，我们稍后将3rd依赖库精简掉，把那个加密逻辑去掉让输出常态化。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108241192/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




