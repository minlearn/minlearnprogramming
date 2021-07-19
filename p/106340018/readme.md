在群晖docker上构建私有云IDE和devops构建链
=====

__本文关键字：云IDE。docker as cloud ide，在群晖上安装docker gitlab,gitlab ci for docker__

在以前的文章中我们说到docker是一种，集云虚拟化，装机，开发机，user modeos,langvm,app runtime为一体的东西。(或者不严格地说，仅仅可以当所有这些东西来用)。而这，其实就是我们一直想集成达到的DISKBIOS方案。

在《docker as engitor及云构建devops选型》一文中我们还说到，docker可用于组建私有devops，模拟engitor的效果，在那文的文尾我们提到云IDE，git是这个云IDE收集工程源码文件的云化过程（git同时是实现为客户端也是服务端一体的，所以它是云IDE客户端负责收集工程文件，在服务端它返回给下一级CI过程），那么集成了CI的git服务器实现品(如gitlab version8+版本以上自带CI模块)，就是云IDE中定义如何自动化构建这个工程的过程。

可见在云开发中，docker生态是一个非常流行和强大的东西，云IDE的先进理念实际就是devops(实际上，像gitlab这样的实现品已有cloud ide这样的插件)。

下面我们就来讨论如何用docker的gitlab ci模拟云IDE中的自动化构建链效果。我们的环境是群晖docker上。用外置postgresql实例的方法，我们最终要实现的结果，就是实现gitlab以docker为executor的CI链，可以实现面向docker为开发机的构建，发布的自动化过程。VS 托管在远处的devops服务器，有一个私有devops的好处是，我们可以在本地即时快捷地观看和控制程序构建的过程。

群晖docker上搭建gitlab
-----

跟《docker上安装ellie》一样，这同样是个复杂的过程，gitlab是ruby的，gitlab cl是nodejs的，跟ellie docker一样是涉及到多语言环境的。我们复用ellie中的postgresql9.5镜像。

我用的是2019.2.2号左右dockerhub上sameersbn/gitlab的GitLab Community Edition 11.7.0（在他的镜像中，7.4.3之前版本，镜像里包含所有组件，7.4.3版本镜像里只包含核心组件：nginx、sshd、ruby on rails、sidekiq），不要下载官方的gitlab/gitlab-ce，那个镜像里内内置了postgresql数据库。启动时占用内存过大。而且不正交。由于这个镜像很大，外网线路下载起来很费事，容易中断，我们可以利用上shadowsocks的方法，在windows上开一个允许局域网连接。然后在群晖控制面板->你当前使用的网络界面中配置一个代理服务器。之后下载就会快多了，下载完全后，同时下载redis:latest，这样postgresql9.5,redis,gitlab镜像都有了。先启动postgresql和redis的实例。

再开启一个sameersbn/gitlab的实例，link到postgresql9.5:别名postgresql，redis:别名redisio，80容器端口映到8001，因为主机群晖占用了80。增加几个环境变量，

```
GITLAB_SECRETS_DB_KEY_BASE=随便写
GITLAB_SECRETS_SECRET_KEY_BASE=随便写
GITLAB_SECRETS_OTP_KEY_BASE=随便写
```

启动，gitlab会自动连接postgresql，发现容器退出，查看日志后发现，FATAL: role "root" does not exist,数据库中没有root用户，这是因为gitlab实例对postgresql实例的数据库有root检查，及其它一些硬性配置上的要求。下面这些做：在群晖的web版进postgresql1实例的终端机界面(点新增会自动打开一个bash终端)新建一个root用户并赋于权限。

```
su - postgres
psql
create user root with password 'password';
ALTER ROLE root WITH SUPERUSER;
此时再尝试启动应该没有上述错误了。但又退出，且提示psql: FATAL: database "gitlabhq_production" does not exist
CREATE USER gitlab WITH PASSWORD 'password';
CREATE DATABASE gitlabhq_production OWNER gitlab;
GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production TO gitlab;
\q
```

最终启动成功，发现内存维持在1G多比gitlab/gitlab-ce少很多，打开群晖ip:8001，会提示让你修改root的密码，这个root是gitlab用户的不是postgresql的。用root和这个密码登录，进群晖ip:8001/admin/。

现在可以在上面建立repo，clone的界面上显示的是localhost，你需要额外加二个启动环境参数来定制这里显示为localhost的部分,另外如果你想导出各种volumes，参照ellie关于权限的处理方法就行。

最后，然后进admin/runners查一个token，备用。

在群晖docker上安装gitlab ci for docker
-----

这里的坑有点多。

首先不要下载sameersbn/gitlab-ci-multi-runner:latest(gitlab/gitlab-runner也是multi的)，这个版本太老，启动后link到一个别名为gitlab的第一步安装的gitlab实例，sameersbn的runner是可以定义环境变量注册的

```
RUNNER_TOKEN：上面的token
CI_SERVER_URL：http://link到的gitlab别名:80到主机的转发端口/ci
RUNNER_DESCRIPTION：随便填
RUNNER_EXECUTOR：这个暂时先填shell
```

虽然方便，然而我尝试了下这种方法在上述sameersbn/gitlab-ci-multi-runner版本中根本无法使用，一直提示404,PANIC: Failed to register this runner. 404，PANIC: Failed to register this runner. Perhaps you are having network problems

我们下载同gitlab版本的gitlab/gitlab-runner:v11.7.0，启动后link到第一步安装的gitlab别名gitlab，然后进终端机用命令行方式注册runner到CI：

像上一个方法一样新建一个bash，会进入/home/gitlab_runner中，打入gitlab-runner register会提示输入六个选项的参数。依次是：

```
url：这个填http://gitlab/ci
registration-token：这个填第一步获取备用到的那个token
executor这里填docker
docker-image这里我可以按需求填alpine:3，这个有什么讲究呢？这什么选这个呢？预置的有什么用呢？其实这是构建Docker image时填写的image名称，根据项目代码语言不同，指定不同的镜像。
description随便填
tag-list这里填v1170
```

所以你看出来了，以后devops的触发主要是由其中对应到这里的tags来触发，docker ci build的原理其实就是以某docker image为虚拟机，在里面一层一层构建fs，然后叠加成最终镜像，这里的docker-img即为那个虚拟机。

所以docker image加tag的组合可以根据很多不同目的来定义多个。多用。

以上我们注册的runner是全局的。也有per工程私有的runner，上述tag为v1170的docker runner就是工程全局共享的

至于各种参数具体有什么用，等以后讲吧。那个触发文件流程定义.gitlab-ci.yml更是复杂,反正runner是建立起来了，在项目的/settings/ci_cd，CI/CD Pipelines -> Runners activated for this project，会看到已激活的runners

-----------

还有，我们可以搞个for elmlang,下回吧，这样在我们的私人服务器上就可以即时持续集成了（以达到不断向其喂给碎片化项目内容，持续集成为大应用的目的，这也许就是微服务的由来）



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340018/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



