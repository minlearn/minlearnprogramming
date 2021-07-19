raw js下使用Electron，以webapp思路开发支持Electron小程序的移动APP(1):谈web前端的演变
=====

__本文关键字：web前端演变的历史,vue,react,pwa,spa,mpa__

在《用开发本地tcpip程序的思路开发webapp》等许多文章中我们都讲过web/webstack的原型(那文主要讲前后端分离的web开发思路)，如果说一种ui代表一种app，(我们用app特指那种需要下载/安装/运行，区别于用浏览器打开作为的那种桌面和移动app，它们一般寄宿在os上用native rendered gui库发明,体验为无延迟的UI和APP，可以调用本地的服务)，那么或许在2015年前就从来没出现过什么严肃意义上的web app ui/web app，只有page ui和网站，虽然我们谈webapp已经很多年但其实它是个新事物，为了应付日渐庞大的 Web 页面，经过优化的 JavaScript 引擎已经可以和一些编译语言的速度相提并论已经不仅是用js实现交互就行了。因此用web打造app的思路出现了，从网站到webasapp而这二种需求看似共享同样的基础，但面向它们的设计都不相同，web一开始就被设计成静态，保有鲜明的内容属性而非可编程属性（语义，routerable,可退回/可前进，页面内容可索引可爬取可seo），因此它要变身“某种用html表达的app”，类似native render ui/native app(甚至整合它们形成多端合一的app:一云多端的OS+多端合一的APP,universal webui+siteui app)。这不但要求web本身规范变(w3c)，而且要求支撑现有web运行起来的那些东西也要变，甚至变化过后，会破坏现有的web本身。

从siteui page到webapp
-----

首先是硬件和OS平台要为支持它们（而产生变化），webos有chromeos和html手机(基于 Firefox OS 的 KaiOS据称全球第三大移动os)这种，包括最近出来的fuschia。

然后是web本身的富网页技术(基于flash的内容创作型工具被面向程序员的js技术代替)要到位，用js写成用浏览器跑出来的ui，会跟真正的原生UI在体验上相差很大比如延迟高没有长链不能调用摄像头无法推送无法离线，这一般来自浏览器或应用的“webview”支持，（虽然说现在的浏览器有localstorage,websocket,webgl,etc.近二年又有service worker.但还是不够）因此要求增强浏览器本身的api(一般来说，正常的浏览器page运行在沙盒里没有权限调用本地api，而特制的浏览器如electron这种会面向“发明webapp”的需求保留一些特权API,，这也是它经常被apple mas禁止的原因，微信那个浏览器也是增强了私有 api的,实际上，在w3c规范之上定制浏览器这种行为就破坏了web的应用形式，故提高webapp体验的同时又不致于破坏现在的web这是一对矛盾)

> webapp的意义：
>（这个技术如果成立，web ui可以完全代替渲染原理的远程桌面形式的remote app，真正实现原生不光跨端甚至跨云的UI：我们知道，远程UI的原理大部分都是投屏技术这丝毫不原生，云没有自己的原生UI与APP技术:webapp与容器正在填补这个空隙，在前面一些文章中我们多次提到远程APP这种云原生APP的需求,曾经我追求这样的统一本地和远程的appstack《cloudwall：一种真正的mixed nativeapp与webapp的统一appstack》，但现在看来：正是桌面早已成一套的情况下原生云UI都做不到成为规范，，所以导致现在都没有“universal desktop/web app”）

最后也是最大的变化还是语言层上这种appstack本身和开发这种WEBAPP的范式上(因为前端始终是面向程序员的而不仅是内容制作)，事情发生在2015年之后，产生了将web ui作为app ui的技术潮流，并由此诞生了很多 js框架。

> 我们在前面说到框架，往往是基于对整个ola（os/langsys/applicationdomain）的修正增强补充。"js框架与浏览器/server环境"的关系跟"ola与框架"的关系一样，可以全方位去修补，因此，类似react,vue这样的东西其意义可大可小，可以是一个库也可是一个框架。虚拟DOM就是这类框架用来修补目前还跟不上webapp需要而作的框架内暂代方案。

webfront/webfront框架的前世今生
-----

我们来谈下webfront/webfrontdev的前世今生（从网站前端到webapp）。首先是自适应、响应式设计，前后端用模版实现分离，DOM API最开始是一些JS框架要面向的全部东西（这些DOMAPI就是网站ui可编程的全部，注意从这里开始，我们严格区分siteui与webappui的说法），JQ这些能很好封装ajax和dom，然后出现了更高级一点的MVC，前后端通过json数据交换实现分离《用开发本地tcpip程序的思路开发webapp》，那种monolith websiteapp的局面(服务器脚本中套前端和模板)被打破,一切用到的展示数据都是后端经由过程异步接口(AJAX/JSONP)的方法供给的，前端尽管展示。。再后来就是MVVC，到这里开始，已有webapp的初级形式，一些网站前端开始谋求progssive(注意这里是网站用的Progressive)。与移动端UI结合，又有pwa,spa,mpa(这里才是webapp化之后的前后端分离技术),

对于webapp:spa,mpa基本上，它们的分离前后端处理方法是把原来后端做的事放到了纯前端，SPA 可以在客户端提供完整的路由、页面渲染、甚至一部分数据处理； 这往往需要一个比 jQuery 时代更重的 JavaScript 框架，来实现这些原本发生在后端的逻辑(在js下的web开发就是前后端分离又融合又融合又分离全栈技术方向会越来越统一，甚至环境都可以互换，ssr可以放到服务端做，而serviceworker也可以像cloudflare一样放到服务端做，被称为大前端，比如angular的出现，使得后端的mvc,和依赖注入等技术在前端也可用，但这只是例子之一)。而且使用虚拟DOM组件来代替raw DOM。----- 目前比较主流框架：vue、react、angular等框架。AngularJS 可以构建一个单一页面应用程序（SPAs：Single Page Applications）。Vue (pronounced `/vjuː/`, like view) is a **progressive framework** for building user interfaces.我们在《elmlang：一种编码和可视化调试支持内置的语言系统》《why elmlang:最简最安全的full ola stack的终身webappdev语言选型》讲过react和fp和虚拟dom组件（这里的组件并非指脚本语言二进制构件级的概念，而是指前端和浏览器DOM层级节点的“组件”，代表前端ui的统一可管理单元，类似桌面的widget），vue就是较它们更为轻量一点的方案，也有虚拟dom和组件，Vue 除了是 React/Angular 这种“重型武器”的竞争对手外，其轻量与高性能的优点使得它同样可以作为传统多页应用开发中流行的 “jQuery/Zepto/Kissy + 模板引擎” 技术栈的完美替代。angular,react,vue都属于纯粹用组件，用程序员的思路来管理前端，虽然这样可以使得html ui更像webapp ui，但这样会破坏web的原始语义（比如seo），所以普通网站不太适合用这套框架，这都是网站管理后端，工具化应用用的。

> 大前端，这就有点像游戏引擎下开发游戏APP的思路了（这里是逻辑渲染引擎）。virtualdom就是从chrome真实渲染引擎中再建立一套编程用的场景管理和渲染实体/节点。而游戏UI/传统桌面GUI都是组件管理和组件逻辑，这个组件可以是程序上的componet也可以是可视的widget，组件就是变web html模板元素为传统的gui方式进行开发(template based ui变成了带状态的可编程ui widgets)。

PWA 方案（pwa+轻度spa）更接近于 Web 的方式，它是 Web 的增强而不是替代。PWA 一词出自 Alex Russell 的 Progressive Web Apps: Escaping Tabs Without Losing Our Soul，构成 PWA 的标准都来自 Web 技术， 它们都是浏览器提供的、向下兼容的、没有额外运行时代价的技术。 因此可以把任何现有的框架开发的 Web 页面改造成 PWA，而且与 SPA 方案不同， 没有强组件化机制，因此不需要一把重构可以逐步地迁移和改善。

Progressive 是指 PWA 的构建过程。Service Worker 是一个特殊的 Web Worker，PWA 对性能的提升主要靠 Service Worker(cloudflare workers用的类serverless云函那套)，它是在传统的 Client 和 Server 之间新增的一层。性能提升程度取决于这一层的具体策略。例如：它可以提供缓存服务,通过service worker在后台更新下载实现页面离线也可以浏览(如material design)，加速应用程序渲染，并改善用户体验。

SPA:得益于ajax，我们可以实现无跳转刷新，又多亏了浏览器的histroy机制，我们用hash的变化从而可以实现推动界面变化，纯粹依赖原生浏览器环境，而无须用到vue,react组件里那些重度方案。

更多etc..

electron开始webapp
-----

虽然web是本身跨平台跨语言的,但它并非严格跨端的，上面说到其存在多种浏览器以及内置浏览器的APP（如微信小程序以微信为宿主）的具体实现形式，这些依然需要在js框架层进行适配，web应用(包括webapp为前端的形式的整个web)都一般有明显的前后端分离做成“一云多端”结构，后端一般是传统意义上的普通网站，（你可以在后端托管具体端，比如小程序端的前端资源文件，和提供小程序框架性的服务API。当然如果是管理性后端你也完全可以可以做成webapp形式)，前端则对应PC/mobile上原生浏览器网页,pwa page(ES5起兼容的普通浏览器pwa+spa那套)，webapp spa/mpa,应用内的小程序，H5（这货也可以理解为原生浏览器），等多种形式作为UI端和客户端。----- 这些客户端共享webfront技术、w3c规范虽然不是严格意义上真正的跨端，但已经做得差不多了，而vue,react等框架用“AST,渲染引擎，bridge,runtime等抽象中间层”进一步实现了“框架和开发层跨端”，(这个跨端才是跨各种运行js的浏览器实现而不是传统意义上的跨OS) ---- 也干脆出现了一些“跨服务商的webapp引擎”，如小程序开发框架，开源小程序引擎，在中国主要是一些电商系和外卖系移动APP的前端，其本质无外乎内置了自己的浏览器，加上vue封装的小程序引擎，处理“提供一个小程序的宿主运行环境（如微信小程序的运行环境：分成渲染层和逻辑层，WXML和WXSS工作在渲染层，JS脚本工作在逻辑层），并植入自己业务的服务性API”的东西，及更多其它事情。

跟普通浏览器环境一样，微信内置浏览器一样，electron也是一种端,它提供了环境(整个v8服务,及它自己特有的一些api和扩展)，支持严肃和纯粹，追新的webapp创造，，electron可嵌入显示普通网页和webapp，你也可以用electron做pc小程序也可以做移动小程序，electron下的小程序也不是普通网页套个electron壳(比如我要做个blog客户端就是给个主页逻辑给electron)，而是网页意义上真正的app化。

跟它们不一样的是，electorn还提供了devstack(开发，语言支持，注意这并不是js而是更强大的nodejs/包结构和可开发生态)，框架方面你还需要vue等。vue和nodejs.还有vue-electron这样的封装，通过webpack生产webapp打包及流程自动化，产生支持普通浏览器js的vue引用。


-------


下文我们就准备使用raw api,使用electron提供一个小程序的宿主运行环境，配合简单后端，打造我们的移动/PC小程序:一个类《打造小程序版本公号和自托管的公号》miniblog的客户端。利用它发明多端合一的“小程序”可免去依附于微信这样的环境(微信小程序审核很费事。一点社交性质，个人版的小程序博客带评论都不允许。为免责关注程序而不是内容)。








