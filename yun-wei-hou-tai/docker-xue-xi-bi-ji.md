# Docker学习笔记

\[TOC\]

## Docker简介

Docker 最初是 dotCloud 公司创始人 Solomon Hykes 在法国期间发起的一个公司内部项目，它是基于 dotCloud 公司多年云服务技术的一次革新，并于 2013 年 3 月以 Apache 2.0 授权协议开源，主要项目代码在 GitHub 上进行维护。Docker 项目后来还加入了 Linux 基金会，并成立推动 开放容器联盟（OCI）。 Docker 自开源后受到广泛的关注和讨论，至今其 GitHub 项目已经超过 4 万 6 千个星标和一万多个 fork。甚至由于 Docker 项目的火爆，在 2013 年底，dotCloud 公司决定改名为 Docker。Docker 最初是在 Ubuntu 12.04 上开发实现的；Red Hat 则从 RHEL 6.5 开始对 Docker 进行支持；Google 也在其 PaaS 产品中广泛应用 Docker。 Docker 使用 Google 公司推出的 Go 语言 进行开发实现，基于 Linux 内核的 cgroup，namespace，以及 AUFS 类的 Union FS 等技术，对进程进行封装隔离，属于 操作系统层面的虚拟化技术。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。

Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得 Docker 技术比虚拟机技术更为轻便、快捷。 下面的图片比较了 Docker 和传统虚拟化方式的不同之处。传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便

## docker优势

* 更高效的利用系统资源

  由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更高效。因此，相比虚拟机技术，一个相同配置的主机，往往可以运行更多数量的应用。

* 更快速的启动时间

  传统的虚拟机技术启动应用服务往往需要数分钟，而 Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。

* 一致的运行环境

  开发过程中一个常见的问题是环境一致性问题。由于开发环境、测试环境、生产环境不一致，导致有些 bug 并未在开发过程中被发现。而 Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性，从而不会再出现 「这段代码在我机器上没问题啊」 这类问题。

* 持续交付和部署 对开发和运维（DevOps）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。 使用 Docker 可以通过定制应用镜像来实现持续集成、持续交付、部署。开发人员可以通过 Dockerfile 来进行镜像构建，并结合 持续集成\(Continuous Integration\) 系统进行集成测试，而运维人员则可以直接在生产环境中快速部署该镜像，甚至结合 持续部署\(Continuous Delivery/Deployment\) 系统
* 进行自动部署。 而且使用 Dockerfile 使镜像构建透明化，不仅仅开发团队可以理解应用运行环境，也方便运维团队理解应用运行所需条件，帮助更好的生产环境中部署该镜像。
* 更轻松的迁移

  由于 Docker 确保了执行环境的一致性，使得应用的迁移更加容易。Docker 可以在很多平台上运行，无论是物理机、虚拟机、公有云、私有云，甚至是笔记本，其运行结果是一致的。因此用户可以很轻易的将在一个平台上运行的应用，迁移到另一个平台上，而不用担心运行环境的变化导致应用无法正常运行的情况。

* 更轻松的维护和扩展

  Docker 使用的分层存储以及镜像的技术，使得应用重复部分的复用更为容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单。此外，Docker 团队同各个开源项目团队一起维护了一大批高质量的 官方镜像，既可以直接在生产环境使用，又可以作为基础进一步定制，大大的降低了应用服务的镜像制作成本。

## 基本概念

Docker 包括三个基本概念

* 镜像（Image）
* 容器（Container）
* 仓库（Repository）

  理解了这三个概念，就理解了 Docker 的整个生命周期。

### Docker 镜像

Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

#### 分层存储

因为镜像包含操作系统完整的 root 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 Union FS 的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。 镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。 分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。

### Docker 容器

镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。 容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。

前面讲过镜像使用的是分层存储，容器也是如此。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。 容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。 按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。 数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

### Docker Registry

镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。 一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。 通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 &lt;仓库名&gt;:&lt;标签&gt; 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签。 以 Ubuntu 镜像 为例，ubuntu 是仓库的名字，其内包含有不同的版本标签，如，16.04, 18.04。我们可以通过 ubuntu:14.04，或者 ubuntu:18.04 来具体指定所需哪个版本的镜像。如果忽略了标签，比如 ubuntu，那将视为 ubuntu:latest。 仓库名经常以 两段式路径 形式出现，比如 jwilder/nginx-proxy，前者往往意味着 Docker Registry 多用户环境下的用户名，后者则往往是对应的软件名。但这并非绝对，取决于所使用的具体 Docker Registry 的软件或服务。

#### Docker Registry 公开服务

Docker Registry 公开服务是开放给用户使用、允许用户管理镜像的 Registry 服务。一般这类公开服务允许用户免费上传、下载公开的镜像，并可能提供收费服务供用户管理私有镜像。 最常使用的 Registry 公开服务是官方的 [Docker Hub](https://hub.docker.com/)，这也是默认的 Registry，并拥有大量的高质量的官方镜像。除此以外，还有 CoreOS 的 [Quay.io](https://quay.io/repository/)，CoreOS 相关的镜像存储在这里；Google 的 [Google Container Registry](https://cloud.google.com/container-registry/)，Kubernetes 的镜像使用的就是这个服务。 由于某些原因，在国内访问这些服务可能会比较慢。国内的一些云服务商提供了针对 Docker Hub 的镜像服务（Registry Mirror），这些镜像服务被称为加速器。常见的有 [阿里云加速器](https://account.aliyun.com/login/login.htm?oauth_callback=https%3A%2F%2Fcr.console.aliyun.com%2F#/accelerator)、[DaoCloud](https://www.daocloud.io/mirror#accelerator-doc) 加速器 等。使用加速器会直接从国内的地址下载 Docker Hub 的镜像，比直接从 Docker Hub 下载速度会提高很多。

### 私有 Docker Registry

除了使用公开服务外，用户还可以在本地搭建私有 Docker Registry。Docker 官方提供了 [Docker Registry](https://hub.docker.com/_/registry/) 镜像，可以直接使用做为私有 Registry 服务。

## [docker安装](https://yeasy.gitbooks.io/docker_practice/install/ubuntu.html)

如果在使用过程中发现拉取 Docker 镜像十分缓慢，可以配置 [Docker 国内镜像加速](https://yeasy.gitbooks.io/docker_practice/install/mirror.html)。

## 镜像管理

### 获取镜像

之前提到过，Docker Hub 上有大量的高质量的镜像可以用，这里我们就说一下怎么获取这些镜像。 从 Docker 镜像仓库获取镜像的命令是 docker pull。其命令格式为：

```text
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

具体的选项可以通过 docker pull --help 命令看到，这里我们说一下镜像名称的格式。 Docker 镜像仓库地址：地址的格式一般是 &lt;域名/IP&gt;\[:端口号\]。默认地址是 Docker Hub。 仓库名：如之前所说，这里的仓库名是两段式名称，即 &lt;用户名&gt;/&lt;软件名&gt;。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。 我们测试拉取ubuntu 18.04的镜像：

```text
plainchant@plainchant-pct:~$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
6cf436f81810: Pull complete 
987088a85b96: Pull complete 
b4624b3efe06: Pull complete 
d42beb8ded59: Pull complete 
Digest: sha256:7a47ccc3bbe8a451b500d2b53104868b46d60ee8f5b35a24b41a86077c650210
Status: Downloaded newer image for ubuntu:18.04
```

上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub 获取镜像。而镜像名称是 ubuntu:18.04，因此将会获取官方镜像 library/ubuntu 仓库中标签为 18.04 的镜像。 从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的 ID 的前 12 位。并且下载结束后，给出该镜像完整的 sha256 的摘要，以确保下载一致性。

### 运行镜像

有了镜像后，我们就能够以这个镜像为基础启动并运行一个容器。以上面的 ubuntu:18.04 为例，如果我们打算启动里面的 bash 并且进行交互式操作的话，可以执行下面的命令。

```text
plainchant@plainchant-pct:~$ docker run -it --rm ubuntu:18.04 bash
root@cce17638946a:/# cat /etc/os-release 
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.1 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

命令解释：

* -it：这是两个参数，一个是 -i：交互式操作，一个是 -t 终端。我们这里打算进入 bash 执行一些命令并查看返回结果，因此我们需要交互式终端。
* --rm：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 docker rm。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 --rm 可以避免浪费空间。
* ubuntu:18.04：这是指用 ubuntu:18.04 镜像为基础来启动容器。
* bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 bash。

通过docker ps可以看到docker容器被创建：

```text
plainchant@plainchant-pct:~$ docker ps -a

CONTAINER ID        IMAGE              COMMAND            CREATED            STATUS              PORTS              NAMES

cbea5cd48255        ubuntu:18.04        "bash"              9 seconds ago      Up 9 seconds                            eager_chandrasekhar
```

然后我们使用exit退出docker运行，通过docker ps可以看到docker容器被删除：

```text
plainchant@plainchant-pct:~$ docker ps -a

CONTAINER ID        IMAGE              COMMAND            CREATED            STATUS              PORTS              NAMES
```

### 列出镜像

要想列出已经下载下来的镜像，可以使用 docker images 命令。

```text
plainchant@plainchant-pct:~$ docker images 

REPOSITORY          TAG                IMAGE ID            CREATED            SIZE

ubuntu              18.04              47b19964fb50        3 weeks ago        88.1MB

hello-world        latest              fce289e99eb9        2 months ago        1.84kB
```

镜像 ID 则是镜像的唯一标识，一个镜像可以对应多个标签。标签表示版本号。

列表中的镜像体积总和并非是所有镜像实际硬盘消耗。由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。 你可以通过以下命令来便捷的查看镜像、容器、数据卷所占用的空间。

```text
plainchant@plainchant-pct:~$ docker system df

TYPE                TOTAL              ACTIVE              SIZE                RECLAIMABLE

Images              2                  1                  88.14MB            1.84kB (0%)

Containers          1                  1                  0B                  0B

Local Volumes      0                  0                  0B                  0B

Build Cache        0                  0                  0B                  0B
```

### 虚悬镜像

有时候可以看到一个特殊的镜像，这个镜像既没有仓库名，也没有标签，均为 。

```text
<none> <none> 00285df0df87 5 days ago 342 MB
```

这个镜像原本是有镜像名和标签的，原来为 mongo:3.2，随着官方镜像维护，发布了新版本后，重新 docker pull mongo:3.2 时，mongo:3.2 这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 。除了 docker pull 可能导致这种情况，docker build 也同样可以导致这种现象。由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为  的镜像。这类无标签镜像也被称为 虚悬镜像\(dangling image\) ，可以用下面的命令专门显示这类镜像：

```text
plainchant@plainchant-pct:~$ docker images -f dangling=true

REPOSITORY          TAG                IMAGE ID            CREATED            SIZE
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。

```text
$ docker image prune
```

### 中间层镜像

为了加速镜像构建、重复利用资源，Docker 会利用 中间层镜像。所以在使用一段时间后，可能会看到一些依赖的中间层镜像。默认的 docker image ls 列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 -a 参数。

```text
$ docker image ls -a
```

这样会看到很多无标签的镜像，与之前的虚悬镜像不同，这些无标签的镜像很多都是中间层镜像，是其它镜像所依赖的镜像。这些无标签镜像不应该删除，否则会导致上层镜像因为依赖丢失而出错。实际上，这些镜像也没必要删除，因为之前说过，相同的层只会存一遍，而这些镜像是别的镜像的依赖，因此并不会因为它们被列出来而多存了一份，无论如何你也会需要它们。只要删除那些依赖它们的镜像后，这些依赖的中间层镜像也会被连带删除。

### 列出部分镜像

不加任何参数的情况下，docker image ls 会列出所有顶级镜像，但是有时候我们只希望列出部分镜像。

* 根据仓库名列出镜像

```text
$ docker images ubuntu
```

* 指定仓库名和标签

```text
$ docker images ubuntu:18.04
```

* 过滤器参数 --filter

我们希望看到在 mongo:3.2 之后建立的镜像，可以用下面的命令：

```text
$ docker images -f since=mongo:3.2
```

如果镜像构建时，定义了 LABEL，还可以通过 LABEL 来过滤。

```text
$ docker images -f label=com.example.version=0.1
```

### 以特定格式显示

* 列出镜像的 ID 

```text
docker images -q
```

* [go模板语法](https://gohugo.io/templates/introduction/)

下面的命令会直接列出镜像结果，并且只包含镜像ID和仓库名：

```text
plainchant@plainchant-pct:~$ docker images --format "{{.ID}}: {{.Repository}}"

47b19964fb50: ubuntu

fce289e99eb9: hello-world
```

以表格等距显示，并且有标题行，和默认一样，不过自己定义列：

```text
plainchant@plainchant-pct:~$ docker images --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"

IMAGE ID            REPOSITORY          TAG

47b19964fb50        ubuntu              18.04

fce289e99eb9        hello-world        latest
```

### 删除本地镜像

如果要删除本地的镜像，可以使用 docker image rm 命令，其格式为：

```text
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

#### 用 ID、镜像名、摘要删除镜像

其中，&lt;镜像&gt; 可以是 镜像短 ID、镜像长 ID、镜像名 或者 镜像摘要。docker images 默认列出的就已经是短 ID 了，一般取前3个字符以上，只要足够区分于别的镜像就可以了。如：

```text
$ docker image rm 501
```

也可以用镜像名，也就是 &lt;仓库名&gt;:&lt;标签&gt;，来删除镜像。

```text
$ docker image rm centos
```

更精确的是使用 镜像摘要 删除镜像。

```text
plainchant@plainchant-pct:~$ docker images --digests 

REPOSITORY          TAG                DIGEST                                                                    IMAGE ID            CREATED            SIZE

ubuntu              18.04              sha256:7a47ccc3bbe8a451b500d2b53104868b46d60ee8f5b35a24b41a86077c650210  47b19964fb50        3 weeks ago        88.1MB

hello-world        latest              sha256:2557e3c07ed1e38f26e389462d03ed943586f744621577a99efb77324b0fe535  fce289e99eb9        2 months ago        1.84kB
```

删除：

```text
$ docker image rm node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
```

#### Untagged 和 Deleted

如果观察上面这几个命令的运行输出信息的话，你会注意到删除行为分为两类，一类是 Untagged，另一类是 Deleted。我们之前介绍过，镜像的唯一标识是其 ID 和摘要，而一个镜像可以有多个标签。 因此当我们使用上面命令删除镜像的时候，实际上是在要求删除某个标签的镜像。所以首先需要做的是将满足我们要求的所有镜像标签都取消，这就是我们看到的 Untagged 的信息。因为一个镜像可以对应多个标签，因此当我们删除了所指定的标签后，可能还有别的标签指向了这个镜像，如果是这种情况，那么 Delete 行为就不会发生。所以并非所有的 docker image rm 都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已。 当该镜像所有的标签都被取消了，该镜像很可能会失去了存在的意义，因此会触发删除行为。镜像是多层存储结构，因此在删除的时候也是从上层向基础层方向依次进行判断删除。镜像的多层结构让镜像复用变动非常容易，因此很有可能某个其它镜像正依赖于当前镜像的某一层。这种情况，依旧不会触发删除该层的行为。直到没有任何层依赖当前层时，才会真实的删除当前层。这就是为什么，有时候会奇怪，为什么明明没有别的标签指向这个镜像，但是它还是存在的原因，也是为什么有时候会发现所删除的层数和自己 docker pull 看到的层数不一样的源。

#### 批量删除

删除所有仓库名为 redis 的镜像：

```text
$ docker image rm $(docker image ls -q redis)
```

或者删除所有在 mongo:3.2 之前的镜像：

```text
$ docker image rm $(docker image ls -q -f before=mongo:3.2)
```

### commit 镜像构成

**注意**： docker commit 命令除了学习之外，还有一些特殊的应用场合，比如被入侵后保存现场等。但是，不要使用 docker commit 定制镜像，定制镜像应该使用 Dockerfile 来完成。如果你想要定制镜像请查看下一小节。

我们拉取一个http的镜像运行测试：

```text
plainchant@plainchant-pct:~$ docker run --name webserver -d -p 80:80 nginx

Unable to find image 'nginx:latest' locally

latest: Pulling from library/nginx

f7e2b70d04ae: Pull complete 

08dd01e3f3ac: Pull complete 

d9ef3a1eb792: Pull complete 

Digest: sha256:e2048a115f51d61fa589f5f592f677300f1c122de07904e0ebefe44a6a89da23

Status: Downloaded newer image for nginx:latest

bc8229561b808bf75adc3f139d809f32d939fdd229e9071e4bf821f603a74fef
```

如果没有发现镜像，会自动从docker hub拉取最新的nginx镜像。

然后访问[http://localhost/](http://localhost/) 即可看到web服务已经启动。我们进入容器修改内容：

```text
plainchant@plainchant-pct:~$ docker exec -it webserver bash

root@bc8229561b80:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html

root@bc8229561b80:/# exit

exit
```

刷新浏览器，会看到网页内容变成了我们想要的内容。

我们修改了容器的文件，也就是改动了容器的存储层。我们可以通过 docker diff 命令看到具体的改动。

```text
plainchant@plainchant-pct:~$ docker diff webserver 

C /usr

C /usr/share

C /usr/share/nginx

C /usr/share/nginx/html

C /usr/share/nginx/html/index.html

C /root

A /root/.bash_history

C /var

C /var/cache

C /var/cache/nginx

A /var/cache/nginx/uwsgi_temp

A /var/cache/nginx/client_temp

A /var/cache/nginx/fastcgi_temp

A /var/cache/nginx/proxy_temp

A /var/cache/nginx/scgi_temp

C /run

A /run/nginx.pid
```

Docker 提供了一个 docker commit 命令，可以将容器的存储层保存下来成为镜像。换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。 docker commit 的语法格式为：

```text
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```

我们可以用下面的命令将容器保存为镜像：

```text
plainchant@plainchant-pct:~$ docker commit --author "plainchant" --message "修改了默认网页" webserver nginx:v2

sha256:eb70513429f5a2a7486a43fa05b7443a62d94a8e79a2511fd4d766628ac4841f
```

其中 --author 是指定修改的作者，而 --message 则是记录本次修改的内容。

```text
plainchant@plainchant-pct:~$ docker images 

REPOSITORY          TAG                IMAGE ID            CREATED            SIZE

nginx              v2                  eb70513429f5        38 seconds ago      109MB

nginx              latest              881bd08c0b08        6 hours ago        109MB

ubuntu              18.04              47b19964fb50        3 weeks ago        88.1MB

hello-world        latest              fce289e99eb9        2 months ago        1.84kB
```

可以看到我们提交成功了新镜像。我们还可以查询修改记录：

```text
plainchant@plainchant-pct:~$ docker history nginx:v2 
IMAGE CREATED CREATED BY SIZE COMMENT
eb70513429f5 About a minute ago nginx -g daemon off; 97B 修改了默认网页
881bd08c0b08 6 hours ago /bin/sh -c #(nop) CMD ["nginx" "-g" "daemon… 0B                  
<missing> 6 hours ago /bin/sh -c #(nop) STOPSIGNAL SIGTERM 0B                  
<missing> 6 hours ago /bin/sh -c #(nop) EXPOSE 80 0B                  
<missing> 6 hours ago /bin/sh -c ln -sf /dev/stdout /var/log/nginx… 22B                 
<missing> 6 hours ago /bin/sh -c set -x && apt-get update && apt… 54MB                
<missing> 6 hours ago /bin/sh -c #(nop) ENV NJS_VERSION=1.15.9.0.… 0B                  
<missing> 6 hours ago /bin/sh -c #(nop) ENV NGINX_VERSION=1.15.9-… 0B                  
<missing> 6 hours ago /bin/sh -c #(nop) LABEL maintainer=NGINX Do… 0B                  
<missing> 11 hours ago /bin/sh -c #(nop) CMD ["bash"] 0B                  
<missing> 11 hours ago /bin/sh -c #(nop) ADD file:5ea7dfe8c8bc87ebe… 55.3MB
```

我们可以来运行这个镜像。

```text
docker run --name web2 -d -p 81:80 nginx:v2
```

这里我们命名为新的服务为 web2，并且映射到 81 端口。我们可以直接访问 [http://localhost:81](http://localhost:81) 看到结果。

#### 慎用 docker commit

使用 docker commit 命令虽然可以比较直观的帮助理解镜像分层存储的概念，但是实际环境中并不会这样使用。 首先，如果仔细观察之前的 docker diff webserver 的结果，你会发现除了真正想要修改的 /usr/share/nginx/html/index.html 文件外，由于命令的执行，还有很多文件被改动或添加了。这还仅仅是最简单的操作，如果是安装软件包、编译构建，那会有大量的无关内容被添加进来，如果不小心清理，将会导致镜像极为臃肿。 此外，使用 docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。而且，即使是这个制作镜像的人，过一段时间后也无法记清具体在操作的。虽然 docker diff 或许可以告诉得到一些线索，但是远远不到可以确保生成一致镜像的地步。这种黑箱镜像的维护工作是非常痛苦的。 而且，回顾之前提及的镜像所使用的分层存储的概念，除当前层外，之前的每一层都是不会发生改变的，换句话说，任何修改的结果仅仅是在当前层进行标记、添加、修改，而不会改动上一层。如果使用 docker commit 制作镜像，以及后期修改的话，每一次修改都会让镜像更加臃肿一次，所删除的上一层的东西并不会丢失，会一直如影随形的跟着这个镜像，即使根本无法访问到。这会让镜像更加臃肿。

## 使用 Dockerfile 定制镜像

镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。 Dockerfile 是一个文本文件，其内包含了一条条的指令\(Instruction\)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

### FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像我们之前运行了一个 nginx 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 FROM 就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

在 Docker Hub 上有非常多的高质量的官方镜像，有可以直接拿来使用的[服务类的镜像](https://yeasy.gitbooks.io/docker_practice/image/build.html)，如 nginx、redis、mongo、mysql、httpd、php、tomcat 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 node、openjdk、python、ruby、golang 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。 如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像，如 ubuntu、debian、centos、fedora、alpine 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。 除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

```text
FROM scratch
...
```

如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。 不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm、coreos/etcd。对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。使用 Go 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。

### RUN 执行命令

RUN 指令是用来执行命令行命令的。由于命令行的强大能力，RUN 指令在定制镜像时是最常用的指令之一。其格式有两种：

* shell 格式：RUN &lt;命令&gt;，就像直接在命令行中输入的命令一样。

```text
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

* exec 格式：RUN \["可执行文件", "参数1", "参数2"\]，这更像是函数调用中的格式。

既然 RUN 就像 Shell 脚本一样可以执行命令，那么我们是否就可以像 Shell 脚本一样把每个命令对应一个 RUN 呢？比如这样：

```text
FROM debian:stretch
RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

之前说过，Dockerfile 中每一个指令都会建立一层，RUN 也不例外。每一个 RUN 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。

而上面的这种写法，创建了 7 层镜像。这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。

Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。 上面的 Dockerfile 正确的写法应该是这样：

```text
FROM debian:stretch
RUN buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

首先，之前所有的命令只有一个目的，就是编译、安装 redis 可执行文件。因此没有必要建立很多层，这只是一层的事情。因此，这里没有使用很多个 RUN 对一一对应不同的命令，而是仅仅使用一个 RUN 指令，并使用 && 将各个所需命令串联起来。将之前的 7 层，简化为了 1 层。在撰写 Dockerfile 的时候，要经常提醒自己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建。Dockerfile 支持 Shell 类的行尾添加  的命令换行方式，以及行首 \# 进行注释的格式。

此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 apt 缓存文件。这是很重要的一步，我们之前说过，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

### 构建镜像

我们写一个最简单的Dockerfile，构建nginx镜像：

```text
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

然后构建镜像：

```text
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker/nginx$ docker build -t nginx:v3 ./
Sending build context to Docker daemon 2.048kB
Step 1/2 : FROM nginx
 ---> 881bd08c0b08
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 87499cf86b7e
Removing intermediate container 87499cf86b7e
 ---> a42683f8e41c
Successfully built a42683f8e41c
Successfully tagged nginx:v3
```

从命令的输出结果中，我们可以清晰的看到镜像的构建过程。在 Step 2 中，如同我们之前所说的那样，RUN 指令启动了一个容器87499cf86b7e，执行了所要求的命令，并最后提交了这一层a42683f8e41c，随后删除了所用到的这个容器87499cf86b7e。

运行看下结果：

```text
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker/nginx$ docker run --name webserver -d -p 80:80 nginx:v3 

adb148952d8ce60567c8228eb0119f6c6f0a4c6f3177db5fe991fb6e192218d7
```

在网页[http://localhost:81/](http://localhost:81/) 中可以看到内容。

#### 镜像构建上下文（Context）

如果注意，会看到 docker build 命令最后有一个 .。. 表示当前目录，而 Dockerfile 就在当前目录，因此不少初学者以为这个路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定上下文路径。也就是命令执行的目录。

首先我们要理解 docker build 的工作原理。Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。 当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？ 这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

如果在 Dockerfile 中这么写：

```text
COPY ./package.json /app/
```

这并不是要复制执行 docker build 命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json，而是复制 上下文（context） 目录下的 package.json。也就是build命令最后指定的目录。

一般来说，应该会将 Dockerfile 置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 .gitignore 一样的语法写一个 .dockerignore，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。在默认情况下，如果不额外指定 Dockerfile 的话，会将上下文目录下的名为 Dockerfile 的文件作为 Dockerfile上下文目录。

#### 直接用 Git repo 进行构建

或许你已经注意到了，docker build 还支持从 URL 构建，比如可以直接从 Git repo 中构建：

```text
$ docker build https://github.com/twang2218/gitlab-ce-zh.git#:11.1
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM gitlab/gitlab-ce:11.1.0-ce.0
11.1.0-ce.0: Pulling from gitlab/gitlab-ce
aed15891ba52: Already exists
773ae8583d14: Already exists
...
```

这行命令指定了构建所需的 Git repo，并且指定默认的 master 分支，构建目录为 /11.1/，然后 Docker 就会自己去 git clone 这个项目、切换到指定分支、并进入到指定目录后开始构建。

#### 用给定的 tar 压缩包构建

```text
$ docker build http://server/context.tar.gz
```

如果所给出的 URL 不是个 Git repo，而是个 tar 压缩包，那么 Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。

#### 从标准输入中读取 Dockerfile 进行构建

```text
docker build - < Dockerfile
或
cat Dockerfile | docker build -
```

如果标准输入传入的是文本文件，则将其视为 Dockerfile，并开始构建。这种形式由于直接从标准输入中读取 Dockerfile 的内容，它没有上下文，因此不可以像其他方法那样可以将本地文件 COPY 进镜像之类的事情。

## Dockerfile 指令详解

### COPY 复制文件

格式：

```text
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
```

和 RUN 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

COPY 指令将从构建上下文目录中 &lt;源路径&gt; 的文件/目录复制到新的一层的镜像内的 &lt;目标路径&gt; 位置。比如：

```text
COPY package.json /usr/src/app/
```

&lt;源路径&gt; 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 [filepath.Match](https://golang.org/pkg/path/filepath/#Match) 规则，如：

```text
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

&lt;目标路径&gt; 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 WORKDIR 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。此外，还需要注意一点，使用 COPY 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。

在使用该指令的时候还可以加上 --chown=: 选项来改变文件的所属用户及所属组。

```text
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

### ADD 更高级的复制文件

ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。如果 &lt;源路径&gt; 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 &lt;目标路径&gt; 去。在某些情况下，这个自动解压缩的功能非常有用。

另外需要注意的是，ADD 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。因此在 COPY 和 ADD 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 COPY 指令，仅在需要自动解压缩的场合使用 ADD。

### CMD 容器启动命令

CMD 指令的格式和 RUN 相似，也是两种格式：

* shell 格式：CMD &lt;命令&gt;
* exec 格式：CMD \["可执行文件", "参数1", "参数2"...\]
* 参数列表格式：CMD \["参数1", "参数2"...\]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。

CMD 指令就是用于指定默认的容器主进程的启动命令的。 在运行时可以指定新的命令来替代镜像设置中的这个默认命令，比如，ubuntu 镜像默认的 CMD 是 /bin/bash，如果我们直接 docker run -it ubuntu 的话，会直接进入 bash。我们也可以在运行时指定运行别的命令，如 docker run -it ubuntu cat /etc/os-release。这就是用 cat /etc/os-release 命令替换了默认的 /bin/bash 命令了，输出了系统版本信息。

```text
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker/nginx$ docker run -it ubuntu:18.04 cat /etc/os-release

NAME="Ubuntu"

VERSION="18.04.1 LTS (Bionic Beaver)"

ID=ubuntu

ID_LIKE=debian

PRETTY_NAME="Ubuntu 18.04.1 LTS"

VERSION_ID="18.04"

HOME_URL="https://www.ubuntu.com/"

SUPPORT_URL="https://help.ubuntu.com/"

BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"

PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"

VERSION_CODENAME=bionic

UBUNTU_CODENAME=bionic
```

在指令格式上，一般推荐使用 exec 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 "，而不要使用单引号。

如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：

```text
CMD echo $HOME
```

在实际执行中，会将其变更为：

```text
CMD [ "sh", "-c", "echo $HOME" ]
```

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

如下面命令：

```text
CMD service nginx start
```

会被理解为

```text
CMD [ "sh", "-c", "service nginx start"]
```

因此主进程实际上是 sh。那么当 service nginx start 命令结束后，sh 也就结束了，sh 作为主进程退出了，就会令容器退出。结果就是容器执行后立即退出了。

正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：

```text
CMD ["nginx", "-g", "daemon off;"]
```

### ENTRYPOINT 入口点

ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 格式和 shell 格式。 ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：

```text
<ENTRYPOINT> "<CMD>"
```

#### 场景一：让镜像变成像命令一样使用

```text
FROM ubuntu:18.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT [ "curl", "-s", "https://ip.cn" ]
```

运行docker run myip -i时会将-i做为cmd传递给entrypoint，从而实现了追加参数的功能。

#### 场景二：应用运行前的准备工作

启动容器就是启动主进程，但有些时候，启动主进程前，需要一些准备工作。 比如 mysql 类的数据库，可能需要一些数据库配置、初始化的工作，这些工作要在最终的 mysql 服务器运行之前解决。

官方镜像 redis 中就是这么做的：

```text
FROM alpine:3.4
...
RUN addgroup -S redis && adduser -S -G redis redis
...
ENTRYPOINT ["docker-entrypoint.sh"]
EXPOSE 6379
CMD [ "redis-server" ]
```

可以看到其中为了 redis 服务创建了 redis 用户，并在最后指定了 ENTRYPOINT 为 docker-entrypoint.sh 脚本。

```text
#!/bin/sh
...
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    chown -R redis .
    exec su-exec redis "$0" "$@"
fi
exec "$@"
```

该脚本的内容就是根据 CMD 的内容来判断，如果是 redis-server 的话，则切换到 redis 用户身份启动服务器，否则依旧使用 root 身份执行。比如：

```text
$ docker run -it redis id
uid=0(root) gid=0(root) groups=0(root)
```

ENV 设置环境变量 格式有两种：

```text
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

这个指令很简单，就是设置环境变量而已，无论是后面的其它指令，如 RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。

```text
ENV VERSION=1.0 DEBUG=on \
    NAME="Happy Feet"
```

使用时和shell一致，加$号即可，下列指令可以支持环境变量展开： ADD、COPY、ENV、EXPOSE、LABEL、USER、WORKDIR、VOLUME、STOPSIGNAL、ONBUILD

### ARG 构建参数

```text
格式：ARG <参数名>[=<默认值>]
```

构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是，ARG 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 ARG 保存密码之类的信息，因为 docker history 还是可以看到所有值的。 Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg &lt;参数名&gt;=&lt;值&gt; 来覆盖。

### VOLUME 定义匿名卷

格式为： VOLUME \["&lt;路径1&gt;", "&lt;路径2&gt;"...\] VOLUME &lt;路径&gt; 之前我们说过，容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷\(volume\)中。

为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。

```text
VOLUME /data
```

这里的 /data 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。当然，运行时可以覆盖这个挂载设置。比如：

```text
docker run -d -v mydata:/data xxxx
```

在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代了 Dockerfile 中定义的匿名卷的挂载配置。

### EXPOSE 声明端口

```text
格式为 EXPOSE <端口1> [<端口2>...]。
```

EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 docker run -P 时，会自动随机映射 EXPOSE 的端口。 要将 EXPOSE 和在运行时使用

```text
-p <宿主端口>:<容器端口>
```

区分开来。-p，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。

### WORKDIR 指定工作目录

```text
格式为 WORKDIR <工作目录路径>。
```

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录。

```text
RUN cd /app
RUN echo "hello" > world.txt
```

如果将这个 Dockerfile 进行构建镜像运行后，会发现找不到 /app/world.txt 文件，或者其内容不是 hello。原因其实很简单，在 Shell 中，连续两行是同一个进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令；而在 Dockerfile 中，这两行 RUN 命令的执行环境根本不同，是两个完全不同的容器。这就是对 Dockerfile 构建分层存储的概念不了解所导致的错误。 之前说过每一个 RUN 都是启动一个容器、执行命令、然后提交存储层文件变更。第一层 RUN cd /app 的执行仅仅是当前进程的工作目录变更，一个内存上的变化而已，其结果不会造成任何文件变更。而到第二层的时候，启动的是一个全新的容器，跟第一层的容器更完全没关系，自然不可能继承前一层构建过程中的内存变化。 因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。

### USER 指定当前用户

格式：USER &lt;用户名&gt;\[:&lt;用户组&gt;\] USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。 当然，和 WORKDIR 一样，USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

```text
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

如果以 root 执行的脚本，在执行期间希望改变身份，比如希望以某个已经建立好的用户来运行某个服务进程，不要使用 su 或者 sudo，这些都需要比较麻烦的配置，而且在 TTY 缺失的环境下经常出错。建议使用 [gosu](https://github.com/tianon/gosu)。

```text
# 建立 redis 用户，并使用 gosu 换另一个用户执行命令
RUN groupadd -r redis && useradd -r -g redis redis
# 下载 gosu
RUN wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/1.7/gosu-amd64" \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true
# 设置 CMD，并以另外的用户执行
CMD [ "exec", "gosu", "redis", "redis-server" ]
```

### HEALTHCHECK 健康检查

```text
格式：
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
```

HEALTHCHECK 指令是告诉 Docker 应该如何进行判断容器的状态是否正常，这是 Docker 1.12 引入的新指令。

当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting，在 HEALTHCHECK 指令检查成功后变为 healthy，如果连续一定次数失败，则会变为 unhealthy。 HEALTHCHECK 支持下列选项：

* --interval=&lt;间隔&gt;：两次健康检查的间隔，默认为 30 秒；
* --timeout=&lt;时长&gt;：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
* --retries=&lt;次数&gt;：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。

  和 CMD, ENTRYPOINT 一样，HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

## Dockerfile多阶段构建

Docker v17.05 开始支持多阶段构建 \(multistage builds\)。使用多阶段构建我们就可以很容易解决前面提到的问题，并且只需要编写一个 Dockerfile： 例如，编写 Dockerfile 文件

```text
FROM golang:1.9-alpine as builder
RUN apk --no-cache add git
WORKDIR /go/src/github.com/go/helloworld/
RUN go get -d -v github.com/go-sql-driver/mysql
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
FROM alpine:latest as prod
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/go/helloworld/app .
CMD ["./app"]
```

构建镜像

```text
$ docker build -t go/helloworld:3 .
```

对比三个镜像大小

```text
$ docker image ls
REPOSITORY TAG IMAGE ID CREATED SIZE
go/helloworld 3 d6911ed9c846 7 seconds ago 6.47MB
go/helloworld 2 f7cf3465432c 22 seconds ago 6.47MB
go/helloworld 1 f55d3e16affc 2 minutes ago 295MB
```

很明显使用多阶段构建的镜像体积小，同时也完美解决了上边提到的问题。

我们可以使用 as 来为某一阶段命名，例如

```text
FROM golang:1.9-alpine as builder
```

例如当我们只想构建 builder 阶段的镜像时，增加 --target=builder 参数即可

```text
$ docker build --target builder -t username/imagename:tag .
```

构建时从其他镜像复制文件 上面例子中我们使用 COPY --from=0 /go/src/github.com/go/helloworld/app . 从上一阶段的镜像中复制文件，我们也可以复制任意镜像中的文件。

```text
$ COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

## 操作 Docker 容器

容器是 Docker 又一核心概念。 简单的说，容器是独立运行的一个或一组应用，以及它们的运行态环境。对应的，虚拟机可以理解为模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用。

### 启动容器

启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止状态（stopped）的容器重新启动。

#### 新建并启动

所需要的命令主要为 docker run。 例如，下面的命令输出一个 “Hello World”，之后终止容器。

```text
$ docker run ubuntu:18.04 /bin/echo 'Hello world'
Hello world
```

下面的命令则启动一个 bash 终端，允许用户进行交互。

```text
$ docker run -t -i ubuntu:18.04 /bin/bash
root@af8bae53bdd3:/#
```

其中，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， -i 则让容器的标准输入保持打开。

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

* 检查本地是否存在指定的镜像，不存在就从公有仓库下载
* 利用镜像创建并启动一个容器
* 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
* 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
* 从地址池配置一个 ip 地址给容器
* 执行用户指定的应用程序
* 执行完毕后容器被终止

#### 启动已终止容器

可以利用 docker container start 命令，直接将一个已经终止的容器启动运行。

### 后台运行

更多的时候，需要让 Docker 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加 -d 参数来实现

要获取容器的输出信息，可以通过 docker container logs 命令。

### 终止容器

可以使用 docker container stop 来终止一个运行中的容器。 此外，当 Docker 容器中指定的应用终结时，容器也自动终止。此外，docker container restart 命令会将一个运行态的容器终止，然后再重新启动它

### 进入容器

在使用 -d 参数时，容器启动后会进入后台。 某些时候需要进入容器进行操作，包括使用 docker attach 命令或 docker exec 命令，推荐大家使用 docker exec 命令.

#### attach 命令

下面示例如何使用 docker attach 命令。

```text
$ docker run -dit ubuntu
243c32535da7d142fb0e6df616a3c3ada0b8ab417937c853a9e1c251f499f550
$ docker container ls
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
243c32535da7 ubuntu:latest "/bin/bash" 18 seconds ago Up 17 seconds nostalgic_hypatia
$ docker attach 243c
root@243c32535da7:/#
```

注意： 如果从这个 stdin 中 exit，会导致容器的停止。

#### exec 命令

-i -t 参数 docker exec 后边可以跟多个参数，这里主要说明 -i -t 参数。 只用 -i 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。 当 -i -t 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。

```text
$ docker run -dit ubuntu
69d137adef7a8a689cbcb059e94da5489d3cddd240ff675c640c8d96e84fe1f6
$ docker container ls
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
69d137adef7a ubuntu:latest "/bin/bash" 18 seconds ago Up 17 seconds zealous_swirles
$ docker exec -i 69d1 bash
ls
bin
boot
dev
...
$ docker exec -it 69d1 bash
root@69d137adef7a:/#
```

如果从这个 stdin 中 exit，不会导致容器的停止。这就是为什么推荐大家使用 docker exec 的原因。

### 导出和导入容器

#### 导出容器

如果要导出本地某个容器，可以使用 docker export 命令。

```text
$ docker container ls -a
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
7691a814370e ubuntu:18.04 "/bin/bash" 36 hours ago Exited (0) 21 hours ago test
$ docker export 7691a814370e > ubuntu.tar
```

这样将导出容器快照到本地文件。

#### 导入容器快照

可以使用 docker import 从容器快照文件中再导入为镜像，例如

```text
$ cat ubuntu.tar | docker import - test/ubuntu:v1.0
$ docker image ls
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
test/ubuntu v1.0 9d37a6082e97 About a minute ago 171.3 MB
```

此外，也可以通过指定 URL 或者某个目录来导入，例如

```text
$ docker import http://example.com/exampleimage.tgz example/imagerepo
```

### 删除容器

可以使用 docker container rm 来删除一个处于终止状态的容器。例如

```text
$ docker container rm trusting_newton
trusting_newton
```

如果要删除一个运行中的容器，可以添加 -f 参数。Docker 会发送 SIGKILL 信号给容器。 清理所有处于终止状态的容器 用 docker container ls -a 命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用下面的命令可以清理掉所有处于终止状态的容器。

```text
$ docker container prune
```

## 访问仓库

仓库（Repository）是集中存放镜像的地方

### Docker Hub

目前 Docker 官方维护了一个公共仓库 Docker Hub，其中已经包括了数量超过 15,000 的镜像。大部分需求都可以通过在 Docker Hub 中直接下载镜像来实现。

可以通过执行 docker login 命令交互式的输入用户名及密码来完成在命令行界面登录 Docker Hub。通过 docker logout 退出登录。通过 docker search 命令来查找官方仓库中的镜像，并利用 docker pull 命令来将它下载到本地。通过 docker push 命令来将自己的镜像推送到 Docker Hub。

### 私有仓库

有时候使用 Docker Hub 这样的公共仓库可能不方便，用户可以创建一个本地仓库供私人使用。 \(docker-registry\]\([https://docs.docker.com/registry/\)是官方提供的工具，可以用于构建私有的镜像仓库。](https://docs.docker.com/registry/%29是官方提供的工具，可以用于构建私有的镜像仓库。)

#### 安装运行 docker-registry

* 容器运行

  你可以通过获取官方 registry 镜像来运行。

```text
$ docker run -d -p 5000:5000 --restart=always --name registry registry
```

这将使用官方的 registry 镜像来启动私有仓库。默认情况下，仓库会被创建在容器的 /var/lib/registry 目录下。你可以通过 -v 参数来将镜像文件存放在本地的指定路径。例如下面的例子将上传的镜像放到本地的 /opt/data/registry 目录。

```text
$ docker run -d \
    -p 5000:5000 \
    -v /opt/data/registry:/var/lib/registry \
    registry
```

#### 创建自己的仓库

先在dockerhub上创建一个仓库，可以是公有或者私有仓库，然后使用docker login登录账户，然后推送本地镜像到仓库：

```text
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker tag hello-world:latest plainchant/helloworld
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker images
REPOSITORY TAG IMAGE ID CREATED SIZE
nginx v3 a42683f8e41c 17 hours ago 109MB
nginx v2 eb70513429f5 17 hours ago 109MB
nginx latest 881bd08c0b08 23 hours ago 109MB
node alpine 842caa90d45b 5 days ago 75.2MB
ubuntu 18.04 47b19964fb50 4 weeks ago 88.1MB
ubuntu latest 47b19964fb50 4 weeks ago 88.1MB
hello-world latest fce289e99eb9 2 months ago 1.84kB
plainchant/helloworld latest fce289e99eb9 2 months ago 1.84kB
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker push plainchant/helloworld
plainchant/helloworld plainchant/helloworld:latest  
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker push plainchant/helloworld
The push refers to repository [docker.io/plainchant/helloworld]
af0b15c8625b: Mounted from library/hello-world 
latest: digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a size: 524
```

上面命令将镜像推送到了主目录下，我们可以推送到指定目录下：

```text
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker push plainchant/study
plainchant/study plainchant/study:helloworld  
plainchant@plainchant-pct:~/workspace/BlockABC/codes/docker$ docker push plainchant/study:helloworld 
The push refers to repository [docker.io/plainchant/study]
af0b15c8625b: Mounted from plainchant/helloworld 
helloworld: digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a size: 524
```

* 在私有仓库上传、搜索、下载镜像

  创建好私有仓库之后，就可以使用 docker tag 来标记一个镜像，然后推送它到仓库。例如私有仓库地址为 127.0.0.1:5000。

使用 docker tag 将 ubuntu:latest 这个镜像标记为 127.0.0.1:5000/ubuntu:latest。 格式为 docker tag IMAGE\[:TAG\] \[REGISTRY\_HOST\[:REGISTRY\_PORT\]/\]REPOSITORY\[:TAG\]。

```text
$ docker tag ubuntu:latest 127.0.0.1:5000/ubuntu:latest
$ docker image ls
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
ubuntu latest ba5877dc9bec 6 weeks ago 192.7 MB
127.0.0.1:5000/ubuntu:latest latest ba5877dc9bec 6 weeks ago 192.7 MB
```

使用 docker push 上传标记的镜像。

```text
$ docker push 127.0.0.1:5000/ubuntu:latest
The push refers to repository [127.0.0.1:5000/ubuntu]
373a30c24545: Pushed
a9148f5200b0: Pushed
cdd3de0940ab: Pushed
fc56279bbb33: Pushed
b38367233d37: Pushed
2aebd096e0e2: Pushed
latest: digest: sha256:fe4277621f10b5026266932ddf760f5a756d2facd505a94d2da12f4f52f71f5a size: 1568
```

## Docker 数据管理

这一章介绍如何在 Docker 内部以及容器之间管理数据，在容器中管理数据主要有两种方式： 数据卷（Volumes） 挂载主机目录 \(Bind mounts\)

### 数据卷

数据卷 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

* 数据卷 可以在容器之间共享和重用
* 对 数据卷 的修改会立马生效
* 对 数据卷 的更新，不会影响镜像
* 数据卷 默认会一直存在，即使容器被删除

  > 注意：数据卷 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的 数据卷。

创建一个数据卷

```text
$ docker volume create my-vol
```

查看所有的 数据卷

```text
$ docker volume ls
local my-vol
```

启动一个挂载数据卷的容器 在用 docker run 命令的时候，使用 --mount 标记来将 数据卷 挂载到容器里。在一次 docker run 中可以挂载多个 数据卷。 下面创建一个名为 web 的容器，并加载一个 数据卷 到容器的 /webapp 目录。

```text
$ docker run -d -P \
    --name web \
    # -v my-vol:/wepapp \
    --mount source=my-vol,target=/webapp \
    training/webapp \
    python app.py
```

### 挂载主机目录

挂载一个主机目录作为数据卷 使用 --mount 标记可以指定挂载一个本地主机的目录到容器中去。

```text
$ docker run -d -P \
    --name web \
    # -v /src/webapp:/opt/webapp \
    --mount type=bind,source=/src/webapp,target=/opt/webapp \
    training/webapp \
    python app.py
```

上面的命令加载主机的 /src/webapp 目录到容器的 /opt/webapp目录。这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。本地目录的路径必须是绝对路径，以前使用 -v 参数时如果本地目录不存在 Docker 会自动为你创建一个文件夹，现在使用 --mount 参数时如果本地目录不存在，Docker 会报错。

查看数据卷的具体信息 在主机里使用以下命令可以查看 web 容器的信息

```text
$ docker inspect web
```

## Docker 中的网络功能介绍

Docker 允许通过外部访问容器或容器互联的方式来提供网络服务。

