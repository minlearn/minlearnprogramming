在群晖docker上装elmlang可视调试编码器ellie
=====

__本文关键字：elmlang live editor,docker要注意的地方。__

在前面发布《elmlang时》我们谈到elmlang的函数FRP和可视调试特征，使得为其装配一个live ide变得可能，elmlang提供的插件，已经使其它能很轻松地接入市面上几大IDE，如本地我们有atom，vscode这样的东西，在业界是推崇用vim的，他命令区和编辑区合一的ui方案使之成为通用ide，那么在远程呢，越来越流行的还有很多web IDE，elmlang for webapp的特性使得其天然就与web ide相生相融，与我的想法颇为迎合的是，elmlang的官方发布了一个ellie:el-li-e，elmlang live editor的意思，它模拟了atom这样的本地编辑器方案，该项目托管在https://github.com/ellie-app/ellie。

下面介绍如何将其安装到docker下。其实上述github repo已有docker支持了，且同时提供了for development和for production的二套方案，然而我测试时发现这二套直接利用生成的image和是存在很多问题的，container最终也运行不起来，所以我自己测试修正了一套。

我选用的测试环境是群晖下vmm出来的纯净ubuntu-16.04.5，安装好docker-ce和docker-compose后。git代码（我git pull到的是2018.8.22左右的cd242bea9114bf4b835cefeb228c77233a88ac07）。

>基本上ellie源码就是混合erlang->elixir，nodejs->elmlang，haskell-elmlang五种语言组建出来的：
elixir与nodejs都是语言，分别执行exs与js，其应用以语言库的源码形式发布。elixir又作为erlang的一个库与可执行服务正如elmlang是nodejs的一个库与可执行服务一样，erlang也是源码形式发布的，所以erlang->elixir是语言源码套源码形式发布的。
可nodejs->elmlang不一样，虽然elmlang本身以haskell开发，但是elmlang是以haskell compiled binary形式整合在nodejs生态中的，所以ellie中，nodejs是源码套二进制语言。可elmlang本身的库与应用又是源码发布的。
所以整个ellie源码的语言套语言架构中，源码形式逻辑发布的共有nodejs和elixir和elmlang，其中elmlang负责自身的执行，整个ellie app层次，nodejs源码是后端，负责elmlang代码的执行结果反馈(webpack框架)，而elixir负责的是前端（phoenix框架），负责你打开ellie时的那个界面，总之很绕。。。
所以它们被做进ellie这个docker编排逻辑中时，需要安排好几种语言的运行时和库支持 -- 在development版本的docker中可以看到清楚的逻辑，前后端各维持在一套dockerfile build中独立生成image和不同的entrypoint run中运行，而在prod中前后端整合到了elixir image下,它们最大的区别是，dev环境下的webpack需要附加express 8080持续运行(npm run watch)，而prod模式下，一次webpack build就行了(npm run build)，不要持续运行。

好了，在针对prod的dockerfile和docker-compose.yml作修改之前，先改几个源码中的文件：

配置文件config/prod.exs中的config :ellie, Ellie.Repo段
-----

在adpter条目下增加：

```
  username: "postgres",
  password: "postgres",
  database: "ellie",
  hostname: "database",
  port: 5432,
  ssl: false,
```

以上是ellie container实例启动时连接postgresql实例的配置。

database是数据库所在主机的主机名，docker-compose.yml中数据库 postgresql9.5对应container的ID，一般是database，对于那个ssl，如果不加ssl，会在运行时出现ssl not avaliable

config :logger段也换成这个：config :logger, :console, format: "[$level] $message\n"，否则接下来被一个run.sh包含的phx.server控制台不出任何信息。

assets/package-lock.json中，找到natives，升级一下其版本
-----

"natives": { "version": "1.1.6", "resolved": "https://registry.npmjs.org/natives/-/natives-1.1.6.tgz" },

以上是为了在防止nodejs在编译deps时出现natives有关的错误。

dockerfile中：
-----

DEPS下加一段安装postgresql-client:

```
# Install postgres-client
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ stretch-pgdg main" >> /etc/apt/sources.list.d/pgdg.list \
    && wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | apt-key add - \
    && apt-get update \
    && apt-get -yq --no-install-recommends install inotify-tools postgresql-client-9.5
```

然后就是dockerfile主体部分了：

```
# Download Elm platform binaries
RUN mkdir -p /tmp/elm_bin/0.18.0 && mkdir -p /tmp/elm_bin/0.19.0 \
    # goon executable for Procelain Elixir library, to run executables in Elixir processes
    && wget -q https://github.com/alco/goon/releases/download/v1.1.1/goon_linux_386.tar.gz -O /tmp/goon.tar.gz \
    && tar -xvC /tmp/elm_bin -f /tmp/goon.tar.gz \
    && chmod +x /tmp/elm_bin/goon \
    && rm /tmp/goon.tar.gz \
    # Elm Platform 0.18
    && wget -q https://github.com/elm-lang/elm-platform/releases/download/0.18.0-exp/elm-platform-linux-64bit.tar.gz -O /tmp/platform-0.18.0.tar.gz \
    && tar -xvC /tmp/elm_bin/0.18.0 -f /tmp/platform-0.18.0.tar.gz \
    && rm /tmp/platform-0.18.0.tar.gz \
    # Elm Format 0.18
    && wget -q https://github.com/avh4/elm-format/releases/download/0.7.0-exp/elm-format-0.18-0.7.0-exp-linux-x64.tgz -O /tmp/format-0.18.0.tar.gz \
    && tar -xvC /tmp/elm_bin/0.18.0 -f /tmp/format-0.18.0.tar.gz \
    && rm /tmp/format-0.18.0.tar.gz \
    && chmod +x /tmp/elm_bin/0.18.0/* \
    # Elm Platform 0.19
    && wget -q https://github.com/elm/compiler/releases/download/0.19.0/binaries-for-linux.tar.gz -O /tmp/platform-0.19.0.tar.gz \
    && tar -xvC /tmp/elm_bin/0.19.0 -f /tmp/platform-0.19.0.tar.gz \
    && rm /tmp/platform-0.19.0.tar.gz \
    && chmod +x /tmp/elm_bin/0.19.0/* \
    # Elm Format 0.19
    && wget -q https://github.com/avh4/elm-format/releases/download/0.8.0-rc3/elm-format-0.19-0.8.0-rc3-linux-x64.tgz -O /tmp/format-0.19.0.tar.gz \
    && tar -xvC /tmp/elm_bin/0.19.0 -f /tmp/format-0.19.0.tar.gz \
    && rm /tmp/format-0.19.0.tar.gz \
    && chmod +x /tmp/elm_bin/0.19.0/* \
    # 以上都是准备elmlang的binaries到tmp下的原逻辑，，以下准备整个app执行环境,命名为tmp2是为了将这二步骤以对应的方式列出。
    # 这里的tmp2，其实对应原版的dockerfile中是 add . /app，只是原版的构建出来在单机跑起来没事，在迁移安装到别的docker主机上跑起来，会提示找不到文件(定位不到正确的app顶层。所以deps.get时会找不到package.json等,entrypoint也找不到run.sh)。你多构建几次原版dockerfile与这里对比就知道了。
    # 你可能已经注意到这条很长的RUN，它将所有关于生成app的逻辑都维持在一个RUN中，否则就超了docker构建时的分层文件系统了，会导致不意料的事情发生。猜测原版 add . /app 就是没有维持在同一个文件系统中。docker-compose.yml中的volume也会不能生效。
    && git clone https://github.com/minlearn/ellie-corrected /tmp2 \
    && mkdir -p /tmp2/priv/bin \
    && cp -r /tmp/elm_bin/* /tmp2/priv/bin \
    && mkdir -p /tmp2/priv/elm_home \
    # 安装elixir相关的所有扩展并生成项目的数据库文件
    && cd /tmp2 \
    && mix deps.get \
    && mix compile \
    && mix do loadpaths, absinthe.schema.json /tmp2/priv/graphql/schema.json \
    ## 安装nodejs相关的所有扩展，并生成项目的webpack静态文件
    && cd /tmp2/assets \
    && npm install \
    && npm run graphql \
    && npm run build
```

至此，生成构建了所有项目运行时的资源。

已经差不多可以运行了。准备ENV预定义的参数，docker run时会欠入到实例：
-----

```
ENV MIX_ENV=prod \
    NODE_ENV=production \
    PORT=4000 \
    ELM_HOME=/tmp2/priv/elm_home \
  SECRET_KEY_BASE="+ODF8PyQMpBDb5mxA117MqkLne/bGi0PZoTl5uIHAzck2hDAJ8uGJPzark0Aolyi"
```

定义运行
-----

```
# WORKDIR的主要作用就是定义接下来所有指令，尤其是entrypoint等的路径
WORKDIR /tmp2
EXPOSE 4000
RUN chmod +x /tmp2/run.sh
ENTRYPOINT ["/tmp2/run.sh"]
```

这个run.sh是分离postgresql所在容器和ellie所在容器的entrypoint，所有连接数据库初始化的工作都要在这里完成，因为它继承了ENV关于prod的预埋参数所以运行时不会出错，否则比如在非docker构建的情况下，你把mix phx.server单独在命令行中执行，会出现如下错误：(EXIT) no process: the process is not alive or there's no process currently associated with the given name。你就需要在run.sh中export所有这些参数，这也是docker的联合文件系统在编译（dockerfile）/运行(run.sh)不同阶段需要做到逻辑同步的要求。

run.sh的内容（它是git repos中要新增的一个文件，需提交到新git repos中）：
-----

```
#! /usr/bin/env bash
set -e
cd /tmp2
until PGPASSWORD=postgres psql -h "database" -U "postgres" -c '\q'; do
  >&2 echo "Postgres is unavailable - sleeping"
  sleep 5
done
mix ecto.create
mix ecto.migrate
mix phx.server
```

最后,docker-compose.yml也一目了然了。
-----

```
ellie:
  image: minlearn/ellie-corrected
  links:
  - database:database
  ports:
  - 4000:4000
  restart: always
  environment:
  - SERVER_HOST=52.81.25.39
database:
  image: postgres:9.5
  environment:
  - POSTGRES_PASSWORD=postgres
  restart: always
```

minlearn/ellie-corrected是我在dockerhub上编译正确的ellie，实际上，上面的ellie的volumes同样是没有起作用的。留给其它人解决吧（这就是分层文件系统给人理解上带来的极大不便）。反正项目部署到任何支持docker的机器都可以启动并进入ellie所在IP:4000的界面了。

假设上面的没加SERVER_HOST,进去你会发现ip:4000/new显示ellie的动画，但一直hangout，控制台显示，[error] Could not check origin for Phoenix.Socket transport.

这就需要设置SERVER_HOST=ip变量了(这个ip是你部署ellie所在机器的外网IP或被访问IP：4000所在的IP)，这个变量不能放在dockerfile中，也不能放在run.sh中(因为这二个文件要做进docker image中的，而你无法预知要将这个docker image放哪个IP的主机上)，故要放在docker-compose.yml中ellie段下在实际开启ellie container时指定，比如我部署运行时的IP是52.81.25.39。

--------

其实docker就是一个通用的应用和OS的虚拟容器，它可以同时虚拟出我在《DISKBIOS》系列设想中用openvz虚拟出的同时运行的，却又可应用可OS的通用虚拟环境。只是它使用的aus联合文件系统我一直都不太喜欢，因为会带来污染问题和以上说到的编排dockerfile时的理解不便，突然想到联合文件系统会不会是客户端的安卓应用缓存清理的技术，其存储中，系统/应用双清的技术会不会也与它有关，就不那么讨厌了。。

有了ellie和公网盒子的群晖，你就有了一个超级方便的js学习环境，软硬环境的超级联系，使组成一个高效的生产工具变得可能，或许你还需要一个小终端，比如，《利用7寸小本打造programming pad 的 segment box》？

关注我，关注“shaolonglee公号”



-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106340036/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>



