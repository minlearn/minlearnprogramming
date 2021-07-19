捉虫与寻龙：从0打造wordpress插件wp2oc fileshare (1) – 将wp存储后端做进owncloud
=====

__关键字：wp2oc fileshare，wordpress媒体存进网盘,网盘作为wordpress图床,owncloud wordpress backend storage__

其实用网盘做wordpress网站的图床一直是一个很流行的想法，业界存在oss,七牛网盘，百度盘wp2pcs,wp2pcs_sy方案，不过oss,七牛，百度盘pcs这三者始终是面向外接第三方服务，这些都不能得到服务保障，其中免费且最好用的百度盘pcs api需要申请权限，实用性大打拆扣。

这里我们选择的用owncloud作为wordpress的存储后端，这二者生态相似，完成后的插件可以，1，基本（不能完全）代替wordpress原生图片媒体管理功能，2，网盘图床的操作/备份符合在文件夹操作文件习惯，且可以网盘特有的同步方式进行备份和打包，3，当媒体文件很大时，转移wordpress整个媒体也就是改一条外链。不再需要涉及到数据库备份。4，当然还有更多。。

1，确立需求：我们仅需要开发一个APP
-----

我们需要的仅仅是将owncloud存储服务做进wordpress，owncloud有自己的rest api,可以将其服务以wordpress插件的方式做进wordpress形成其后端图床。

我们找到的是ocs filessharing api,为什么必须是fileshare而不是file呢，因为做图床的网盘必须是可以外链的。主要用到的是其get all share部分，所需的参数形式是http://www.xxx.com/ocs/path?=/dir&subfiles=true，首先对于使用到的参数部分我已经在后台加了设置接口了，主要就是四个：

接下来就是开发和调试了

PS：开发是一步一步确立调试的过程，如果说编码确定技术点然后一个一个攻克是寻龙过程，因为龙比较大还是比较容易发现的，而调试则是一个捉虫的过程，常指代开发过程中，这二者所花的时间和过程往往在开发软件和APP（APP指一些小软件只有几个）穿插。尤其是调试部分，需要频繁进行，一个负责的开发实践往往要体现这二者。
在下面的各个技点难点中，我们会同时谈到技术点和调试手段，即龙和虫：

2，技术难点：wordpress plugin开发
-----

1，往wordpress媒体上传框新加选项卡,以下参阅了否子戈的部分代码。

```
// 在新媒体管理界面添加一个百度网盘的选项
function wp_storage_to_pcs_media_tab($tabs){
// if(!is_wp_to_pcs_active())return;
$newtab = array(‘tab_slug’ => ‘From Owncloud Fileshare’);
return array_merge($tabs,$newtab);
}
add_filter(‘media_upload_tabs’, ‘wp_storage_to_pcs_media_tab’);
// 这个地方需要增加一个中间介wp_iframe，这样就可以使用wordpress的脚本和样式
function media_upload_file_from_pcs_iframe(){
wp_iframe(‘wp_storage_to_pcs_media_tab_box’);
}
add_action(‘media_upload_tab_slug’,’media_upload_file_from_pcs_iframe’);
?>
```

2，改造owncloud files_sharing app，使之显示链接文件而不是外链共享文件。这是因为原文件中得到的结果是返回所有的共享而不是指定root share dir下的所有文件，而后者才是我们需要的，我使用的是8.0.16的相关文件，简单修改如下：

```
private static function getSharesFromFolder($params) {
$path = $params[‘path’];
$view = new \OC\Files\View(‘/’.\OCP\User::getUser().’/files’);
if(!$view->is_dir($path)) {
return new \OC_OCS_Result(null, 400, “not a directory”);
}
$content = $view->getDirectoryContent($path);
$result = array();
foreach ($content as $file) {
$result = array_merge($result, array(‘1’=>$file[‘name’]) );
}
return new \OC_OCS_Result($result);
}
```

3，调试明确rest api一次request/response过程中的数据主要是什么形式的：
-----

好像bookmark用的rest api是第一代，用的是json,而ocs api用的是owncloud api，那为什么二套可以共存呢，这是因为开源软件都是慢慢发展起来的，历史遗留中好的部分会存在很久。


注意，这里会出现不确定的复杂情况比如无限要求密码，此时记得要清空浏览器所有缓存重新粘贴完整url，调试一次就要清空一次才能保障调试结果顺利进行。

4，让owncloud ocs rest api免密码，这是因为上面的调视是可视化进行的，而owncloud ocs api是需要程序内编码验证的，而这些不能浏览器端以传递给URL的方式进行，只能通过CURL http basic auth方式进行，能传给URL的是以上几个提到的配置参数。
-----

```
function wp_storage_to_pcs_media_list_files(){
$ch = curl_init(“get_option(‘oc2wpfs_oc_server’)”./ocs/v1.php/apps/files_sharing/api/v1/shares?path=get_option(‘oc2wpfs_oc_dir’).”&subfiles=true”);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);
curl_setopt($ch, CURLOPT_USERPWD, “get_option(‘oc2wpfs_oc_user’):get_option(‘oc2wpfs_oc_password’)”);
$output = curl_exec($ch);
curl_close($ch);
echo $output;
……
}
```

得出以下基本调试视图：


好了，接下来就是把获到的API response解析为上传媒体上的文件，让editor支持选择媒体等部分了。这样插件就基本完成了，留到以后做。。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339705/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>





