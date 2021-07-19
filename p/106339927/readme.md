demoasengine,postasapp–paasone于软件开发运营的创新：所有人共享一app code和demobase
=====

__本文关键字：利用nginx和jupyter打造开发发布运营教育一体的多语言paas,内容创作工具CCT，多人协作平台UGC，demo as engine,post as app,云语言系统，云开发社区__

在《发布engitor》中我们说到，基于jupyter的engitor使语言系统云化，通俗来说,它是一个带IDE的（这个特点不可略）语言系统和ternoda等web服务件紧密结合一体的东西，其面向语言后端kernel自带notebook客户端as client的cs/bs特点可使jupyter应用直接从架构上WEB化,与此同时，其作为云语言系统IDE可按受用户输入并instantly show demo effect的CCT特点，使该应用开发上的东西接上了用户UGC，本身可作为demolet engine可用于UGC运营。

在《发布enginx》和《发布gbc》二文我们又谈到，基于openresty的enginx有强大的IO流控和定制胶合能力，不但可用于web集群组件的聚合搭webstack也可用于普通服务器搭任何自定义srvstack，且搭配一门脚本，enginx可以将其发展成这门语言为后端的领域逻辑服务器且纯粹脚本组件化的方式进行，再结合openresty中lua可去专门服务器化的能力，enginx可促成srvless的直接语言后端的干净srvstack系统，同时跨game/web appmodel and mixed problem domain生态一体化。

好了，恶补到此，打住！ 以上这二个产品中，对于语言系统作为后端和其它服务器作为前端的处理特点是同样鲜明的，这就开始有了技术整合这二者的基础和形成一套baas->paas容器的意味，那么，整合这二者 - 可能其一种技术实现方向就如同“走enginx的jupyter engitor”之类的东西吧（比如，让enginx代替其中的ternodo，让enginx中的语言系统engitor kernel化，加一些容器方面用nginx实现域名流转的事）会有什么特效呢？我可以告诉你 ------ 它除了有二者固有的前述优点，甚至还可以使任意应用和架构达成“一栈”结构，因为基本上engitor可以做到无客户端，而enginx可以做到无服务端，无不是真正没有，而是可免或可整合的意思，所以如果说bs/cs是二栈，那么去二者的过程结合可以促成一个一栈的东西，这个“一栈”架构我将其总结为paasone架构，这个架构简直是万金油：

免网络协议，免客户端开发，免重造轮子因为我可以复用mods，免API因为自带编辑器，免装调试器编码器因为带云语言后端，免UGC编辑器，免容器

而驱动这一切的，就是engitor+enginx支持的demo as engine,post as app特色这几个字，下面试从开发、部署，运营，实践教育工程角度来说一下这个集成化的优点，先说engitor+enginx之于开发部署方面：

app lvl一栈体系：让任何程序变身可开发引擎demo as engine,让任何程序自带CCT工具post as app
-----

为什么说这二者的结合是BS、CS双栈变一栈，因为开发上传统需要面向二端独立开发现在真正变成了一端（没有了服务端或没有了客户端开发需求，paasone集成了它们为一套从langsys到applvl的demobase和codebase,以前的开发第一次真正意义上地进入了具体应用as engine的境界，发贴即是开发应用）：

详细来说，客户端方面，engitor的ipynb小程序直接识别为网页或qtconsole结果，不需要开发网页或专门的客户端UI，一种语言通吃服务端客户端，这个跟JS用于服务端与客户端开发不同，虽然是一种语言那至少还存在二个端全栈，paasone中只有一个端多种语言一栈（为jupyter集成多种语言kernel可以带来多种开发），而服务端方面，paasone中直接有了appstack as srv，和nginx,免容器，脚本组件服务器系统。开发上复用即可。搭配beanstalk和pbc这样的东西可以定制协议和实现交互语义化。以脚本语言解释器为worker的paas,baas，这一切，比传统多语言paas需要自己定制成bs/cs,而paasone天然转化其为uni appmodel成applvl 级可用的engine，比docker更紧密langsys，更合理，更轻量。始终要记住的是(这一切都是发生在applvl和demolvl层级的不要涉及到语言，容器那些基础PAAS)：

故，
1,在开发方面,paasone允许用户：不必为一种appmodel发明特定的客户端，engitor中的qtconsole使得一切变成app broswer
2,在部署方面的集成化优点：不必为一种appmodel发明特定的独立服务器，这是因为enginx是uni appmodel container,直接免专门服务器，且可以混用web,game同生态结构..
这种统一化的微cs/bs架构跨domain的paasone选型架构，可以做到免容器，比docker之类的更轻更清希化语言和直接化应用的概念。
再总结一次：切记！这二个特点，使得任何程序本身就是“微APP引擎”和CCT工具。

免运营接入，比微信小程序乱入更合理:让任何程序直接变身UGC平台
-----

开发上的东西，一旦它有UGC，，就接上了运营啊。。平常我们都是线下开发，而engitor允许线上直接coding，comitting，在线即运营。这就涉及到了用户和用户再开发扩展。微信小程序这样的方案是以用户和社区为基础，建成开发，facebook这样的也是如此，它们都没有明朗化一个“demolet engine”的概念，应用容器可以是OS级的服务，也可以是开发级的应用服务器，也可以是demolet engine。

而engitor+enginx显式化了demolet engine,微信小程序明显没有以demolet engine为基础的PAAS，显得更合理。

用于工程，和实践和教育
-----

postasapp的微架构为运营准备了条件，对于实践和教育也是极佳的，这就是jupyter最初作为教育品出现的最初作用，paasone发展了它，在面向新手的亲善度方面：

1，无须客户端开发，其有UGC，天然自带作为engitor tools用。2，丰富的服务端开发mods，其可作为post as app 3，uniform appmodel and domain 4，只需面对一套语言和纯粹脚本组件可复用环境。

一句话，微APP架构，更容易促成实践和用户贡献。

所有编程要涉及到的角色的一体化选型：共享同样的1ddemobase选型
-----

整合enginx+engitor成paasone，我们最终要达到的目的是联结起一个应用所有生态的参与者，实现人员，开发人员，应用开发人员，管理者，用户UGC扩展人员，最终用户，这些当中所有的人员，表现为，进入一个应用后：我调出客户端，就可以云开发去试用这款软件。

---------------------

(这个paasone，未来可能将engitor+enginx结合体具现化并用它命名），这应该是1dddemobase基础建设方面最大的boss product了。对应用minlearnprogramming,micropractise,这应该是microapplet demobase了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339927/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



