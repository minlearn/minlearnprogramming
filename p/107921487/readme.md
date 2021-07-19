打造小程序版本公号和自托管的公号(2)：利用mini-blog将你的blog做到微信
=====

__本文关键字：合并多个cloudfuncions为一个__

在《打造小程序版本公号和自托管的公号(1)》中，我们谈到了建立blog小程序的一些理论基础和必要条件，现在，我们选择一个这类小程序的源码，来实践一下：

我们选择的是https://github.com/CavinCao/mini-blog/commit/7921a126122f4a980e1270475aeb22bb2d50c3d0，积分功能v2版本后面的一个修复版，为什么选mini blog，因为它小。该有的都有，不该有的也没有。很容易看清这类实践的基本套路和这类程序的技术手法，下载源码后打开微信开发IDE，扫码IDE左上角小程序（开发社区号）绑定微信，，用IDE导入正确路径下源码，appid填你注册过的小程序appid，因为miniblog使用的腾讯云cloudbase托管的云函数后端（当然miniblog还用了cb的数据库），在点击云开发按钮新建第一个数据库或云函数之前，它会自动帮你注册或绑定已有的一个腾讯云帐号，为了达到在已有帐号里操作的效果，请保证预先处理你绑定IDE用的微信，它背后的某个腾讯云号已有绑定该小程序作为登录方式之一，这样新建云函或数据库会自动在你已有的腾讯云的cloudbase下进行，如果你使用了自动生成的腾讯云号，那么会得到一个新帐号已绑有小程序登录但未绑定微信，需要二次登录（先微信后小程序）才能进入帐号，解决方法是在那个新帐号里绑定一次微信。

IDE打开，我们发现源码有cloudfunctions云函和miniprogram小程序，点云开发，新建一个clodubase环境包月免费自动续费，登录web版会发现用IDE建的是只读的而且与网页端申请的那个免费可以共存，。接下来就是miniblog的安装：有12个库（access_token,mini_posts,mini_comments,mini_posts_related,mini_config,mini_logs,mini_formids,mini_member,mini_sign_detail,mini_point_detail,mini_subcribute,mini_share_detail），和8个函数，安装12库比较简单，记得把权限全设为所有只读仅管理员和创造者修改，如果稍后你发现编出的小程序有各种BUG，那么可能是12库少安装了某个或某些，或漏了权限没改过来。然后在IDE里定位到正确的cb部署环境，在IDE文件浏览器下点击各个函数对应的文件夹，右击部署（包括安装node modules方式），一一给函数的版本管理->配置处添加Env:cb名的环境变量项，其中syncservice还需要提供内容同步的公号的AppId和AppSecret，adminservice还需要author:你的管理者微信的openid串（可在稍后调出小程序界面后切到/index/index/从ide的console中直接输出的信息复制得到），miniprogram只需改下utils/config.js里的env id到正确的cb名，还需要在IDE文件管理器中右击index/posts/detail打到命令行安装一个客端模块：npm install wxa-plugin-canvas --production (设置，项目设置处勾上使用npm模块)，至于源码用的各种templateid，由于是公共ID，所以不用修改。小程序前后端基本能一起工作起来了，执行一次synservices拉取一些测试文章（根据错误添加IP白名单）。

下面，我们来深度定制它：

将8个云函数合并
-----

由于部署8个云函数带来了管理上的困难，也显得零散，所以我们把它合并成一个叫mini的云函数中去，思路是先在cloudfunctions下新建mini，把各xxxservices中的config.json，package.json合并去除重复部分放到这，package-lock.json可删掉，如以下修改：

config.json(json不允许注释用时需要去掉//后的内容,下同)

```
{
  "triggers": [
    {
      "name": "myTrigger",
      "type": "timer",
      "config": "0 30 8 * * * *"
    }
  ],
  "permissions": {
    "openapi": [
      "wxacode.getUnlimited",
      "templateMessage.send",
      "templateMessage.addTemplate",
      "templateMessage.deleteTemplate",
      "templateMessage.getTemplateList",
      "templateMessage.getTemplateLibraryById",
      "templateMessage.getTemplateLibraryList",
      "subscribeMessage.send",
      "subscribeMessage.getTemplateList",

      "security.msgSecCheck"   //注意这个
    ]
  }
}
```

package.json：

```
{
  "name": "mini",
  "version": "1.0.0",
  "runtime": "Nodejs8.9",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "date-utils": "^1.2.21",
    "request-promise": "^4.2.4",
    "towxml": "^2.1.4",
    "hydrogen-js-sdk": "^2.2.0",
    "request": "^2.88.0",
    "wx-server-sdk": "^1.8.2"   //注意这里，提到了最高版
  }
}
```

然后是重点整合部分，把各个index.js整成/mini下总的index.js（login/index.js已被如下融合，除外）:

```
//////以下从各个index.js头部整合到的全局变量
const cloud = require('wx-server-sdk')
cloud.init({ env: process.env.Env })
const rp = require('request-promise');
const dateUtils = require('date-utils')
const db = cloud.database()
const _ = db.command
const RELEASE_LOG_KEY = 'releaseLogKey'
const APPLY_TEMPLATE_ID = 'DI_AuJDmFXnNuME1vpX_hY2yw1pR6kFXPZ7ZAQ0uLOY'
//收到评论通知
const template = 'cwYd6eGpQ8y7xcVsYWuTSC-FAsAyv5KOAVGvjJIdI9Q'
const Towxml = require('towxml');
const towxml = new Towxml();
const COMMENT_TEMPLATE_ID = 'BxVtrR681icGxgVJOfJ8xdze6TsZiXdSmmUUXnd_9Zg'

//////以下整合的main,用了自建的route分发，实际上就是折分二个return：event,openid

exports.main = (event, context) => {
  console.log(event)
  console.log(context)

  //注释掉
  //return {
  //  event,
  //  openid: wxContext.OPENID,
  //  appid: wxContext.APPID,
  //  unionid: wxContext.UNIONID,
  //}

  //这句重要
  return router(event)

}

async function router(event) {

  //从main中移到这里
  const wxContext = cloud.getWXContext()

  //不需要了
  //if (event.action !== 'checkAuthor' && event.action !== 'getLabelList' && event.action !== 'getClassifyList' && event.action !== 'getAdvertConfig') {
  //  let result = await checkAuthor(event)
  //  if (!result) {
  //    return false;
  //  }
  //}

  switch (event.action) {

    //postsservices
    case 'getPostsDetail': {
      return getPostsDetail(event)
    }
    case 'addPostComment': {
      return addPostComment(event)
    }
    case 'addPostChildComment': {
      return addPostChildComment(event)
    }
    case 'addPostCollection': {
      return addPostCollection(event)
    }
    case 'deletePostCollectionOrZan': {
      return deletePostCollectionOrZan(event)
    }
    case 'addPostZan': {
      return addPostZan(event)
    }
    case 'addPostQrCode': {
      return addPostQrCode(event)
    }
    case 'checkPostComment': {
      return checkPostComment(event)
    }

    //按葫芦造样，把memberservices,messageservices,adminservices,scheduleservices,syncservices,syncTokenservices其它case部分也弄到这

    //注意这句
    case default : return {openid: wxContext.OPENID}
  }
}

  .....
  //再接下来就是从各个原xxxservice/index.js中移过来不包括开头变量和main()的代码的部分,直接复制即可。

```

然后就可以删掉所有xxxservices文件夹了，客户端修求主要集中在utils/api.js和app.js中。将它们改为全部callfunction("mini")，完工！！


将主题集成到首页
-----

由于主题被做成了一个与文章首页和我的（后台）并列的项。不爽。将其归到首页“最新，热门，标签”后的“主题”文章部分。

index.wxml，从topics.wxml中提取到这里，介于搜索栏和文章列表之间

```
<!-- 专题列表 -->
<view class="cu-list menu card-menu margin-top" wx:for="{{classifyList}}" wx:key="idx" wx:for-index="idx" wx:for-item="item"  id="{{item._id}}" data-tname="{{item.value.classifyName}}" bindtap='openTopicPosts'>
    <view class="cu-item">
        <view class="content padding-tb-sm">
            <view>
                <text class="cuIcon-title text-orange "></text>
                {{item.value.classifyName}}
            </view>
            <view class="text-gray text-sm">
                {{item.value.classifyDesc}}
            </view>
        </view>
    </view>
</view>
```

由于展示是依靠js的，所以index.js:

```
data: {
  classifyList: [],
  posts: [],
  .....
  navItems: [{ name: '最新', index: 1 }, { name: '热门', index: 2 }, { name: '标签', index: 3 }, { name: '专题', index: 4 }],
  .....
}
.....

// 如下设置内容，避免UI块重叠
tabSelect: async function (e) {
......
  case 1，2，3: {
    that.setData({
      .....
      showHot: false,
      classifyList: [],
      showLabels: false,  //case3时为true
      posts: [],
      ......
    })
  .....
  }

  case 4: {
    that.setData({
      ......
      showHot: false,
      showLabels: false,
      classifyList: [],
      posts: [], 
      .....
    })
        
    let task = that.getClassifyList()
    await task

    break
  }
}
```

把函数getClassifyList: async function ()从/topics/topics.js中移到这，可把topiclist文件夹也移过来到index/下，修改相关路径,之后可以删掉topics文件夹了，

你还可以修改一些硬编码的东西。
```
app.json:    "navigationBarTitleText":
syncservice/index.js:          totalComments: 0,//总的点评数 totalVisits: 100,//总的访问数 totalZans: 50,//总的点赞数
pages/mine/mine.js，申请vip的提示语等
```

一些未来改进建议：

后台管理，留言可以加一个是否开启审核，“程序有一点点小异常，操作失败啦',这类消息提示可以更精确。因为一般都是数据库权限或templateid问题。

templateid位置：

```
adminservice/index.js : APPLY_TEMPLATE_ID
postservices/index.js评论：COMMENT_TEMPLATE_ID
utils/config.js：subcributeTemplateId
messageservice/index.js, template 评论被回复模板ID
scheduleservice/index.js签到提醒：SIGN_TEMPLATE_ID
mine/sign/sign.js:tempalteId
pages/mine/mine.js申请vip: tempalteId
```

----

整个过程都是在IDE模拟器中不断的修改，调试，再调试。之后就是提交体验了，提交体验版本然后微信浏览基本接近以发布后效果而IDE中的模拟器较之有一些效果体现不出来，还有，审核的时候有可能会说你动态网站不能通过很奇葩，那么以cloudbase为后端意义何在？

下文我们将用fodi分开前后端的方式来划分onemanager将它做成miniblog小程序的内容源或者给miniblog直接弄一个PC网站前端和pc网站后端写作模块。其实小程序和onemanger非常像。因为后者也是云函数为后端，只不过一个定位网站前端，一个定位小程序前端。后端都是api服务。mini blog for wechat是直接利用cloudbase sdk调用云函，天然鉴权请求，而onemanager的前端是利用普通网站提交，基于url route提交请求。fodi前后端利用更高级的ajax。这三者都有相似之处,其实cloudbase也可用于google flatter和mobile apps。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/107921487/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


