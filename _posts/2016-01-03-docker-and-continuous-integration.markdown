---
layout: post
title:  "Docker 与 UPYUN 集成测试环境"
date:   2016-01-03 20:33:00
author: polym
comments: true
---

> 测试是软件开发过程中的重中之重，一个没有测试的项目，就像摸着石头过河，格外艰辛。而一个统一规范的集成测试环境，更是一个公司内部必不可少的组件。

## 什么是 Docker，Docker 的优势是什么

![docker-logo]({{ site.remoteurl }}/assets/docker-logo.png)

[Docker](https://github.com/docker/docker) 是一个开源项目，代码完全托管在 github 上，并保持着一个月一个版本的迭代速度。Docker 在 Linux 容器（LXC）的基础上进行进一步封装，让用户无需关心容器的管理，操作 Docker 就像操作一个轻量级虚拟机一样。Docker 与传统虚拟化的区别在于，每个容器是在操作系统层面实现的虚拟化，而不是在硬件层面。

Docker 的优势有很多：

- 秒级别启动容器
- 系统资源利用率高
- 资源隔离性好
- 在内核层面做虚拟化，更加高效
- 打包成 Docker 镜像就可以直接交付发布，交付部署简单
- 只要安装 Docker 的机器，都能使用该镜像，迁移性强
- 支持多平台扩展

## 集成测试环境需要具备哪几点

- 项目与项目之间，同项目不同代码提交之间，要确保绝对隔离，互不影响。
- 尽可能缩短构建项目的时间。
- 每次测试完成后，要保证测试环境绝对干净，避免对下次测试造成影响。
- 支持多版本语言测试，例如，Python 2.6.8，Python 2.7.3。
- 灵活配置第三方服务，如 redis，mysql 等。
- 系统资源限制，避免对并行测试造成影响。

## Docker + 集成测试 = GitLab-CI

开源的集成测试工具有很多，最为主流的当属 Jenkins 跟 GitLab-CI，GitLab 对此也提供了支持。GitLab-CI 是 GitLab 官方提供的一套持续集成环境，用以管理 GitLab Runner。GitLab 项目中有新的提交，就会触发 GitLab CI 服务器创建构建测试任务，并将任务指派给相关的 Runner，最后将测试结果展现出来。

## GitLab-Multi-Runner

GitLab-Multi-Runner 是基于 Go 语言实现的官方的 GitLab CI Runner，用于测试代码，并将测试结果返回给 GitLab CI 服务器。GitLab **8.0** 之后，集成了 CI 服务器，无需额外安装部署。在部署 GitLab 时，需要额外关注 GitLab 是否支持 HTTP 拉取代码，因为所有的 GitLab Runner 都是采用 HTTP 的方式拉取项目代码，然后构建测试的。

### Runner 安装

由于是 Go 项目，安装 GitLab-Multi-Runner 很简单，仅需下载二进制包，便可直接运行，无额外依赖。而且，官方还给出了 GitLab-Multi-Runner 的 Docker 镜像。因为习惯性用 Docker 解决问题，UPYUN 采用的是 Docker 镜像的安装方式。

~~~
docker run -d --name gitlab-runner --restart always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
gitlab/gitlab-runner:latest
~~~

可以看到启动的过程中，挂载了两个目录文件，一个是 /var/run/docker.sock 这个是由于 GitLab-Multi-Runner 会使用到 Docker，所以需要将 Docker 的套接字映射进去。另一个是 /srv/gitlab-runner/config，这里面放置的是 GitLab-Multi-Runner 的相关配置，包括其管理的 Runner 的配置。

### Runner 注册

~~~
docker exec -it gitlab-runner gitlab-runner register \
  --url "http://gitlab.widget-inc.com/ci" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "docker-shared" \
  --executor "docker" \
  --docker-image upyun/shared \
  --docker-mysql latest
~~~

`url`: CI 服务器的地址，`registration-token`: 项目公共 或者 私有 Runner 的注册 Token，`description`: Runner 的描述，`executor`: 执行器，可以选择 shell, ssh, docker, docker-ssh。`docker-image`: Docker 执行器使用的 Docker 镜像，会被项目中的 Docker 镜像的配置覆盖，`docker-mysql`: 连接第三方服务。

### docker，docker-ssh

docker, docker-ssh 都是采用 Docker 作为测试环境，都是启动一个 Docker 容器（containner），然后将测试脚本「注入」到此容器中执行，而两者区别就在「注入」二字。「docker 执行器」是在 Docker 启动容器时，将测试脚本作为命令行（CMD）传入。而
「docker-ssh 执行器」需要确保 Docker 启动的容器中有 sshd 服务，通过 ssh 的方式执行测试脚本。

### 项目配置 Runner 示例

Runner 已经注册成功，现在需要在项目中增加一个 .gitlab-ci.yml 的文件，使得 Runner 知道需要执行哪些命令。 .gitlab-ci.yml 是 Runner 的执行配置，需要放在项目的根目录上。.gitlab-ci.yml 的参数众多，这边就简单分享两个示例。项目参数可以参看 [Configuration of your builds with .gitlab-ci.yml](http://doc.gitlab.com/ce/ci/yaml/README.html)


~~~
image: repo.upyun.com:5043/ci-shared-runner

services:
    - mysql:latest
    - redis

variables:
    # Configure mysql environment variables (https://hub.docker.com/_/mysql/)
    MYSQL_DATABASE: upyun
    MYSQL_ROOT_PASSWORD: upyun
    MYSQL_USER: runner
    MYSQL_PASSWORD: runner123456

before_script:
    - echo "tcppm 6379 redis 6379" > /tmp/3proxy.cfg
    - echo "tcppm 3306 mysql 3306" >> /tmp/3proxy.cfg
    - 3proxy /tmp/3proxy.cfg &

test:
    script:
        - make dev
        - make start
        - make test
~~~

以上是一个 UPYUN ngx_lua 项目常用的 .gitlab-ci.yml，使用的执行器为 Docker 。`image`: Docker 镜像，例子中指定的是 UPYUN 私有仓库的镜像，也可以指定 [Docker Hub](http://hub.docker.com/) 的镜像。`services`: 第三方服务镜像，`mysql:latest` 中 mysql 是 Docker 镜像名，lastest 是版本号，第三方服务镜像也可以指定私有仓库镜像。`variables`: 环境变量，例子中设置了 mysql 的数据库名，帐号密码等。`before_scripts` 在执行 Job 前执行的一段代码，例子中配置了 3proxy，并将其启动，3proxy 后面会详细说明。`test`: Job 名，每一个 Job 都会有 scripts 字段，用来定义执行命令。

~~~
build-shared:
    stage: build
    script:
        - ver=$(cat .git/refs/remotes/origin/master | cut -c1-8) && echo $ver
        - cd shared && for i in {1..3} ; do (docker build -t ci-shared-runner:$ver .) && break; done
        - docker tag -f ci-shared-runner:$ver repo.upyun.com:5043/ci-shared-runner:$ver
        - docker tag -f ci-shared-runner:$ver repo.upyun.com:5043/ci-shared-runner
        - docker push repo.upyun.com:5043/ci-shared-runner:$ver
        - docker push repo.upyun.com:5043/ci-shared-runner
    only:
        - master

~~~

本例中，使用的执行器是 shell。`build-shared`: Job 名，`stage` 用于指定执行阶段，Runner 会按 stages 的顺序进行执行，本例中没有指定 stages，因此这个参数是无效的。`scripts` 上一例子中已经提及。`only`: 限定条件，例子中仅当 master 分支上有提交才触发 build-shared。

### 如何优美地使用 docker 执行器

> 使用 「docker 执行器」，会遇到一些问题，但是使用 Docker 利大于弊，这边给出两个常见问题的解决方案

#### localhost:port

用过 Docker 的都知道，Docker 在处理容器间通信是采用 link 的方式，不同容器意味着不同的 ip 地址，这对项目测试是极其不利的。比如，一个依赖 redis 的项目，习惯性的，我们会将测试的配置文件写成 localhost，但是如果放到「docker 执行器」中，这一套是否可行呢？

- 在「docker 执行器」中预装好 redis，在跑测试前将其启动。优点，简单，无需调整代码；缺点，把「docker 执行器」做的过于庞大累赘，「docker 执行器」应该尽可能的通用轻量。
- 采用 services 的方式来配置所需的 redis。优点，GitLab-Multi-Runner 原生，采用增加 Docker 容器的方式，配置方便，拓展性好；缺点，需要调整代码，将 localhost 改写成 redis，而且对于那些不支持域名解析的就麻烦了。

第一种方法，弊大于益，第二种方法，需要解决 localhost 到 redis 的映射关系。在 oneoo 的提示下，使用 [3proxy](https://www.3proxy.ru/) 完美的解决了这个问题。

#### C/C++ 编译速度

项目中有 C/C++ 项目依赖，而很多 C/C++ 项目编译过程格外漫长，这十分影响测试效率。如何解决这个问题呢？我们选择 ccache，ccache 是通过缓存编译过程中的中间件，起到二次编译加速的目的。ccache 默认会将缓存的文件放在 ~/.ccache 下。但是每次测试任务，Runner 都是启动一个全新的容器，因此，~/.ccache 不会被保存。[advanced-configuration.md](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/configuration/advanced-configuration.md) 中提及， 可以设置数据卷挂载位置，查看 Runner 的配置文件 config.toml 发现仅需将文件放在 /cache/ 下便可保证可持久化存储。

### 让事情变得更简单

为了方便管理 Runner 使用的镜像，我创建了一个 GitLab 项目。但是每次有修改或者新增镜像，都需要在本地将代码 pull 下来，然后各自 build 镜像，push 到 UPYUN 私有仓库中。这是一件很原始的体力活。

有了 .gitlab-ci.yml，我们就可以这么做，每当 master 分支上新的提交或者有新的 tag 创建，就触发 build，将镜像 build 好后直接 push 到 UPYUN 私有仓库。由于 `docker build` 的过程是缓存的，因此，构建效率是相当高的。OK，这件事情就完美的解决了。其实这就是第二个 .gitlab-ci.yml 做的事情。

