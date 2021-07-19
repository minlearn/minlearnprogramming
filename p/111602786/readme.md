为什么学js，a way to illustrate webfront dev and debug essentials in raw js/api，及用js获取azure AAD refresh token for onedrive
=====

__本文关键字:why xxx lang series，使用pure js获取微软azure AAD refresh tokenfor onedrive，安装chrome扩展禁用浏览器跨域保护__

在前面我们讲到了《我为什么选择rust》，《why elmlang:最简最安全的full ola stack的终身webappdev语言选型》，这些都是标题中已经讲明的why xxx lang series，但其实，在本书所有提到语言选型，其它选型的相关章节中，都是某种意义上的”why xxx series  vs yyy,zzz,blaaa...“，我们还有一些专门针对语言合理性和简单性对比的文章，如《lua/js/py复杂度分析，及terralang:一种最容易和最小的“双核”应用开发语言》和《一种最小(限制规模)语言kernel配合极简（无语法）扩展系统的开发》，在《why elmlang:最简最安全的full ola stack的终身webappdev语言选型》中我们进一步介绍了elm，并提到其与一些语言的简单合性理比较。甚至还有py,perl,ruby,php等。

本文要谈到的js，它是elmlang的直接近祖，在简单合理性方面，如果说py,perl,php,ruby这些通用无限领域语言是一派，那么elm,js,shell局限化一体环境语言就是醒目的另一派：

> js的最大优势是简单合理，JS的简单是是因为它绑定了应用环境和调试环境。与前端这种开发领域形成非常紧密的DS-L对应,整个js生态跟bash与shellprogramming一样，是环境与语言绑定的，所以调试方便。（它本身属于让js处在一个单进单线环境中的可调用环境,language in final app as lanuage runtime结构，类似游戏编辑器和编辑器脚本的组合,vs通用语言。因为它们绑定一个最终运行环境，可以避免因提供的类型和内置数据，都较泛化，调试环境依赖不断搭建，所带来的学习成本。

> 合理是，它基于c和函数式。这基本是现代编程流派的二大代表，每样都只取一点点又恰当好处，js非常精简，它混合一部分C和一部分函数式。使得不依赖OOP中过渡泛滥的观察者设计模式，用函数就实现了效果接近MVC。事件相当于一个主题(Subject)，而所有注册到这个事件上的处理函数相当于观察者(Observer)。要知道js函数比类更适合当XXX者XXX对象。nodejs并非通用语言没有多线程和胶水能力那些,但它的IO很快，多线程带来的效果可以不计，胶水能力需求倒显得其次，由于js本身就支持这种异步IO，所以库级的观察者订阅者模式，只是薄薄的一层就能实现。甚至不到10行。异步IO天然就是网络服务交互处理DSL的。它本身也是一种MVC绑定语言。框架绑定语言。专门用来写WEB，它对WEB的那些机制几乎是一比一原生的。虽然js及其生态，包生态有一些缺点，但这是语言发展过程中不能避免的。

其实，类似观点在《一种开发发布合一，语言问题合一的shell programming式应用开发设想》《elmlang：一种编码和可视化调试支持内置的语言系统》，《编程实践选型通史：平坦无架构APP开发支持与充分batteryincluded的微实践设施》,都涉及过）解决编程难度的真正途径只有一个，就是装配一个免IDE的调试环境，让程序与环境一起绑定和出现。让源码级的业务逻辑所指的环境天然出现。就像plan9一样，成为任何语言APP的寄体，OS作为云原生环境而不是k8s。(这也是本书选型shell和web开发作为实践部分的目的所在)

我们先从一个例子开始，这个例子是js的原生前端应用领域的典型,原生web services api交互，第一步
-----

在《利用fodi给onemanager前后端分离(1)：将fodi py后端安装在腾讯免费cloudbase》的后面部分我们讲到将onemanager获得refresh token的过程用几个html搭建，使之脱离onemanager php后端的工作，在那里，我们的工作其实并没有达成，获得的refreshtoken要从url中复制，且放到onemanager的disk配置中并不生效，

这是因为我们只做了第一步：获得一个随机refresh token的过程，其它的都没有做，其实这里有二个过程，有二个html，除了前面的html和生成refreshtoken的工作，这个随机refresh token还必须要再次向azure交互一次，这里来完成它：

先给出第一步的html和逻辑解释：

```
<html><meta charset=utf-8><body><h1>get refresh token for onedrive (in static)</h1>
<body><p>allow javascript,and prepare the app reg before starting below:


<div>
    <br>
    <label><input type="radio" name="Drive_ver" value="CN" checked onclick="document.getElementById('morecustom').style.display='';document.getElementById('inputshareurl').style.display='none';">CN: 世纪互联版</label><br>
    <label><input type="radio" name="Drive_ver" value="MS" onclick="document.getElementById('morecustom').style.display='';document.getElementById('inputshareurl').style.display='none';">MS: 国际版（商业版与个人版）</label><br>
    <label><input type="radio" name="Drive_ver" value="shareurl" onclick="document.getElementById('inputshareurl').style.display='';document.getElementById('morecustom').style.display='none';">ShareUrl: 共享链接</label><br>
            
    <br>
    <div id="morecustom">
        <label><input type="checkbox" name="Drive_custom" checked onclick="document.getElementById('secret').style.display=(this.checked?'':'none');">自己申请应用ID与机密，不用OneManager默认的</label><br>
        <div id="secret" style="margin:10px 35px">
            <a href="https://portal.azure.cn/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/Overview" target="_blank">申请应用ID与机密</a><br>
            client_id:<input type="text" name="client_id" id="a1"><br>
            client_secret:<input type="text" name="client_secret" id="a2"><br>
        </div>
        <label><input type="checkbox" name="usesharepoint" onclick="document.getElementById('sharepoint').style.display=(this.checked?'':'none');">使用Sharepoint网站的空间，不使用Onedrive</label><br>
        <div id="sharepoint" style="display:none;margin:10px 35px">
            登录office.com，点击Sharepoint，创建一个网站（或使用原有网站），然后将它的站点地址填在下方<br>
            <input type="text" name="sharepointSiteAddress" style="width:100%" placeholder="https://xxxxx.sharepoint.com/sites(teams)/{name}"><br>
        </div>
    </div>
    <div id="inputshareurl" style="display:none;margin:10px 35px">
        对一个Onedrive文件夹共享，允许所有人编辑，然后将共享链接填在下方
        <input type="text" name="shareurl" style="width:100%" placeholder="https://xxxx.sharepoint.com/:f:/g/personal/xxxxxxxx/mmmmmmmmm?e=XXXX"><br>
    </div>

    <br>
    <input type="submit" id="btn" value="点击获得一个refresh token">

</div>


<script>
......
</script>


</p></body></html>
```

在这个典型简化div-script结构的这样一个html页面中，我们看到了js内嵌于browser，用语言控制dom元素,形成前端效果的原始操作能力，<div>和div内的子元素，子div元素都存在上下层次关系，document.getElementByxxx('xxxx')就是这复杂关系下统一的选择符，js分散在可点击元素的onclick过程中控制这些效果。，它们也可写成对应的<script>......</script>块，对于有ID的元素，它用document.getElementById('xxxx')，对于只有tag或name的，document.getElementByName('xxxx')返回的是数组，因为ID是页面上唯一的，而name是一种归类标识，一个页面上可以有好几个同样name的元素存在，（document.getElementByxxx('xxxx')这种完写法会比较累，当然jquery这种库会类似perl和bash一样定样一些专用符号简化这些逻辑，这毕竟是外来库，不讲。）

讲完了前端效果控制，下面是主要<script>......</script>块中，干实事的逻辑了：

```
    //function notnull(t)
    //{
    //    if (t.Drive_ver.value=='shareurl') {
    //        if (t.shareurl.value=='') {
    //            alert('shareurl');
    //            return false;
    //        }
    //    } else {
    //        if (t.Drive_custom.checked==true) {
    //            if (t.client_secret.value==''||t.client_id.value=='') {
    //                alert('client_id & client_secret');
    //                return false;
    //            }
    //        }
    //        if (t.usesharepoint.checked==true) {
    //            if (t.sharepointSiteAddress.value=='') {
    //                alert(''.'InputSharepointSiteAddress'.'');
    //                return false;
    //            }
    //        }
    //    }
    //    document.cookie='disktag='+t.disktag_add.value+'; path=/';
    //      return true;
    //}


    clientid=document.getElementById('a1').value;
    clientsecret=document.getElementById('a2').value;
    document.cookie="clientsecret="+clientsecret;
    document.cookie="clientid="+clientid;

    url=window.location.href;
    if (url.substr(-1)!="/") url+="/";
    url="https://login.partner.microsoftonline.cn/common/oauth2/v2.0/authorize?scope=https%3A%2F%2Fmicrosoftgraph.chinacloudapi.cn%2FFiles.ReadWrite.All+offline_access&response_type=code&client_id="+clientid+"&redirect_uri=http://localhost:8000/SCFOnedrive.github.io/CN/&state="+encodeURIComponent(url);


    document.getElementById('btn').onclick = function (e) { window.location=url;}

```

首先，（我们这里以互联版作参照）你应该在https://portal.azure.cn/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/Overview中按《利用fodi给onemanager前后端分离(1)：将fodi py后端安装在腾讯免费cloudbase》说的注册好应用，回调地址中填一条http://localhost:8000/SCFOnedrive.github.io/CN/，azure规定重定向 URI 必须以方案 https 开头。 但这类localhost 重定向 URI 例外，因为测试时我们都是本地测试的，因此azure允许这类例外存在。//注释的那一大段是检测你输入有效性的，图省事没有用起来，

这里的逻辑主要是获得clientid输入框a1,clientsecret输入框a2，存入cookie供接下来要转向到的那个长长的url对应的页面中（所以你的浏览器须允许js允许cookie）使用（跨页面传递变量要借助cookie而不再总是url分析,其中clientid在本页中直接放到url中应用了一次），然后这个url是点击button直接转向过去的window.location=url，表明这个长长的url是一次面向azure的get提交，跟直接在浏览器中粘贴地址访问得到什么样的返回，效果一致，redirect_uri=http://localhost:8000/SCFOnedrive.github.io/CN/，表明https://login.partner.microsoftonline.cn/common/oauth2/v2.0/authorize要向你设置的这个回调页面传递结果,它将转向到:http://localhost:8000/SCFOnedrive.github.io/CN/?code=xxxxxx&state=http%3a%2f%2flocalhost%3a8000%2fSCFOnedrive.github.io%2f&session_state=xxxxx#这样一个页面，传递的结果就是那个长长的生成的code，这是随机refresh token。开头说过，它并没有生效。需要在接下来的步骤中再跟azure交互一次。

> client id,access token，refresh token,这些其实都是在主帐号下分出子帐号，属于客户端调用专用验证机制，这样就不用涉及到主帐号。这是一次性的，用access secret验证完refreshtoken后，call back page其实用不着access secret，主程序也用不到callback page，以后的每次调用也不必维持这个callbackpage。甚至refresh token可以用别人的client id,access token得到。为什么本地要维护一个转向（至APP内部）处理页面呢，因为这是web services之间的交互逻辑。认证。多个不同来源的只能通过callback，转向这向逻辑来相互传递数据。


我们先从一个例子开始，这个例子是js的原生前端应用领域的典型,原生web services api交互，第二步
-----

这是第二个页面，回调页面,/CN/index.html

```
<html><meta charset=utf-8><h1>azure callback page</h1><p>


<div>
    组装一条含随机refreshtoken的注册链接形式：<br><br>
    <a id="t1">No link here!</a><br>

    <br>
    调用注册链接并分析返回结果，请自行判断并复制！<br><br>
    <label id='t2' style="width: 95%">no result here!</label><br>
</div>

<script>
......
</script>

</p></body></html>
```

页面效果已尽量简化，重点是在<script>......</script>中：

```
    function getCookie(name) {
        var arr;
        var reg = new RegExp("(^| )" + name + "=([^;]*)(;|$)");
        if (arr = document.cookie.match(reg)){return arr[2];}
        else return null;
    }


    function getContent(url2,url22) {
        var xhr = new XMLHttpRequest()
        // post请求方式，接口后面不能追加参数
        xhr.open('post', url2)
        // 如果使用post请求方式 而且参数是以key=value这种形式提交的
        // 那么需要设置请求头的类型

        xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')
        //xhr.setRequestHeader('Origin', 'http://localhost:8000')

        xhr.send(url22)
        xhr.onreadystatechange = function () {
            // 数据全部返回的判断
            if (xhr.readyState == 4) {
                document.getElementById('t2').innerHTML=xhr.responseText
            }
        }
    }


    var q = new Array();
    var query = window.location.search.substring(1);
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

        var url2 = "https://login.partner.microsoftonline.cn/common/oauth2/v2.0/"+"token";
        var url22 = "client_id="+getCookie("clientid")+"&scope=https://microsoftgraph.chinacloudapi.cn/Files.ReadWrite.All offline_access"+"&client_secret="+getCookie("clientsecret")+"&grant_type=authorization_code&requested_token_use=on_behalf_of&redirect_uri="+"http://localhost:8000/SCFOnedrive.github.io/CN/"+"&code="+code;
	document.getElementById('t1').innerText = url2+"?"+url22;


	getContent(url2,url22);
        //if ($tmp['stat']==200) $ret = json_decode($tmp['body'], true);
        //document.getElementById('t2').innerHTML=result;


	} else {
	    var str='Error! 有误！';
	    if (!url) str+='No url! url参数为空！';
	    if (!code) str+='No code from MS! 微软code为空！';
	    document.getElementById('t1').innerHTML=str+'<br>'+decodeURIComponent(query);
	    alert(str);
	}
```

这里的流程正如页面效果中提到，先组装一条含随机refreshtoken的注册链接形式，然后调用注册链接并分析返回结果，请自行判断并复制，这一切都在pure js和单个页面中用ajax xhr完成，而类似onemanager这样的后端安装界面，用的是封装了服务器能力的curl_request，用的是post www-form-data，再次将得到的结果转到其它页面处理的，比如保存这个refresh token，而我们直接展示输出返回结果让用户判断是否成功决定复不复制了事。

在整个逻辑中，先定义二个函数，一个是取cookie，由于cookie只能先取得cookie数组，所以只能从数组中按名字正则方式取值，模拟关联数组，一个是取返回结果，这里的url和取得url的返回结果的方式和第一步有较大不同，因为这个callback和azure交互的方式是post，而且要以非url参数形式提交数据，普通情况下我们要准备一个form收集这些数据。且作为前端的js传递上并没有xhr这样的东西，无法做到curl这样的服务器函数效果。只不过我们现在的浏览器幸运地支持它，可以模拟curl。

getContent(url2,url22)就是上面提到的效果对应的函数，准备url2,url22请求接收方,请求发起方双方数据，url2即页面与azure交互的后端：https://login.partner.microsoftonline.cn/common/oauth2/v2.0/token(注意这个与前面一步不同)，要求的参数也不同（url22作为data发送，而不是url参数），写到t1中(由url2+"?"+"url22"形成的这个url不能直接在浏览器中访问，AADSTS900561: The endpoint only accepts POST, OPTIONS requests. Received a GET request)，这个交互url在函数中用POST提交方法，setheader提交头部信息进行，这样处理端就准备好了，接下来逻辑主流程会收集这个callback页面的发起方url，分析出发起请求参数用的code，即那个随机refresk token，在if else流程中的if部分，会调用getContent(url2,url22)会将异步调用得到的结果写进document.getElementById('t2').innerHTML代替初值（如果输出一个正常的json就对了那么refresh token就生效了，如果其它就是不成功的，可能是多次调用同一clientid,client sercet发生redeem了，返回第一页重来）。else退出逻辑。

> 这里的调试手段，是借助chrome的f12来调试。也可以console.log()输出变量在chrome中查看。也可以alert()，也可以用curl -X或postman,postman是一个类似cloudflare那个界面的东西(话说这个东西做在浏览器作调试器多好)。https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow，点"run postman"会自动将测试case导入到postman，我们要用的是“Token Request - Auth Code”，在postman中，要去body,xwww-form-urlencode中把参数弄进去，而不是最开头那个中Params，否则提示"AADSTS900144: The request body must contain the following parameter: 'grant_type'.，如果是curl -X方式，curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'xxx&scope=xxx&client_secret=xxxx&grant_type=xxxx' 'xxxx'

> 这里特别注意一个问题，curl和postman和browser（chrome）处理web service 交互的逻辑有点差别，chrome对跨域有限制，在本地利用localhost这样的地址测试时，会发生跨域错误。        //xhr.setRequestHeader('Origin', 'http://localhost:8000') 无效，请参照《3 Ways to Fix the CORS Error — and How the Access-Control-Allow-Origin Header Works》类似的文章，使用改变浏览器行为的笨办法，The quickest fix you can make is to install the moesif CORS extension，安装好了把它由OFF调到ON，从头开始。


------

在后端,js的开发，环境结合性和可调试性也是一样的。



