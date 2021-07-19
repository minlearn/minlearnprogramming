在tinycolinux上编译odoo8
=====

__本文关键字：在tinycolinux上源码安装odoo8,动态模式python+uswgi+nginx,精简安装odoo8模块__

在前面《发布基于openerp的erpcmsone》时，我们谈到openerp其实是一种后端erp前端CMS的东西，其网站模块部分是通用cms网站选型的技术楷模，有可视化拖拉建站支持，且可集成后端erp部分（在线聊天啊，联系表单，购物车模块,etc..），当然，谈到cms，不是说它就是前端，它反而正是属于odoo后端支持部分的，只不过其展示部分是前端技术：

什么是CMS呢？所谓CMS，隐藏在内容管理系统(CMS)之后的基本思想是分离内容的管理和设计。页面设计存储在模板里，而内容存储在数据库或独立的文件中。 当一个用户请求页面时，各部分联合生成一个标准的HTML（标准通用标记语言下的一个应用）页面。对于一个CMS，其后台admin系统就代表了它的技术全部（负责内容模型表示和前端展示）。所以我们这里说到前端就是指其生成到html的后台模块支持部分等等  --- 这跟不生成前台静态页面的前台纯动态交互网站需求不一样后者不需要html化。----- 在前面《发布mineportal,ocwp》《让oc hosting static website》中我们都谈到静态网站，它其实全称是静态生成器网站，它其实是CMS工具化了的一种形式。

而且，odoo还采用了pgsql，从Postgres 9.x开始，Postgres又添加了激动人心的NoSQL的支持,,Postgres是通过添加一个json(jsonb)数据类型来实现文档型存储的。这迎合了采用统一存储后端的设计，可以使得odoo的document模块使用分块filestor文件系统，见《发布mongopress,基于统一的分布式数据库和文件系统mongodb》同类文章。

最后，odoo采用python，要谈到语言的优异对比足于掀起大论战了，我不重复那些聚焦语言内部如何pythonic的老话题，只讲几条外部特征：

1，C系和原生程序，是基本所有现实中可见系统实现的基石，但C系不一定就是最好的，都是先用起来的实用主义的产品，而python，就是所有linux发布版事实上的脚本语言环境。

2，在语言选型上，虽然工程层面是提出越来越多的脚本语言来支持各种domain，但其实历史上还是倾向直接一门丰富langtechs语言支持库级表达的DSL，这也是为什么历史上众多语言很好地完成了某领域部分的事现实上在其它领域不好用，但还是会宣称它自己是通用脚本语言一样。比如php不被用于作非WEB开发，其它语言不常用于自然语言处或科学计算等等，python虽然也不够通用，但事实上它的应用领域最通用。

3，在语言选型上，工程上是提倡越来越多的语言，但具体到人和学习者，我们一般倾向于只学二门语言一门C系必学（C or c++），另一门应用脚本语言，且这二种语言形成one host one guest的only two选型特征，根据2中提到的二种语言要面向DSL包纳越来越多这些特征，lua虽然精微与C一样重正交设计易与c as hosting交互但依然需要出现c系的面向对象等CPP多范型里面的需求场景，所以除去lua,c这种较专用，重基础和偏门的，所以在应用上我们依然需要学习python和cpp这种多范型支持的，而python即是这种langtech level和liblevel都battery included语言。

python in onlytwo as guest for c series是种混合语言系统，业界已有混合语言的实作品，下面这些产品也有python界的比对物这里只是拿来作为例子：比如制造DSL支持领域逻辑+jit的terralang,比如compiled to lua的moonscript(它提出新语言免去了直接binding的需要)，还比如cython,zephir这种仅是生成C模块作为原语言模块的“混合语言”系统（它没有提出新语言）。

下面就让我们来打造tinycolinux上的lnpp appstack结构(linux+nginx+python+postgresql)，并安装odoo8，注意这里我们只精简安装odoo的必要模块和web相关模块。

编译lnpp的python+uswgi和postgresql
-----

接《为tinycolinux创建应用和lnmp-源码和toolchain》文，我们这次是编译python,除了那文中gcc中需要的tinycorelinux的tcz，我们还需要openssl-1.0.0-dev.tcz(事实上python编译不要它但是接下来pip要用到它)，解压安装它，下载python src,我选择的是Python-2.7.14rc1.tgz，解压cd到src目录我们这里是/home/tc/Python-2.7.14rc1，sudo ./configure --prefix=/usr/local/python(你可以加条 --enable-threads未来用python启动uswgi多线程支持会用到),sudo make install

cd /usr/local/python/bin,下载pip，wget --no-check-certificate https://bootstrap.pypa.io/get-pip.py ,然后sudo python get-pip.py安装pip。接下来可以安装uswgi了sudo pip install uswgi（会用到与nginx编译时一样的pcre-dev.tcz），运行uswgi，显示安装后的uswgi版本是，ctl+c退出它，下面第二部分我们会谈到以正确详细的参数运行它。

对于pgsql我下载的是postgresql-10.1.tar.gz，按处理python src的方法处理它,会要求用到readline，在sudo ./configure --prefix=/usr/local/pgsql --disable-redline中禁用。sudo make install 编译完。然后在/usr/local/pgsql中创建一个data文件夹，右击权限设置为7777 组root,用户tc[1001]。这是因为pgsql默认实际上也不允许以root方式运行。

sudo -u tc /usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data --encoding=UTF8创建默认系统数据库base，然后启动它sudo -u tc /usr/local/pgsql/bin/pg_ctl start -D /usr/local/pgsql/data(pg_ctl start也可是postgresql)，此时tc用户对于这个数据库的密码为空端口为5432， sudo -u tc /usr/local/pgsql/bin/psql base可连上管理,ctl+c退出管理,进入data目录。修改二个conf文件使得可本地用navcat等工具管理否则会出现server closed the connection unexpectedly postgresql错误,首先在postgresql.conf 打开listenadress="*",然后在pg_hba.conf中加一条: host all  all  10.0.2.2/32 trust(10.0.2.2是tinycolinux slirp模式下的host windows地址，你也可以改成0.0.0.0)。

为什么加--encoding呢。因为不这样做稍后在安装完odoo在base中建立odoo数据库时会提示：new encoding (UTF8) is incompatible with the encoding of the template database (SQL_ASCII)

在lnpp中安装精简odoo，python模块和配置uswgi和nginx参数
-----

我们先安装odoo再来处理python,这样运行它时可以逐个通过pip安装缺少的python模块，将odoo8释放到/usr/local/nginx/html,精简/usr/local/nginx/html/odoo/addons安装的所有模块，仅保留以下：


```
account
account_voucher
analytic
auth_crypt
auth_signup
base_action_rule
base_import
base_setup
board
bus
calendar
contacts
decimal_precision
document
edi
email_template
fetchmail
gamification
google_account
google_drive
im_chat
im_livechat
knowledge
mail
marketing
note
pad
pad_project
payment
payment_paypal
payment_transfer
procurement
product
project
report
resource
sale
sales_team
share
web
website
website_blog
website_forum
website_forum_doc
website_livechat
website_mail
website_partner
website_payment
website_report
website_sale
web_calendar
web_diagram
web_gantt
web_graph
web_kanban
web_kanban_gauge
web_kanban_sparkline
web_tests
web_view_editor
```

下面我们来联合配置启动uwsgi和python,nginx，我们还希望像lnmp一样，分别独立启动nginx,mysql和php-cgi(它就相当于python中的uwsgi)，先启动uswgi：

/usr/local/python/bin/uwsgi --socket :8000 --pythonpath /usr/local/nginx/html/odoo --wsgi-file /usr/local/nginx/html/odoo/openerp-wsgi.py

实际上它也有很多变体和缩略形式(你可以参照网上建立一个小例子代替openerp-wsgi.py中的内容来分别测试)：

--socket=:8000 --master --uid=tc --gid=root --wsgi-file /usr/local/nginx/html/odoo/openerp-wsgi.py --daemonize=/usr/local/python/bin/uwsgi.log

--socket=:8000 --chdir=/usr/local/nginx/html/odoo --wsgi-file openerp-wsgi.py
 (以上chdir也可用pythonpath代替，此pythonpath非python里面的应用模块寻找意义上的pythonmoudlepath)

--manage-script-name --mount /yourapplication=myapp:app

-s :8000 -w uwsgi-server:application -d somelogfile

（以上参数都可写进一个ini，然后以uswgi指定ini的方式进行，但上面我们倾向于不使用uwsgi+ini文件的方式）

可以看到上面总有静态配置的东西，要么地址要么模块名要么类名，而lnmp中的php-cgi后面的参数是不与任何静态地址挂钩的，它就是一个全局服务器将语言服务转化成cgi或uwsgi，所以我们得改动一下，这个改动叫“uswgi的动态模式”：

/usr/local/python/bin/uwsgi --socket=:8000 --master --daemonize=/usr/local/python/bin/uwsgi.log

nginx下正确配置以配合来自上面uwsgi的“动态模式”(可以看出与静态模式下配置条目的相对应性)：

```
include uwsgi_params;
uwsgi_param UWSGI_CHDIR /usr/local/nginx/html/odoo;
uwsgi_param UWSGI_MODULE uwsgi-server; (不需要.py)
uwsgi_param UWSGI_CALLABLE application;
uwsgi_pass 127.0.0.1:8000;
```

修改/usr/local/nginx/html/odoo下的swgi-openerp.py对应于下面的一些条目，(它相当于同cd目录下./openerp-server -c ./openerp-server.conf，openerp-server.conf中的内容即类似下面修改的得到的配置文件）：

```
db_host = 127.0.0.1
db_port = 5432
db_user = tc
db_password = 
pg_path = /usr/local/pgsql/bin
addons_path = /usr/local/nginx/html/odoo/addons,/usr/local/pgsql/data/addons/8.0 (不设置这个，会导致 http://xxx:/web/static.... full.css 404)
data_dir = /usr/local/pgsql/data
```

确定python所须模块在最后进行，注释掉uwsgi启动时的daemonize项，查看启动后的输出，并一一sudo pip install 模块名安装，其中pillow和pychart特殊处理如下：

```
.......
sudo pip install Pillow==3.4.2 (不安装这个版本会出现cant create space错误)
sudo pip install http://archive.ubuntu.com/ubuntu/pool/universe/p/python-pychart/python-pychart_1.39.orig.tar.gz
..... 
```

上述lnpp全部成功启动会自动在/usr/local/pgsql/data下生成filestor,addons/8.0等目录，访问localhost，成功！！

总结起来，我们需要在tinycolinux启动时在/opt/bootlocal.sh中以如下命令分别启动nginx,uswgi和

```
/usr/local/nginx/sbin/nginx
/usr/local/python/bin/uwsgi --socket=:8000 --master --uid=tc --gid=root --daemonize=/usr/local/python/bin/uwsgi.log 
sudo -u tc /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data
```

好了，进入odoo怎么应用和操作又是一种境地了，odoo所有的操作中，数据都有固定的视图，一条博文和一个文件是一样的，一个产品和一个电脑是一样的，faint，我记得怎么进管理模式，忘了。

------------------

我特别喜欢python生态下的jupyter，如果说odoo的cms是一种带前端展示渲染后端控制渲染的综合应用逻辑体，且其可视化拖拉是一种visual editor and demo instant showing system，那么它其实可以结合jupyter，让jupyter直接支持这二种机制。见我的《发布engitor，一个paasone》。但其实paasone我更喜欢将其改成appstackx。一个同时带托管和编辑性质的传统简装paas(不喜欢sandstorm那种，它所处的抽象层是多余的不需要paas这样一个东西只需要appstackx)

下一篇或许是《oc上集成wordpress as cms app》和《oc上利用打造codesnippter note》etc，相比php v1版的mineportal，这是py v2版的mineportal了。


-----


(此处不设回复，扫码到微信参与留言，或直接点击到原文)

![](/p/106339380/qrcode.png)

<!-- Markdeep: -->
<meta charset="utf-8">
<link rel="stylesheet" href="../../res/aloha.css?">

<script src="../../res/markdeep.min.js" charset="utf-8"></script>




