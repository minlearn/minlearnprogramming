让owncloud成为微博式记事本
=====

__本文关键字：利用owncloud做网上记事本__

相信不少人有把QQ空间之类的社交类，微博类程序当网上记事本用的经历，谈到记事本程序，有微博这种（加密或不公开），有mail based note俗称电子邮件便笈，也有利用emlog之类的自建碎语的，当然wordpress+插件或p2p wordpress theme也可以，还有人用wordpress页面+评论打造，当然，直接txt文档或excel表格也可以，单个mediawiki页面也可以(它支持页面内多条记事的归档，即一个页面一个归档)，所以说，网上记事这种需求是很普遍的，尤其是文字工作者（程序员，文学爱好者等等）。------- 程序员的能力几乎都在于速学和实践，大多的细节易忘，一段代码，一个DEBUG必知，所有这些记事在这个环节上必不可少。

当然，现在流行的方案是使用专门的网上的记事程序像有道笔记，印象笔记evernote etc..，还有leafnote，与作者一直说的web程序portal化和面向存储后端集中化比较接近，它们往往有完善的移动端支持，不过它们托管在别人主机上，备份和使用上都往往不尽人意。

加密的微博帐号往往最符合发便条的使用习惯，打开就是一个框，填入就可发布，反而现在的一些大而全的记事程序做得过于完善，比如不需要支持html内容，好了，适合自己用的就是最好的，我个人觉得，记事不需要像写博一样配备一个wywiws editor。

在前面《利用owncloud打造static web hosting空间》一文中，我们使用ownnote打造了一个静态网站托管空间，我们还说到，其实这种掩藏于portal之下的动态程序+呈现在www的静态展示，其实可以构成任何owncloud based cms程序，甚至强大的通用网站程序，这种网站程序的特点是：中心化存储，有OC storage backend支持，便于维护，打造和迁移。而在这篇文章中我们同样利用的是ownnote+OC的方案，且可以拥有前者所有的好处。

第一步是，在OC上打造多个ownnote app，我使用的是oc9+ownnote app 1.08。

在oc上安装二个ownnote app
-----


由于oc上ownnote不可以直接改文件夹名来安装同一个APP产生不同的APP实例，因此我们需要修改ownnote的源码来使OC支持安装二个OWNNOTE，多个ownnote的存在，对于使用它来构建记事程序才是有意义的，否则，oc+ownnote只能使OC成为数量上仅限一个的记事程序，也限制了在这个OC上打造static web hosting的能力。

其实，总体的修改逻辑，就是把源码中ownnote字眼改成ownnote2，先上传一份同样的ownnote2与原ownnote并列，然后进行修改，下面只讲重点部分：

1)core部分，这部分必改，使得APP不与原ownnote混淆(程序逻辑，页面和显示正确)：

appinfo目录下app.php:

```
\OCP\App::registerAdmin('ownnote2', 'admin');

'id' => 'ownnote2',

'name' => \OCP\Util::getL10N('ownnote2')->t('Notes2') //这个使APP上显示notes2，如果是作staticwebhosting用，可改成posts，以与notes区别
```

appinfo目录下application.php:


```
namespace OCA\OwnNote2\AppInfo;

.....

use \OCA\OwnNote2\Controller\PageController;

use \OCA\OwnNote2\Controller\OwnnoteApiController;

use \OCA\OwnNote2\Controller\OwnnoteAjaxController;

(像以上关于命名空间的，在其它原码文件中也一律改为ownnote2，将从2)开始不再列出)

controller目录下pagecontroller.php: $response = new TemplateResponse('ownnote2', 'main', $params);
```

app根目录下admin.php:$tmpl = new OCP\Template('ownnote2', 'admin');

template下main.php：

```
\OCP\Util::addScript('ownnote2', 'script');

\OCP\Util::addScript('ownnote2','tinymce/tinymce.min');

\OCP\Util::addStyle('ownnote2', 'style');

$disableAnnouncement = \OCP\Config::getAppValue('ownnote2', 'disableAnnouncement', ''); \\这句应该移到3)数据库条目部分

$l = OCP\Util::getL10N('ownnote2');
```

template目录下admin.php

```
\OCP\Util::addScript('ownnote2', 'admin');

.....

$l = OCP\Util::getL10N('ownnote2'); //这句显示管理界面左边显示ownnote2
```

2)URL部分，这部分使得打开app时，是正确的http://oc/index.php/app/ownnote2：

appinfo目录下app.php : 'href' => \OCP\Util::linkToRoute('ownnote2.page.index'),

appinfo目录下application.php : parent::__construct('ownnote2', $urlParams);

js目录下admin.js : var newurl = OC.linkTo("ownnote2",url).replace("apps/ownnote2","index.php/apps/ownnote2");

js目录下scripts.js : var newurl = OC.generateUrl("/apps/ownnote2/") + url;

3)使ownnote2能正确读写DB条目:

appinfo目录下database.xml中:

```
<name>ownnote2</name> //这个出现在owncloud appconfig中

<name>*dbprefix*ownnote2</name>

<name>*dbprefix*ownnote2_parts</name>
```

appinfo目录下routes.php : array('name' => 'ownnote_ajax#ajaxsetval2', 'url' => '/ajax/v0.2/ajaxsetval2', 'verb' => 'POST')

js目录下admin.js : 所有$.post(ocOwnnoteUrl("ajax/v0.2/ajaxsetval2"), { field: 'folder', value: val }, function (data)

controller目录下ownnoteajaxcontroller.php:

```
public function ajaxsetval2($field, $value) 

所有$FOLDER = \OCP\Config::getAppValue('ownnote2', 'folder', '');

controller目录下ownnoteapicontroller.php:所有$FOLDER = \OCP\Config::getAppValue('ownnote2', 'folder', '');
```

app根目录下admin.php:

```
OCP\Config::getAppValue('ownnote2', 'folder', ''));

OCP\Config::getAppValue('ownnote2', 'disableAnnouncement', ''));
```

lib/backend.php中所有关于db操作的逻辑也改成针对ownnote2。

可能上面的修改部分没有涉及到全部，但是你可以慢慢调试得出正确的ownnote2，这个尝试是必定存在的，请耐心完成。 

比如如果发现产生不了数据库中的db条目，请删除原有的ownnote2条目重新反注册/注册app。

一切完成之后，使用方法为：如果同时存在ownnote和上面弄出来的ownnote2，在后台设置database and folder时的folder时，要禁掉ownnote，先修改ownnote2的，再启用ownnote修改ownnote的path.然后其它就都不用管了。可以像正常ownnote app一样使用app2了。

而且，我们还需要定制它，使得ownnote更好用，一句话，使之更像个人化的微博。

微博式记事定制
-----

首先，我们使得那个新建note的new始终处于被点击状态，出现的name文本框会当成我们上面所说的微博记事的文本框。在js/scripts.js的function bindListing()未加一条：

$("#new").click();

这样一打开ownnote app，name框始终是待填入的。

因为这个文本框是设想用来直接填入记事内容的，所以未来往这个框中填入的东西也许是一些用户从网上复制来的东西，长度不定，内容有特殊字符，所以我们需要定制下save to folder时生成的文件名逻辑，使得填入的内容被置入记事内容体，而生成的文件名才被放置到这里，且经过了便于save to folder的过滤处理逻辑。

js去除特殊字符的函数是：


```
function stripscript(s) 

{ 

var pattern = new RegExp("[`~!@#$^&*()=|{}':;',\\[\\].<>/?~！@#￥……&*（）——|{}【】‘；：\"”“'。，、？]") 

var rs = ""; 

for (var i = 0; i < s.length; i++) { 

rs = rs+s.substr(i, 1).replace(pattern, ''); 

}

return rs;

}
```

而去除空格的JS逻辑为这样：somestr.replace(/\s+/g,"")，故组合一下这二者，很容易得到过滤输入框内容生成文件名的逻辑：stripscript(somestr.substr(0, 50)).replace(/\s+/g,"");

下面我们将这条逻辑放在程序中的适当位置（主要是程序中产生note的地方）：

```
scripts.js中的createnote()中：

var name1 = $('#newfilename').val();

var name = stripscript(name1.substr(0, 50)).replace(/\s+/g,"");
```

文件名有了，我们还要把内容name1一同放进产生note的逻辑中传送到note内容区：

scripts.js中：

```
$.post(ocUrl("ajax/v0.2/ownnote/ajaxcreate"), { name: name, name1:name1, group: group }, function (data) {

loadListing();

});

controller/ownnoteajaxcontroller.php中：

public function ajaxcreate($name,$name1, $group) {

$FOLDER = \OCP\Config::getAppValue('ownnote2', 'folder', '');

if (isset($name) && isset($group))

return $this->backend->createNote($FOLDER, $name,$name1, $group);

}
```

lib/backend.php中：

```
public function createNote($FOLDER, $in_name,$in_name1, $in_group) {

$name = str_replace("\\", "-", str_replace("/", "-", $in_name));

$name1 = $in_name1;

.....

\OC\Files\Filesystem::touch($tmpfile);

\OC\Files\Filesystem::file_put_contents($tmpfile, $name1);

......

$query->execute(Array($uid,$name,$group,$mtime,$name1,''));
```

为了防止用户在编辑note后保存时破坏上述文件名过滤规则，实际上我们还需要在保存note的地方运用以上过滤逻辑：

script.js中的savenote()中：

```
$('#editfilename').val($('#editfilename').val().replace(/\\/g, '-').replace(/\//g, '-'));

// var editfilename = $('#editfilename').val();  //注释这条

.......

var content = tinymce.activeEditor.getContent({'format':'text'}); //使得后台那个可视编辑器只存取txt

var editfilename = stripscript(content.substr(0, 50)).replace(/\s+/g,""); //注释的那条改变一下逻辑放这里
```

这样就算完成了。新的ownnote清奇好用,恩恩！！

当然你也可以修改CSS加入name文本框加大的样式逻辑。这样编辑起来更方便。

-----------

与leafnote之类的相比，有OC作存储。个人觉得更方便，不过leafnote更强大，使用的mongodb也直接支持数据库引擎级的存储，不过OC有自己的特点就是它本身强调集中化APP数据的WEB APP架构设计，这个我更喜欢。

你其实还可以改htm存储为txt格式，这样新建txt，只要重命名txt（作为记事内容）丢到客户端同步的文件夹就能记事了。

官网andriod客户端支持的发贴逻辑需要定制才能。而且它只针对http://oc/index.php/apps/ownnote/，所以对于一个oc同时有二个ownnote的情况下，第一个要预留成记事而不是static web hosting。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339717/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




