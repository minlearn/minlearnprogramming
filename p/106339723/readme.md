让owncloud hosting static web site
=====

__本文关键字：在owncloud存储中做站，owncloud static website hosting, hosting website in owncloud,owncloud www service,mailinabox static website hosting强化,netdisk netstorage based blog system，netdisk based static website hosting and syncing__

在发布《mineportal》时我们谈到不使用静态网站的理由：它们不具有动态性。不能交互。不可编程扩展。它们只是一个从线下生成然后将结果同步/发布到线上的工具。所以静态网站，并不能真正代表现代意义上的“通用网站程序需求”，它们仅能负责展示，其它交互上的功能或扩展，需要靠那些传统网站意义上的东西外挂进来。

所以那篇文章中mineportal方案1中我们用到了oc+wp。因为我们终究需要的是“网站程序”，可是话说回来，虽然static web site不具备动态性，可是如果一个网站需要交互和动态场景的地方，比如评论，后台编辑，网页生成等处，如果这些都可以被动态程序解决，那么这时这些动态程序（后端如oc）+前端静态网站展示—–由它们组成的整体其实也满足可交互被扩展，也可以被称为“网站程序”成为我们的典型需求。而它的好处：网站可以打包打走，天然静态化是显然易见的，在以上提到的二篇文章中都不断被提到过。

在《发布mineportal2》中我们谈到可在mailinabox的oc中hosting static website，甚至谈到利用比如ownnote就可以作为编辑工具。那么不防更进一步，我们下面我们来谈在maininabox中一步一步实现它：

记得事先在oc中开启ownnote(此ownnote非qownnote，也不是qownnote对应的官方原notes app)，并做好“同时保存到sql和文件夹”设置，并做好与nginx的指向配置。

第一步：设置nginx
-----

我们首先要解决的问题是，owncloud ownnote默认生成htm，我们只能以http://xxx.com/*.htm的方式访问静态网站，以下设置能让nginx像wordpress一像把.htm的静态地址重写为/post/的形式：

```
add_header Content-Type: ‘text/html; charset=utf-8’;
location / { try_files $uri $uri/ @htmext; }
location ~ \.htm$ { try_files $uri =404; }
location @htmext { rewrite ^(.*)/$ $1.htm last; }
```

上面的add_header content-type不能省，因为oc ownnote默认保存的htm是gb2313，即使在网页中添加meta charset=gb2313也不能让浏览器认中文会显示乱码。

第二步:为owncloud ownnote建立static website template支持
-----

第二个问题：ownnote保存贴子内容直接为htm，是没有模板化效果的。我们需要集成外来模板。

我们选择了jekyll的simple jekyll主题。我们通过修改/usr/local/lib/owncloud/apps/ownnote/lib中backend.php来进行，先找到savenote()函数:

以下http://xxx.com是你nginx指向的root目录里面存有jekyll-simple的css，这里先定义了一个pre:

``` 
$precontentpart='<html><head><link rel=”stylesheet” href=”http://xxx.com/jekyll-simple/main.css”></head><body>
<div class=”page-content”>
<div class=”container”>
<div class=”three columns”>
<header class=”site-header”><h2 class=”logo”><a href=”/”>site title</a></h2><div class=”nav”><div class=”site-nav”><nav><ul class=”page-link”><li><a href=””>Home</a></li><li><a href=”/archive”>Posts</a></li><li><a href=”/about”>About</a></li><li><a href=”/feed.xml”>RSS</a></li></ul></nav></div></div></header>
</div><!– end three columns –>
<div class=”nine columns” style=”z-index:100;”>
<div class=”wrapper”>
<article class=”post” itemscope=”” itemtype=”http://schema.org/BlogPosting”>
<header class=”artilce_header”><h1 class=”artilce_title” itemprop=”name headline”>post title</h1><p class=”artilce_meta”><time datetime=”2016-07-03T00:00:00+00:00″ itemprop=”datePublished”>Jul 3, 2016</time></p></header>
<div class=”article-content” itemprop=”articleBody”><!–内容开始–>’;
```
 
再定义post(after)部分：

```
$postcontentpart='<!–内容结束–></div>
<footer class=”article-footer”><section class=”share”><a class=”share-link” href=”” onclick=”window.open(this.href, &#39;twitter-share&#39;, &#39;width=550,height=235&#39;);return false;”>Twitter</a><a class=”share-link” href=”” onclick=”window.open(this.href, &#39;facebook-share&#39;,&#39;width=580,height=296&#39;);return false;”>Facebook</a><a class=”share-link” href=”” onclick=”window.open(this.href, &#39;google-plus-share&#39;, &#39;width=490,height=530&#39;); return false;”>Google+</a></section><hr><section class=”author”><div class=”authorimage box” style=”background: url(/jekyll-simple/assets/img/Taffy.jpg)”></div><div class=”authorinfo box”><p>Author | David Lin</p><p class=”bio”>Currently a Ph.D. student in Singapore University of Technology and Design in the area of Human-Computer Interaction(HCI).</p></div></section></footer>
</article><!– end article –>
</div><!– end wrapper –>
</div><!– end nine columns –>
</div><!– end container –>
<footer class=”site-footer”><div class=”container”><div class=”footer left column one-half”><section class=”small-font”>Theme <a href=”https://github.com/wild-flame/jekyll-simple”> Simple </a> by <a href=”http://wildflame.me/”>wildflame</a>? 2016 Powered by <a href=”https://github.com/jekyll/jekyll”>jekyll</a></section></div><div class=”footer right column one-half”><section class=”small-font”><a href=”https://github.com/wild-flame”><span class=”icon icon–github”></span></a><a href=”https://twitter.com/Taffyer”><span class=”icon icon–twitter”></span></a></section></div></div></footer>
</div><!– end page-content –>
</body></html>’;
```
 
一起写入，上面的内容开始内容结束中的内容是指的ownnote借助编辑器中插入到生成最终htm的文章内容：

\OC\Files\Filesystem::file_put_contents($tmpfile, $precontentpart.$content.$postcontentpart);
这样新写的ownnote贴子会自动插入这些，具备模板化后的css效果。其它主题可类推定制。以上基本是一个文章页的通用html前端技术之essential in a nut了。

最后。虽然我们在这里的方案有很多hacking的痕迹，可是如果考虑进一步把上面所有这些做成oc的插件，还比如更进一步完善ownnote编辑器，使之输出直接支持模板化的效果。它就会成为高可用的产品。至于图片附件什么的，你可以参照站内文章中的《将owncloud作为wordpress的图床》之类。

网上还有一些利用oc做静态网站的方案，不过它们基本上只是纯粹把oc当同步工具用。本质上借助的还是jekyll,jade,gitpage这样的方案，只不过存储由github repo换成了owncloud空间而已。并没有使用到本文提到的ownnote等后台编辑增强方案。后者更自然，更强大。

————–

接下来我们会为mailinabox建立一个利用colinux封装的mailinabox box，称为mineportal2 box，其实不得不说把一切封装在黑盒中是好的方案，运维的一个基本素质就是不要去动黑盒中的东西，这对用户是好的而运维人员也可以避免少动黑盒内部对系统产生问题。



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339723/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>


