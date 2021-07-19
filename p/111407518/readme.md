一种用buildkit打造免registry的local cd/ci工具,打通vscodeonline与openfaas模拟cloudbase打造碎片化编程开发部署环境的设想
=====

__本文关键字：如何直接修改docker中的文件,从外部编辑dockernamespace内文件,share data between host and container?,定制镜像和容器，不经过任何registry重建/修改/commit docker镜像，Creating an image from a commited snapshot,把openfaas还原为非docker结构，可以直接在docker内编辑集成，overlay fs 读写，把你的vps做成cloudfunction环境(小程序碎化片前后端支持环境)和配备一个云IDE开发环境__

在前面《利用onemanager配合公有云做站和nas》，《利用fodi给onemanager前后端分离》我们讲到腾讯cloudbase和scf，提到它的后端管理页面有一个在线IDE。类似内置的简化版vscode online，在《在云主机上安装vscodeonline》，《panel.sh：一个nginx+docker的云函和在线IDE面板,发明你自己的paas》我们讲到自己构建这种online ide+serverless的paas面板技术，现在我们考虑联接起containerd+vscode，打通这二者模拟cloudbase碎片化编程开发部署环境的设想：

我们知道serverless技术背后是容器跑起来的。一般地，dockerd+docker-cli主要被设计成stateless and immutable only as 自动化代码部署工具(部署和生成分离,CI/CD分离)，docker维护一个image build in a place,then in another place的image pull -> image deploy -> container run的cycle（runc还有一个类似进程管理的task命令）是一个以registry为中心的pull->build->deploy循环流程。这就导致了一些问题：

> 如果不作特别说明，本文说到的docker/containerd可以混淆理解，containerd和docker都是容器运行时，docker最早是LXC（Linux Container）的二次封装发行，后来使用的是Libcontainer技术,从1.11开始进一步演化为 runC 和 containerd，其利用的也是Linux内核特性namespaces(名称空间) 、 cgroups (控制组)和AUFS(最新的overlay2)等技术，是操作系统层面的虚拟化技术,containerd重点是继承在大规模的系统中，例如kubernetes，而不是面向开发者，让开发者使用，更多的是容器运行时的概念，承载容器运行。docker偏指docker官方发布的那个运行时版本和dockercli工具。

1）docker aufs/overlayfs被设计成只读，和不变immutable环境(执行df可以看到其挂载到主机上的overlayfs mount，就像tc下可以看到tce用的loop一样，类似/run/containerd/io.containerd.runtime.v2.task/default/xxxx/rootfs)，里面的文件系统虽然是挂载到主机上的但不推荐直接读写因为它会破坏容器元数据，，要修改docker的文件，我们要借助docker提供的命令，进入具体docker容器空间按它正常的命令逻辑，去获得文件系统视图(进入Docker容器比较常见的几种做法如下：1.使用docker attach,2.使用SSH,3.使用nsenter,4.使用exec)后操作,之后docker cli(或其它定制的高级cli如k8s runctr)和dockerd管理器会保证整个应用库中的（/var/lib/docker/image/xxx）的容器元数据不受破坏。------ 但是虽然容器可用这种方法编辑内容，其结果也是不能保存的：下次重启容器，还是会重新进入到这个以pull image开始的循环，除非你重新生成镜像或提交容器即时更改。如上所述docker是部署和生成分离，docker中运行镜像和生成镜像这往往是一个脱机操作 ------ 对于提交即时更改，docker有一个docker commit命令就是干这事的（类似git commit和去主机快照技术），但除非用于特定目的(比如保存现场，比如对于某些容器每次我们只需修改/home/app下的东西并commit，这二种场景下，其实也没什么问题。)这也是不受推荐的，因为它会导致复杂的镜像:首先，如果在安装软件，编译构建，那会有大量的无关内容被添加进来，如果不小心清理，将会导致镜像及其臃肿。此外，使用docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制定镜像的人知道执行过什么命令，怎么生成的镜像，别人根本无从得知，。换言之，docker fs并不是一个flat fs结构，不能按照普通的文件系统的逻辑直接获得路径并读写。docker instant commit虽然受支持但也是不受推荐的，

2) 你也完全可以将需要变动的部分放在外部，然后mount进容器，这个问题的本质其实是打通宿主机和容器中的某些通道，类似我们使用虚拟机时在宿主和虚拟机内共享一个volume或文件夹，类似远程桌面将读写通道重定向到你本机（实际上是下载，对于远程桌面所在服务器是上传），我们将host上可写的数据被作为volume mount(或mount bind)进来到容器，将更改和变动只保存在宿主端容器动态持续CI/CD即可，但是其有权限处理问题(rootless containers...)，我们知道这二套空间的用户和文件都不一样。即我们把可以把装有app的那个/home/app独立出来放到主机。不内置入镜像。containerd层只维护一个到主机相关目录的索引,----- Docker itself does not seem to encourage using host-mounted writable volumes. 共享volume不受推荐。

> 其实docker的应用就是被设计作为执行空间，在很多文章中我们都讲到不要将其作为存放任何数据空间，一是因为它用了奇怪的fs，二是因为它与宿主机share data也是非常不直观的,见《10 ways to avoid in using docker》。。但是这是一种刚需：应用一般都是存在可读写的部分和可变存放数据的部分的，如何让docker组成一个普通的符合可读写的应用环境呢？那么程序上可以做到类似效果吗或有改进方案吗？

如果能做到或能改进，那么在openfaas+vscode上，我们将这二者结合打通,形成cloudbase后端管理页面代码编辑器编辑保存类似的效果，这样我们就可以不依赖openfaas-cli式以registry为中心的那个循环流程（请无视local registry搭建技术，因为它也是以registry为中心的），直接在online ide或web后台管理中，即时地修改代码（这种思路是在经典流程中插入一个commit过程绕过registry 作inplace build using cache，最后按需push deploy回归到经典流程-这步不是必要的）。

buildkit似乎可以完成这个工作。

docker commit方案
-----

buildkit是moby工程的一部分，它是从源码构建到容器的统一构建工具，在buildkit这类工具之前，业界都是用Docker in docker来处理的，后来转为buildkit这类专门工具，它支持多种docker运行时和docker-cli构建时，你可以把buildkit它想象成docker compose的一种上层工具，以达到docker file format无关构建和面向部署各种后端构建(处在中间层插入一个层，正符合我们上述提到的绕过registry,inplace building不push的要求)。即，它是一种统一面向容器的统一CI/CD工具，谈到CI/CD，就是你在各种CI工具和其它提供商产品后台修改git源码容器自动实时构建时那一大块黑色输出框背后用的那一套工具对应的输出界面。因为它有分布式cache构建(传统的Docker build cache only works on the same host)，并行构建，统一中间格式这些。面向多种前端和多种后端，（buildkitd daemon支持runc后端和containerd后端，By default, the OCI (runc) worker is used. You can set --oci-worker=false --containerd-worker=true to use the containerd worker.你也可以用文件定制它的参数https://github.com/moby/buildkit/blob/master/docs/buildkitd.toml.md，它是干什么用的呢。将运行时和构建时像dockerd/dock-cli一样形成cs结构，通过 http 通信的方式执行构建。）openfass build就使用了buildkit，gitlab也有一个buildkite，都是统一容器CI/CD工具链上的东西。

> moby就是我们前文《hyperkit:一个full codeable,full dev support的devops及cloud appmodel》介绍hyperkit的时候那家厂商造的，就是docker那家厂商。里面的好多产品，都是docker转正前的试验品。包括moby,和hyperkit等。在《群晖+DOCKER，一个更好的DEVOPS+WEBOS云平台及综合云OS选型》我们谈到用docker构建原生webos,cloudos，在《一种混合包管理和容器管理方案，及在tinycorelinux上安装containerd和openfaas》我们又一次提到这个观点，据说，用docker构建OS的还真不少。除了coreos还有moby。darch这种，Docker 在DockerCon 2017大会上发布了一个自己的操作系统，宣称LinuxKit，就是kernel+busybox实现的一个微缩linux系统，其中直接安装了containerd和runc服务。其他服务全部都使用容器启动。

前面说到docker偏指docker官方发布的那个运行时版本和dockercli工具和规范，所以oci偏指containerd/runc这样的工具和规范。buildkit都支持它们作为前后端。Open Container Initiative(OCI)目前有2个标准：runtime-spec以及image-spec。OCI(当前)相当于规定了容器的images和runtime的协议，只要实现了OCI的容器就可以实现其兼容性和可移植性。那么它与我们要达到的效果"达到类似web端编辑迅速ci/cd的类似目的吗？不经过registry,就地构建并保存结果给openfaas运行"有什么关系呢？因为buildkit可以以containerd image store直接为后端(containerd image store，The containerd worker needs to be used)。形如：sudo buildctl build --frontend dockerfile.v0 --local context=./ --local dockerfile=./ --output type=image,name=docker.io/minlearn/dafsdf:latest（ctr --namespace=buildkit images ls，To change the containerd namespace, you need to change worker.containerd.namespace in /etc/buildkit/buildkitd.toml）

> buildkit能达到以上效果主要是它掩盖和封装了snapshot和image操作这些中间过程，提供了一个类似docker commit的操作，可以生成含上述oci文件的镜像包，类似直接在镜像上commit，要重新生成镜像，这就不得不谈到容器的snapshot。类似云主机快照的热备原理，ctr 也支持snapshot 操作，比如ctr contained 命令行工具 prepare 命令，基于指定的已提交态快照作为"父"创建一个新快照。还有commit，containerd是用snapshootter来完成这事的，Snapshotter is one of these plugins, which is used for storing extracted layers. During pulling an image, containerd extracts layers in the image and overlays them for preparing rootfs views called “snapshots” in the snapshotter. When containerd starts a container, it queries a snapshot to snapshotter and uses it as the container’s rootfs.基本上在本地重新生成镜像这个过程应该是这样的：1,Use the diff service to create a blob in content store with the committed snapshot 2,Create image config and distribution manifest https://github.com/opencontainers/image-spec in the content store that reference that layer blob 3,Set either the image config or manifest as a target descriptor for containerd image,它涉及到具体容器和镜像的配置文件定制都较繁琐（oci维护一个镜像和容器规范，你可以用ctr oci spec > /etc/containerd/cri-base.json获得这些基础配置。），因为新THE CONTAINER LAYERS和新THE IMAGE LAYERS层处理都很复杂。---- 所以我们得借助buildkit这样的工具。

> 除此之外，你还可以直接push到registry，sudo buildctl build --frontend dockerfile.v0 --local context=./ --local dockerfile=./ --output type=image,name=docker.io/minlearn/dafsdf:latest,push=true
还可以导出为一个OCI tarball
buildctl build ... --output type=oci,dest=path/to/output.tar
buildctl build ... --output type=oci > output.tar

> 对docker OCI runtime对它的理解可以解决前面一个没有解决的问题(ramdisk containerd下不能pivot root的问题，但是ramdisk chroot本身就是不受推荐的所以略过)。

甚至可以导出构建中使用的cache,导出缓存对运营registry的ci/cd服务商非常有用。因为可以加速构建过程。

docker mount方案
-----

上面谈完了skip registry build的docker commit方案，对于mount方案，其实这个似乎更接近cloudbase那套(我们注意到cloudbash那个保存后提交也有一定时间延迟的，但是延迟很小，可能并不是在提交并building。)，这个就是层的概念。docker本身就支持mount主机上的目录并run,buildkit在构建过程中就直接支持Buildkit allows secrets files to be mounted as a part of the build process.  These secrets are kept in-memory and not stored within the image:

RUN --mount=type=bind (the default mount type)
This mount type allows binding directories (read-only) in the context or in an image to the build container.
RUN --mount=type=cache
This mount type allows the build container to cache directories for compilers and package managers.

但这些是buildkit是在构建时的mount，与运行时的mount要分开。对volume的定义是可以写在oci runtime-spec配置文件中的。

Volume Permission and Ownership

Volume permissions van be changed by configuring the ownership within the Dockerfile.You can, like Docker, mount volumes from the host to the container. The following mounts /home/Documents/images from the host to the container at /mnt/images with read and write permissions:

```
"mounts": [
    {
        "destination": "/mnt/images",
        "type": "bind",
        "source": "/home/Documents/images",
        "options": [
            "rbind",
            "rw"
        ]
    }
]

Remember that the UID:GID pair is relative to the user namespace that the user is going to run the container with. 
```

这种方案下，只要类似openfaas gateway:8080/ui或cloudbase后台支持，可以完全分离docker构建/运行时所用的容器本身镜像和外来数据。在数据发生变动时，以手动提交方式或自动方式触发一次构建。

-----


具体到将上述猜想做到openfaas+vscodelonline产品和将这个功能放进管理后端中，这实际上是处理openfaas后端如何调用openfaas-cli工具（通过json api）这类问题和https://code.visualstudio.com/docs/remote/containers-advanced所提到的那些问题。除此之外，新的加了edit按钮的后端，必须使得每个容器被depoly时，faas自动为其导出数据目录，不必写明在docker-composer.yml中最好。



