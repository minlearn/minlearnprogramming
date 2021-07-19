利用onemanager配合公有云做站和nas（2）:在tcb上装om并使它变身实用做站版
=====

__本文关键字：tcb上安装onemanager,onemanger实用做站__

在《利用大容量网盘onedrive配合公有云做你的nas及做站》中，我们初步谈到了使用onemanager配合云函数或云主机做站的投入和尝试，前文说到不能在腾讯cloudbase上安装onemanager，但实际上经过尝试发现是可以的：

注意：云函数产品建议使用包月免费或包月套餐尝试。按量如果控制不好，可能会因为代码问题或外部攻击造成高额费用，比如腾讯scf按量下如果免费额度用完，24小时才停机不会立马停，而且它的计费即使你帐号没钱也会往负了扣，极可能超支。包月则不会有预算超支。

所以我们接下来讲解在包月免费的tcb上安装它：

在tcb上安装onemanager
-----

首先，从http://github.com/qkqpttgf/OneManager-php下载代码，先不上传到cloudbase空间，本地修改platform/tencentscf.php的GetGlobalVariable($event){...}函数体中的$_GET = $event['queryString']为$_GET = $event['queryStringParameters']，这样?admin等参数传递就正确了。然而程序还是得不到入口index.main_handler，直接使用cloudbase后台的新建函数只能用index.man作入口，手动修改入口可以执行，但程序会进一步得不到环境变量，我们可以统一使用cloudbase cli命令行工具全面定制：

cloudbase cli是一个nodejs程序。按腾讯产品文档在本地安装后tcb login --key登录，填入你的用户access keyid和keysecret，在本地做一个待上传目录，在此目录下写如下内容的cloudbaserc.json，同时准备子目录：functions/myonemanager/下放经过上面修改的onemanager代码，到待上传目录(你也可以建一个目录myonemanager，把om源码和cloudbaserc.json统统放进去不用建functions/myonemanager子目录,但是下面cloudbaserc.json中的functionroot要改为../)：

```
{
  "envId": "你的环境",
  "functionRoot": "functions",
  "functions": 
  [{
    "name": "myonemanager",
    "timeout": 6,
    "runtime": "Php7",
    "installDependency": true,
    "handler": "index.main_handler有了这个就不用改入口了",
    "envVariables": {
      "Region":"ap-shanghai",
      "SecretId":"你的腾讯accesskeyid",
      "SecretKey":"你的腾讯accesskeysecret",
      "admin": "你要定义给后台的密码，明文",
      "sitename": "站点名，找一个在线base64转码后，将结果填这",
      "hideFunctionalityFile": "1",
      "disableChangeTheme": "1",
      "passfile": "密码文件名",
      "theme": "主题名",
      "timezone": "8",
      "disktag": "盘名1|盘名2",

      "盘名1": "{\"Drive_custom\": \"on\",\"Drive_ver\": \"CN\",\"client_id\": \"你的azure app portal for onemanager的client app id明文\",\"client_secret\": \"你的azure app portal for onemanager的client app secretbase64明文找一个base64转成结果填这\",\"diskname\": \"明文找一个base64转成结果填这\",\"domain_path\": \"明文找一个base64转成结果填这，形式是域名1:/目录1|域名2:/目录2......\",\"refresh_token\": \"看接下来手动获取方法\",\"token_expires\": 9999999999}",

      "盘名2": "{同盘1生成方式}"

    }
  }]
}
```

可以看到盘名1后面的参数是一个字串，然而它本身也是个json，将json转成字串供cloudbase识别的方法是将所有"都\转义一下，如果你嫌麻烦实在想图方便，可以在正常非cloudbase区或vps上直接搭建一个onemanager,注册盘，然后将结果填到上面。
上面的clientid和secretid，正常方式安装od是自动的，但其实你也可以手动https://portal.azure.cn/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps去生成，我这里是互联，新注册->任何组织目录中的帐户,多租户->重定向url:web,https://scfonedrive.github.io，这里可以直接看到client id了。看左边列，api权限不用设置，证书和密码->新客户端密码，期限永久，就看到secret了。
至于refresh token，也可以从https://service-36wivxsc-1256127833.ap-hongkong.apigateway.myqcloud.com/release/scf_onedrive_filelistor手动得到。token_expires填10位9。

cloudbaserc.json准备完毕，最后cd到这个目录，cloudbase functions:deploy，这样你就得到了一个完全手动和程序化的安装方式，后台改变256m到128m,触发路径/或/xx不能是/xx/结尾，以后deploy，提示覆盖直接确认即可。

更多让onemanager实用做站的考虑
-----

我们知道，云函数主要是处理api结果的传送，在这里不能传递大量数据，保证一次http所有结果在最短的ms里完成，否则按调用次数和调用时长及内存占用的云函数会相应产生相对高的花费，查云函数后台，确保每次2ms内的调用是合理和正常的。故onedrive和托管onemanager等程序的空间（这二者最好是同一地域的，比如世纪互联配国内空间，国际版配港区空间）对提高调用速度至关重要，有些onedrive列表程序支持，前后端分离，云函数纯粹后端只返回api结果不包前端渲染。api速度快(列文件很快)如fodi.

处理静态资源问题和定制模板：

由于od是一个特殊的程序，它定位于网盘文件列表而非带资源的网站展示，它绑定的工作域名下，每一个路径，如果不是显式的?setup这种参数，就是文件调用，因此，它对所有js,css的引用，都是外部的(如果发现网页慢，将它换到快点的cdn地址)。这也是为了上面说的一次request/respon能尽快调用完成，所以od的templates都是不带静态asserts的。------ 所以并不推荐将静态template资源放在代码theme下，然后根据判断它是不是网盘文件进一步处理。

谈到od的templates,其实它也是网页模板技术的运用（本质就是定义一系列开头结尾组合形式的模板变量块，然后替换），你可以查看已有template自己写一个比如最简单的那个nexmoe1.html，，模板体逻辑通常是这样的：开头icon处理块，管理相关的style，前端样式style块,外部css和jss引用，渲染omf，md文件的逻辑块(require一个maked js然后根据md content在页面直接render)，。列文件和目录的逻辑（其实又包括div逻辑块，js逻辑块）,blaaaa.....。

加速和cdn：

我们知道网站速度至关重要。不光对用户对运营也是如此。要实用做站的话，必须要配cdn。对于cdn加速，比如要求文件静态化为各个url路径为目录名的目录下的index,html。onemanager有没有相关方面的支持呢

od是带缓存的。主要是存取到云函数backend空间的system temp目录中。这样列文件和目录的时间会相对变少。程序效果和体验会最佳。od内部对text文件(包括markdown)都是有1800秒缓存的。这个过程在common.php中，查看fetch files,render list主要函数，gethiddenpass()等类似函数。

对于md，上面说到它是在客户端通过client js来渲染从服务端拉取下来的内容的（如果发现大量md的网站慢，有可能这个js处在慢速cdn上，换个），，对于html则是跟md一样直接下载并output不经过主题渲染处理，相当于部署了一个静态页面。

本来它是在客户端生成的。其实在服务端也可完成md,比如下载一个php的渲染器mdparser.php，再在index.php中include 'mdparser.php';common.php中在对应headmd处理位置的地方作修改：

```
$parser = new HyperDown\Parser;            
$headmd = str_replace('<!--HeadmdContent-->', $parser->makeHtml(fetch_files(spurlencode(path_format(urldecode($path) . '/head.md'),'/'))['content']['body']), $tmp[0]);

$tmp = splitfirst($html, '<!--MdRequireStart-->');
$html = $tmp[0];
$tmp = splitfirst($tmp[1], '<!--MdRequireEnd-->');
```

在服务端生成html作为api结果返回会稍微增加api时间，但结果更合理。你可以进一步把渲染好的html结果保存在cache中对应md地址的子目录index.html（而不是原来的raw md content）中，然后下回fetch到这个md地址，直接取cache，按处理html的方式，直接render。这样的“全站伪静态”对cdn也是有用的。

你也可以修改refreshcache的逻辑，od有一个refresh cache，它是先切换到当前目录下就refresh哪个目录的cache。且只工作在手动管理模式下，其实你可以把它做成自动静态化的，浏览到对应md位置就生成对应index.html到cache/md命名子目录下。然后在后台做一个一键md全站生成html到cache静态化按钮和功能。或者生成到cloudbase的存储中。---- 已经有这样的程序了，静态网站生成器作为云函数，生成静态文件到oss，云存储。像极了自动化的github page action。

----

ps:。一般来说，根据我的“硬件使用3年即报废，不应超过500元/年，总共应投入1500，超过这个都是这个时代的奢侈品“的嘴强说法，如果做站用的虚拟资产也可归其中，则3年期的1t互联或淘宝买的个人/家庭office365官方版+3年最低配的1h1m云主机或虚拟主机也正好落入这个区间(office365已改为microsoft365,感觉国际个人/家庭版的没有国际商用版的快，优点是前者多了个文件vault库)。可惜bat等不支持终身制虚拟主机（onemanager需要php curl扩展）或终身制弹性云主机,否则这就是一种不虐心云主机的省事投资了。

在前面，我们为oc增加了wp功能和note功能，现在为od增加静态html功能，同样基于做站和静态化考虑，现在，我们对od开刀了。可能最终，我们要类群晖的dsm based on a netdisk,将od类似的理念打造一个mineportal os，而不仅仅停留在一个file lister，这是比云主机黑群还实用的方案，比如myportalnew: email是必须的+类似微信自托管的chat app+file sync+mateos app那些。。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107018115/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



