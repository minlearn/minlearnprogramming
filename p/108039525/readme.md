打造小程序版本公号和自托管的公号(3):为miniblog接入markdown和增强的一物一码
-----

__本文关键字：微信一物一码，一文一码，小程序码转为普通二维码__

在《打造小程序版本公号和自托管的公号2》中我们讲到了miniblog的初级搭建和定制，这里继续：

加入markdown编辑
=====

前面说到miniblog的8个函数中，前6个函数可以整合，剩下synservice,synctoken不必整合。可以丢弃或保留（保留使用它的意义不甚大，这种同步结果和过程比较曲折，体验不好，要使用它，记得syncservice/index.js/function syncWechatPosts(isUpdate) 中应该 let data = { currentOffset: 0, maxSyncCount: 由10改为1000 } ，同时提醒用户把该函数超时时间设置大一点。  否则很难有正确结果。我120数据用了8秒。不要太频繁调用这个函数，免费额度会减少。也不要搞按时自动触发。这底下是因为那个函数内，调用公号素材库未发布文章一次只能取到20条，循环获取就是maxsynccount指定的最大结果）。丢弃的情况下,我们则需要依赖内部的发布文章机制，由于手机实在不好发文作富文本修饰或输入（除非chromebook,apple arm macbook真的广泛用起来了，人们用它办公和文字生产了），粘贴复制还是可以的，所以最好是markdown的那套。目前miniblog的那个编辑器可以发markdown前端也能通过toxml渲染,但是测试了一下，不能修改和更新文章。

首先，加入miniblog的编辑文章功能。mini-blog-d02344fe6c6b347057983edd1ef6b7450f88fbcc这个rev删掉了发文功能（审核问题删了。），去它的上个版本把文件下载回来修改相关wxml,js调用加进去。然后是把富文本编辑器定制成markdown：

```
miniprogram/pages/admin/article/article.js中:
savePost: async function (e) 函数内：
contentType: "markdown",   //这里由html改为markdown
content: res.text,  //这里由res.html改为text
getContent: function () 函数内：
resolve(res.text)

miniprogram/pages/detail/detail.js中：let content = app.towxml(postDetail.result.content, 'markdown');
```

小程序首页和文章详情也都是调用toxml的地方，实际上，小程序首页如果文章比较多下拉后返回上面很费事，你可以顺便做个添加一个回到顶端，wx.pageScrollTo({scrollTop: 0,duration: 300})。由于源码中toxml是放到src根下的，你还可以把toxml放进miniprogram/component文件夹内，这里是放小程序组件的地方（由于本地的js版本不一，编译miniprogram最好是本地进行，提取dist下生成的用。这也是前面文章提倡就地npm install编译的道理），移动后修改相关路径(component json和miniprogram/app.js中)。接下来会谈到把海报生成poster组件也放进来。

加入增强的一物一码
=====

小程序实际上它也是web原理的page app，有各种路径和调用参数，(IDE左下角可以看到)，而二维码是一种编码解码机制，的背后要么是某段文字要么是某个网址，工作原理是经过图像扫描，编码解码得出码后的结果。为app所用。微信支持三种为小程序生成二维码的API，可以为小程序本身和小程序各种页面和带参数的页面调用生成二维码。即所谓支持一物一码，这里是文章，所以是一文一码。

miniblog中有一个海报生成，里面就有二维码生成，它被放在了前端文章详情页，miniblog用的是api中的getunlimit()，可以为/page/detail/detail?id=blogid形式的带参页面生成对应小程序码，如果有100篇文章可以生成100个二维码。这种api生成的二维码是小程序样式的。较普通方块二维码有些缺陷（比如它难扫，而且这种小程序码中间一个大大的logo很占位置），最重要的问题是：它背后没有一个https打头的url：微信后台对其一物一码的绑定url，虽然它是页面但其实它是小程序页面不是https，仅限该miniblog小程序内绑定，我们需要把mini/index.js/getunlimit()换成createqrcode（较getunlimit和get，这个能得到普通二维码。get和createqrcode各有5W多的限额共10够用）:

```
async function addPostQrCode(event) {
  //let scene = 'timestamp=' + event.timestamp;
  let result = await cloud.openapi.wxacode.createQRCode({
    //scene: event.postId,
    path: 'pages/detail/detail?id=' + event.postId,
    width: 280   //结果得到的从280到1280，我这里只要最小的
  })
  .....
}
```

这样就得到了普通二维码图片（中间LOGO，底下有一个提示语扫码使用小程序的提示）和带https的微信扫码绑定机制，我们把它叫作decodeqrcodeurl。得到这个，我们可以为其重新编码生成二维码，这样再处理之后的二维码有什么用呢，比如，用we-app-qrcode之类的组件再处理，不仅可以将提示语和LOGO去掉，而且可以用草料等工具定制美化。还可以可以压缩其体积，生成base64，供同一份内容的多次外发的文章脚部链接小程序二维码使用。真正让一物一码做到极致。

在mini/index.js中，我们加入重新解码的逻辑：

```
头部加入：
const decodeImage = require('jimp').read
const qrcodeReader = require('qrcode-reader')


async function addPostQrCode(event)前加入：
//解码二维码buffer
function qrDecode(data,callback){
  decodeImage(data,function(err,image){
      if(err){
          callback(false);
          return;
      }
      let decodeQR = new qrcodeReader();
      decodeQR.callback = function(errorWhenDecodeQR, result) {
          if (errorWhenDecodeQR) {
              callback(false);
              return;
          }
          if (!result){
              callback(false);
              return;
          }else{
              callback(result.result)
          }
      };
      decodeQR.decode(image.bitmap);
  });
}

async function addPostQrCode(event)中加入：
async function addPostQrCode(event) {
....
  if (result.errCode === 0) {

    qrDecode(result.buffer,function(urlcontent){
    console.log("decode:" + urlcontent);
  });
  .....
}

在mini下npm install -save jimp和qrcode-reader可以在package.json中加入：
 "dependencies": {
    .....
    "jimp": "^0.16.0",
    "qrcode-reader": "^1.0.4",
    .....
  }

```

miniprogram方面，文章详情页的分享小程序页面本来就存在，拉取（海报中的）二维码又涉及到服务端调用返回二维码费资源，干脆不交给用户，将它做成admin page内供管理员编辑文章处用更合适。也可以达到同样由用户来生成二维码的效果。还更可控，适用。比如，可以砍掉生成整个海报其它部分仅保留二维码生成部分，然后转移相关wxml,js函数调用，component定义（包括js中的初始data定义）到admin/article/articlelist页（不要放在article/article，批量操作不现实），按钮跟那些“展示”，“专题”，“标签”小红方块放一起，当还要定制主逻辑函数，具体如下：

```
//utils/api.js中， function getNewPostsList(page, filter, orderBy) {}和function getPostsList(page, filter, isShow, orderBy, label) {}中，这二函数的_fields都加起：qrCode: true来，以便在articlelist.wxml中能：

<view class='cu-tag bg-red bg-{{item.qrCode == undefined?"red":"green"}} light' data-postqrcode="{{item.qrCode}}" data-postid="{{item._id}}" catchtap='onCreatePoster'>
  {{item.qrCode == undefined?"未生码":"已生码"}}
</view>

//然后是主函数：

onCreatePoster: async function (e) {
  let that = this;

  let postid = e.currentTarget.dataset.postid
  console.info(postid)
  let postqrcode = e.currentTarget.dataset.postqrcode
  console.info(postqrcode)
  let qrCode = await api.getReportQrCodeUrl(postqrcode);
  let qrCodeUrl = qrCode.fileList[0].tempFileURL
  console.info(qrCodeUrl)
  that.data.posterImageUrl = qrCodeUrl

  if (postqrcode == undefined || that.data.posterImageUrl == "") {
    let addReult = await api.addPostQrCode(postid)
    qrCodeUrl = addReult.result[0].tempFileURL
  } else {      
    that.setData({
      posterImageUrl: qrCodeUrl,
      isShowPosterModal: true    //model是模态对话框意思
    })
    return;      
  }

  let posterConfig = {
    width: 600,
    height: 600,
    backgroundColor: '#fff',
    debug: false
  }
  var blocks = []
  var texts = [];
  texts = [];
  var images = [
    {
      width: 560,
      height: 560,
      x: 20,
      y: 20,
      url: qrCodeUrl,//二维码的图
    }
  ];

  posterConfig.blocks = blocks;//海报内图片的外框
  posterConfig.texts = texts; //海报的文字
  posterConfig.images = images;

  that.setData({ posterConfig: posterConfig }, () => {
    Poster.create(true);    //生成海报图片
  });
  await that.onPullDownRefresh()

},

//你会发现列表是按createtime排的，utils/api.js中要改成timestamp,

if (orderBy == undefined || orderBy == "") {
  orderBy = "timestamp"
}

//然后utils/api.js中function getNewPostsList的检索条件和index.js中的检签逻辑(记得把总体文字缩短否则idx6之后显不出来，我是把已xxx全删了仅保留未xxx)也加起来。

if (filter.qrcoded == 1) {
  where.qrCode = _.nin(["", 0, undefined])
}

if (filter.qrcoded == 2) {
  where.qrCode = _.in(["", 0, undefined])
}

//如果你还想把生成的一物一码自动欠入各文章详情页内。，代替原来的onpostcreater()位置，那么可以
showQrcode2: async function (e) {
  let that = this;
  let qrCode = await api.getReportQrCodeUrl(that.data.post.qrCode);
  let qrCodeUrl = qrCode.fileList[0].tempFileURL
  wx.previewImage({
    urls:  [qrCodeUrl],
    current: "一文一码"
  })
},
```

----

整个过程依然都是在IDE模拟器中不断的修改，调试，再调试（提一下，我用的IDE是deepin20商店中https://github.com/cytle/wechat_web_devtools编译的，这个IDE腾讯官方没有列出为支持项）。要注意服务端的在函数后台看日志，IDE只能看出客户端的输出，由于上面是加入到服务端函数后端的，因此调试结果console.log("decode:" + urlcontent);在IDE端是看不到的（除非你打开IDE中的函数调用日志）。突然发现小程序用于实践很微粒化。一天一个微程序，像jupyter一样每天练习，持之以恒，可以积累下大量的语言和项目开发经验。

-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/108039525/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




