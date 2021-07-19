利用fodi给onemanager前后端分离(2):测试json
=====

__本文关键字:利用onemanager给fodi做php后端__

在前面《利用fodi给onemanager前后端分离1》中我们介绍了在cloudbase上安装fodi py后端的方法，这里继续尝试将om作为fodi的后端也尝试弄上。

这里要说个历史，fodi的作者也是参考了onemanager的,精简的onemanager流程就是fodi backend那些(https://github.com/qkqpttgf/OneDrive_SCF即是那个最简的php后端)。别看onemanager比fodi体量比fodi大，它其实后端就二主体逻辑文件。platform/TencentSCF.php和index.php(转common.php)，一个是处理scf的，一个实际上是将od api结果转换成om api结果并渲染的结果。我们的目的非常明确：om最终返回一个网页html，onemanger混合了前端在输出结果里。作为一次api response，所以我们需要给它前后分离一下。让它返回正常的类fodi后端的json结果，而不是post render过的html，然后考虑对接到fodi前端。

纯 api 服务器:基础工作
-----

所幸虽然onemanager本身是客服输出合体的，但内部也是细分到不同函数的易区分，客服也进行了区分。这使得我们的工作较为容易进行：我的版本是https://github.com/qkqpttgf/OneManager-php/commite439ed8e68c4ae0d8d42bc5c293a3ba06aa1bc9c，20200607-1856.19，主线逻辑：main()->list_files->files_json(),or render_list()，分解重要的函数或逻辑有二支：（1）list_files($path)->fetch_files($path)，（2）render_list($path = '', $files = '')->fetch_files($path)->output，renderlist就是客户端输出逻辑。onemanger支持https://www.xxxx.com/?json的调用，这个函数就是common中的function files_json($files)，会输出json，相当于在fodi后端index.js中?path输出json那些逻辑。接下来的工作就是这二个函数的处理。

我们的测试环境是在一台VPS中(对应platform/normal)进行，我是在宝塔面板中完成的，分成二个网站部署好om代码和fodi frontend。然后fodi原来的后端我是在cloudflare中开免费worker搭建的。fodi front复制成二份html放在第二个站下，我们的思路是比对这二套系统的前后端，以促成我们的目标。事先在文件夹中放一些文件，还有一些准备工作：

om需要预处一下：服务端index.php顶层代码块（这里并没有处在一个函数中）中：

```
} else { include 'platform/Normal.php'; //从这句开始改
    //$path = getpath(); 注释掉
    $_GET = getGET()后面加一条：
    $path = $_GET['path']; //$_GET是收集到的用户输入的参数组成的数组，该getGet()在platform/normal.php下，这里是得到path项,这样，用户在浏览器中输入什么网址(http://apiurl/?path+参数)，就会展示该路径下的json结果
```

服务端common.php这个三合一平台通用接下来逻辑中，main($path)函数中：

```
if ( isset($files['folder']) || isset($files['file']) ) {
  return render_list($path, $files); //原来的带完整html输出的注释掉，换成接下来这句
  return files_json(/*$path, */$files);
```

function files_json($files)函数末尾：
```
return output(json_encode($tmp));
return output(json_encode($tmp), 200, ['Content-Type' => 'application/json']); //这条实际上在新版被修复了
```

客户端fodi那个frontend html中：要将/fodi/去掉或模拟出来

```
window.GLOBAL_CONFIG = {
SCF_GATEWAY: "https://apiurl/",    //这里你也可以用https://apiurl/?json如果上面你没有把render_list改成files_json
SITE_NAME: "FODI",
IS_CF: true
};
if (window.GLOBAL_CONFIG.SCF_GATEWAY.indexOf('workers') === -1) {
  window.GLOBAL_CONFIG.SCF_GATEWAY += '/';   //window.GLOBAL_CONFIG.SCF_GATEWAY += '/fodi/';改成/
  window.GLOBAL_CONFIG.IS_CF = false;
}
```

在宝塔服务端网站的设置->配置文件中，加好ssl，并把web安全域名设置一下，否则接下来chrome f12调试会产生跨域错误（cloudflare没有这个问题）。chrome这东西不光是web browser，也是webdever的IDE加调试器，其实这个也可以在程序逻辑中设置，但比较麻烦。

```
for nginx:
server块location下，直接放在伪静态里也可以
add_header 'Access-Control-Allow-Origin' '*';

for apache:
	</Directory>
	Header set Access-Control-Allow-Origin "*"
</VirtualHost>
```

然后就可以接下来调试了(注意不要用safari尽量用最新的google chrome，支持度足够，macos big sur之后的safari才能返回正确结果)，调试程序最难的是找到调试的方法，以上这些都可以在chrome的f12，network->request,respone中看到，否则还是chrome f12产生错误，对于后端，你加入的调试只能在云函数执后台看到，对于前端，在chrome中查看。要区分哪些?path是服务端测试调用的，哪些是客户端的。

纯 api 服务器:调试工作
-----

在fodi构造https://apiurl/?path=%2Fd%2Fmirrors之类的url（%2F是/），比对cf和php后端观察到输出的结果。结果是都不会变的。喂给fodi的?path=返回结果不会变的，因为它是接受json请求的而不是GET形式的?path。喂给php端的需要再处理一下。

这是om输出的：

```
{"list":[
{"type":1,"id":"01AL5B7D25YGLLNRUY5FG3INXIYG5BDB5V","name":"d","time":"2020-07-23T14:14:52Z","size":14716769375,"mime":null},
{"type":1,"id":"01AL5B7D26ZI4V2QKN35HJSUUZPTHQ3KQN","name":"docs","time":"2020-08-02T05:59:38Z","size":5640494,"mime":null},
{"type":0,"url":"https:\/\/balala...","id":"01AL5B7DY3H337ZRZ3BNAIUSPVR5RXLU2M","name":"readme.md","time":"2020-08-05T12:25:06Z","size":1311,"mime":"application\/octet-stream"}
]}
```

这是fodi的（在cloudflare那个界面可以调试到）：

```
{"parent":"/","files":[
{"name":"d","size":14716769375,"time":"2020-07-23T14:14:52Z"},
{"name":"docs","size":5640494,"time":"2020-08-02T05:59:38Z"},
{"name":"readme.md","size":1311,"time":"2020-08-05T12:25:06Z","url":"https://blalaa...."}
]}
```

我们的思路就是让其输出一致，而且由于fodi客户端那个html是用的ajax请求ajax结果，?path并不是提交用的。而是作为form data被返回的。这是后来的问题，先处理输出一致。

因为正确的json是parent,files,name,size,time,url，所以在common.php function files_json($files)中：

```
....
$tmp['list'] = [];改为        $tmp['files'] = [];
......
foreach ($files['children'] as $file) {
  ...
  $tmp1['name'] = $file['name']; //包括这句，以下三新加
  $tmp1['size'] = $file['size'];
  $tmp1['time'] = $file['lastModifiedDateTime'];
  ...
  // $tmp1['type'] = 0;   //这句注释
  // $tmp1['type'] = 1;  //这句注释
  //$tmp1['id'] = $file['id']; //包括这句。接下来5句注释，中间3句被移到上面了
  //$tmp1['name'] = $file['name'];
  //$tmp1['time'] = $file['lastModifiedDateTime'];
  //$tmp1['size'] = $file['size'];
  //$tmp1['mime'] = $file['file']['mimeType'];
  .....
  array_push($tmp['list'], $tmp1);改为array_push($tmp['files'], $tmp1);
  ......
```

加入parent,在main()合适的位置（对应于它在结果中输出parent字段在整个全部字段所在的位置，即$tmp['list'] = [];改为        $tmp['files'] = [];一句上面）加上

```
$a= str_replace($_SERVER['list_path'],"",array_pop(explode(":",$files['parentReference']['path'])));
if (isset($a)) { 
	$tmp['parent'] = $a;
}
if ($a =='') {
	$tmp['parent'] = '/';
}
```

不断测试url，随着path参数的改变，json终于有fodi相同的结果。这样，fodi终于接受到初步正确的数据了(仅初始)，，但是到现在为止,点击任何条目包括文件夹都不会出来正确结果，如上所说，我们只是让后端返回了我们认为正确的结果，我们的程序靠喂?path这个工作。到现在为止这仅是一种手动工作。fodi frontend并不与之联动，这是第一步，fodi前后端它是自动联动api path参数变化经的。比如，查看chrome f12,客户端还接受其它参数&encrypted=&plain=&passwd=undefined。（云开发会校验网页应用请求的来源域名，您需要将来源域名加入到WEB安全域名列表中。）

这是由于fodi客户端那个html是用的ajax请求ajax结果，?path并不是提交用的。而是作为form data被返回的。一个是用户发动的url path，一个是xhr发动的url request path，这二者不是一回事。fodi客户端服务端交互有它自己的逻辑。这里面还有复杂的ajax参数交互（查看chrome f12 ajax类请求产生的xhr对象）：

比如，消息体中的数据起作用,对于URI字段中的参数不起作用，被封装在xmlhttprequest的formdata中。服务端要还原处理这类formdata，，，这一切是因为服务器端请求参数区分Get与Post。get 方法用Request.QueryString["strName"]接收，而post 方法用Request.Form["strName"] 接收，blala..ajax还有其它细节。下一步：与fodi frontend对接。可能需要更多研究。

-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108304896/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





