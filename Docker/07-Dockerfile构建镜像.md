# Dockerfile构建镜像

> 官方文档：https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
> 参考：https://www.runoob.com/docker/docker-dockerfile.html

## 什么是Dockerfile？
  Docker 通过从 Dockerfile 中读取指令来自动构建镜像。
  Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。
Dockerfile 遵循特定的格式和指令集，可以在 [Dockerfile 参考](https://docs.docker.com/engine/reference/builder/)中找到这些指令。
一个 Docker 镜像由只读层组成，每个层代表一个 Dockerfile 指令。这些层是堆叠的，每一层都是前一层变化的增量。
例如这个 Dockerfile：
```
# syntax=docker/dockerfile:1
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```
Dockerfile 的指令每执行一次都会新建一层：
* `FROM` 从 ubuntu:18.04 Docker 镜像创建一个层。
* `COPY` 从 Docker 客户端所在当前目录添加文件到镜像中。
* `RUN` 使用 make 构建编译应用程序。
* `CMD` 指定要在容器内运行的命令。

  当运行一个镜像并生成一个容器时，你会在底层的顶部添加一个新的可写层（“容器层”）。对正在运行的容器所做的所有更改，例如写入新文件、修改现有文件和删除文件，都将写入此可写容器层。
## Dockerfile构建镜像
### 1）准备好Dockerfile
> **注意**：Dockerfile 的指令每执行一次都会新建一层,所以过多无意义的层，会造成镜像膨胀过大，可以以 && 符号连接命令，这样执行后，只会创建 1 层镜像。

以[nginx dockerfile](https://github.com/nginxinc/docker-nginx/blob/3a7105159a6c743188cb1c61e4186a9a59c025db/mainline/debian/Dockerfile)为例
```
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "update.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#
FROM debian:bullseye-slim

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

ENV NGINX_VERSION   1.21.5
ENV NJS_VERSION     0.7.1
ENV PKG_RELEASE     1~bullseye

RUN set -x \
# create nginx user/group first, to be consistent throughout docker variants
    && addgroup --system --gid 101 nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y gnupg1 ca-certificates \
    && \
    NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
    found=''; \
    for server in \
        hkp://keyserver.ubuntu.com:80 \
        pgp.mit.edu \
    ; do \
        echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
        apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
    apt-get remove --purge --auto-remove -y gnupg1 && rm -rf /var/lib/apt/lists/* \
    && dpkgArch="$(dpkg --print-architecture)" \
    && nginxPackages=" \
        nginx=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-xslt=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-geoip=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-image-filter=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-njs=${NGINX_VERSION}+${NJS_VERSION}-${PKG_RELEASE} \
    " \
    && case "$dpkgArch" in \
        amd64|arm64) \
# arches officialy built by upstream
            echo "deb https://nginx.org/packages/mainline/debian/ bullseye nginx" >> /etc/apt/sources.list.d/nginx.list \
            && apt-get update \
            ;; \
        *) \
# we're on an architecture upstream doesn't officially build for
# let's build binaries from the published source packages
            echo "deb-src https://nginx.org/packages/mainline/debian/ bullseye nginx" >> /etc/apt/sources.list.d/nginx.list \
            \
# new directory for storing sources and .deb files
            && tempDir="$(mktemp -d)" \
            && chmod 777 "$tempDir" \
# (777 to ensure APT's "_apt" user can access it too)
            \
# save list of currently-installed packages so build dependencies can be cleanly removed later
            && savedAptMark="$(apt-mark showmanual)" \
            \
# build .deb files from upstream's source packages (which are verified by apt-get)
            && apt-get update \
            && apt-get build-dep -y $nginxPackages \
            && ( \
                cd "$tempDir" \
                && DEB_BUILD_OPTIONS="nocheck parallel=$(nproc)" \
                    apt-get source --compile $nginxPackages \
            ) \
# we don't remove APT lists here because they get re-downloaded and removed later
            \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
# (which is done after we install the built packages so we don't have to redownload any overlapping dependencies)
            && apt-mark showmanual | xargs apt-mark auto > /dev/null \
            && { [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; } \
            \
# create a temporary local APT repo to install from (so that dependency resolution can be handled by APT, as it should be)
            && ls -lAFh "$tempDir" \
            && ( cd "$tempDir" && dpkg-scanpackages . > Packages ) \
            && grep '^Package: ' "$tempDir/Packages" \
            && echo "deb [ trusted=yes ] file://$tempDir ./" > /etc/apt/sources.list.d/temp.list \
# work around the following APT issue by using "Acquire::GzipIndexes=false" (overriding "/etc/apt/apt.conf.d/docker-gzip-indexes")
#   Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
#   ...
#   E: Failed to fetch store:/var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages  Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
            && apt-get -o Acquire::GzipIndexes=false update \
            ;; \
    esac \
    \
    && apt-get install --no-install-recommends --no-install-suggests -y \
                        $nginxPackages \
                        gettext-base \
                        curl \
    && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/nginx.list \
    \
# if we have leftovers from building, let's purge them (including extra, unnecessary build deps)
    && if [ -n "$tempDir" ]; then \
        apt-get purge -y --auto-remove \
        && rm -rf "$tempDir" /etc/apt/sources.list.d/temp.list; \
    fi \
# forward request and error logs to docker log collector
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
# create a docker-entrypoint.d directory
    && mkdir /docker-entrypoint.d

COPY docker-entrypoint.sh /
COPY 10-listen-on-ipv6-by-default.sh /docker-entrypoint.d
COPY 20-envsubst-on-templates.sh /docker-entrypoint.d
COPY 30-tune-worker-processes.sh /docker-entrypoint.d
ENTRYPOINT ["/docker-entrypoint.sh"]

EXPOSE 80

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
```
### 2）构建镜像
在 Dockerfile 文件的存放目录下，执行构建动作。
以下示例，通过目录下的 Dockerfile 构建一个 mynginx:v1（镜像名称:镜像标签）。
> **注意** ：最后的 `.` 代表本次docker 构建的上下文路径
> **上下文路径**：是指 docker 在构建镜像，有时候想要使用到本机的文件（比如复制），docker build 命令得知这个路径后，会将路径下的所有内容打包。
> 由于 docker 的运行模式是 C/S。我们本机是 Client，docker engine是 Server。实际的构建过程是在 docker engine下完成的，所以这个时候无法用到我们本机的文件。这就需要把我们本机的指定目录下的文件一起打包提供给 docker engine使用。
> 如果未说明最后一个参数，那么默认上下文路径就是 Dockerfile 所在的位置。

> **注意**：上下文路径下不要放无用的文件，因为会一起打包发送给 docker engine，如果文件过多会造成过程缓慢。
```
$ docker build -t mynginx:v1 .
$ docker images //可以看到我们新建的镜像
```

## 构建上下文
> 原文链接：https://blog.csdn.net/qianghaohao/article/details/87554255
###1）理解Docker架构
  Docker 是一个典型的 C/S 架构的应用，Docker 客户端通过 REST API 和服务端进行交互，docker 客户端每发送一条指令，底层都会转化成 REST API 调用的形式发送给服务端，服务端处理客户端发送的请求并给出响应。

  Docker 镜像的构建、容器创建、容器运行等工作都是 Docker 服务端来完成的，Docker 客户端只是承担发送指令的角色。

Docker 客户端和服务端可以在同一个宿主机，也可以在不同的宿主机，如果在同一个宿主机的话，Docker 客户端默认通过 UNIX 套接字(`/var/run/docker.sock`)和服务端通信。

### 2）docker build 构建镜像的流程
* 执行 `docker build -t <imageName:imageTag> .` ;
* Docker 客户端会将构建命令后面指定的路径(`.`)下的所有文件打包成一个 tar 包，发送给 Docker 服务端;
* Docker 服务端收到客户端发送的 tar 包，然后解压，根据 Dockerfile 里面的指令进行镜像的分层构建；
### 3）正确理解 Docker 构建上下文
了解了 Docker 的架构和镜像构建的工作原理后，Docker 构建上下文也就容易理解了。Docker 构建上下文就是 Docker 客户端上传给服务端的 tar 文件解压后的内容，也即 docker build 命令行后面指定路径下的文件。

Docker 镜像的构建是在远程服务端进行的，所以客户端需要把构建所需要的文件传输给服务端。服务端以客户端发送的文件为上下文，也就是说 Dockerfile 中指令的工作目录就是服务端解压客户端传输的 tar 包的路径。
关于 docker build 指令的几点重要的说明：
* 如果构建镜像时没有明确指定 Dockerfile，那么 Docker 客户端默认在构建镜像时指定的上下文路径下找名字为 Dockerfile 的构建文件；
* Dockerfile 可以不在构建上下文路径下，此时需要构建时通过 `-f` 参数明确指定使用哪个构建文件，并且名称可以自己任意命名。
例如：
（1）创建一个目录并 cd 到其中。将“hello”写入名为 hello 的文本文件，并创建一个在其上运行 cat 的 Dockerfile。从构建上下文 (`.`) 中构建镜像：
```
$ mkdir myproject && cd myproject
$ echo "hello" > hello
$ echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
$ cat Dockerfile
FROM busybox
COPY /hello /
RUN cat /hello
$ docker build -t helloapp:v1 .
Step 1/3 : FROM busybox
latest: Pulling from library/busybox
5cc84ad355aa: Pull complete 
Digest: sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678
Status: Downloaded newer image for busybox:latest
 ---> beae173ccac6
Step 2/3 : COPY /hello /
 ---> 8382fd6068e2
Step 3/3 : RUN cat /hello
 ---> Running in e4fc688667ba
hello
Removing intermediate container e4fc688667ba
 ---> 07ffdf91d1e1
```
（2）将 Dockerfile 和 hello 移动到单独的目录中并构建镜像的第二个版本（不依赖于上次构建的缓存）。使用` -f` 指向 Dockerfile 并指定构建上下文的目录：
 ```
$ pwd
/root/myproject
$  mkdir -p dockerfiles context
$  mv Dockerfile dockerfiles && mv hello context
$  docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
Sending build context to Docker daemon  2.602kB
Step 1/3 : FROM busybox
 ---> beae173ccac6
Step 2/3 : COPY /hello /
 ---> 3aed1a85ba6f
Step 3/3 : RUN cat /hello
 ---> Running in 38f88ebf7ca7
hello
Removing intermediate container 38f88ebf7ca7
 ---> 89ff91f49f0d
Successfully built 89ff91f49f0d
Successfully tagged helloapp:v2
 ```
> *注意*：上下文路径下不要放无用的文件，因为会一起打包发送给 docker 服务端，如果文件过多会增加构建镜像的时间、拉取和推送镜像的时间以及容器运行时的大小。要查看构建上下文有多大，请在构建 Dockerfile 时查找如下消息：
> Sending build context to Docker daemon  187.8MB

## Dockerfile指令
#### COPY
复制指令，从上下文路径中复制文件或者目录到容器里指定路径。
```
COPY [--chown=<user>:<group>] <源路径1>...  <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",...  "<目标路径>"]
[--chown=<user>:<group>]：可选参数，用户改变复制到容器内文件的拥有者和属组。

<源路径>：源文件或者源目录，这里可以是通配符表达式，其通配符规则要满足 Go 的 filepath.Match 规则。
例如：
COPY hom* /mydir/
COPY hom?.txt /mydir/
<目标路径>：容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建。
```
####ADD
ADD 指令和 COPY 的使用格类似（同样需求下，官方推荐使用 COPY）。功能也类似，不同之处如下：
* ADD 的优点：在执行 <源文件> 为 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，会自动复制并解压到 <目标路径>。
* ADD 的缺点：在不解压的前提下，无法复制 tar 压缩文件。会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。具体是否使用，可以根据是否需要自动解压来决定。
####CMD
类似于 RUN 指令，用于运行程序，但二者运行的时间点不同:
* CMD 在docker run 时运行。
* RUN 是在 docker build。

**作用**：为启动的容器指定默认要运行的程序，程序运行结束，容器也就结束。CMD 指令指定的程序可被 docker run 命令行参数中指定要运行的程序所覆盖。
> 注意：如果 Dockerfile 中如果存在多个 CMD 指令，仅最后一个生效。
```
格式：
CMD <shell 命令> 
CMD ["<可执行文件或命令>","<param1>","<param2>",...] 
CMD ["<param1>","<param2>",...]  # 该写法是为 ENTRYPOINT 指令指定的程序提供默认参数
```
推荐使用第二种格式，执行过程比较明确。第一种格式实际上在运行的过程中也会自动转换成第二种格式运行，并且默认可执行文件是 `sh`。
####ENTRYPOINT
类似于 CMD 指令，但其不会被 docker run 的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序。
但是, 如果运行 docker run 时使用了 `--entrypoint` 选项，将覆盖 CMD 指令指定的程序。
* 优点：在执行 docker run 的时候可以指定 ENTRYPOINT 运行所需的参数。
> 注意：如果 Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效。
```
格式：
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]
可以搭配 CMD 命令使用：一般是变参才会使用 CMD ，这里的 CMD 等于是在给 ENTRYPOINT 传参，以下示例会提到。
```
示例：
```
# Dockerfile
FROM nginx
ENTRYPOINT ["nginx", "-c"] # 定参
CMD ["/etc/nginx/nginx.conf"] # 变参 
# 构建nginx:test镜像
$ docker build -t nginx:test .

#1、不传参运行
$ docker run nginx:test #此时容器内会默认运行 nginx -c /etc/nginx/nginx.conf 命令，启动主进程。

#2、传参运行
$ docker run nginx:test -c /etc/nginx/new.conf #此时容器内会默认运行 nginx -c /etc/nginx/new.conf 命令，启动主进程(/etc/nginx/new.conf:假设容器内已有此文件)
```
#### ENV
设置环境变量，定义了环境变量，那么在后续的指令中，就可以使用这个环境变量。
```
格式：
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```
以下示例设置 NODE_VERSION = 7.2.0 ， 在后续的指令中可以通过 $NODE_VERSION 引用：
```
ENV NODE_VERSION 7.2.0

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc"
```
#### ARG
构建参数，与 ENV 作用一致。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就是说只有 docker build 的过程中有效，构建好的镜像内不存在此环境变量。
构建命令 docker build 中可以用 `--build-arg <参数名>=<值> `来覆盖。
```
格式：
ARG <参数名>[=<默认值>]
```
#### VOLUME
定义匿名数据卷。在启动容器时忘记挂载数据卷，会自动挂载到匿名卷。
**作用**
* 避免重要的数据，因容器重启而丢失，这是非常致命的。
* 避免容器不断变大。
```
格式：
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```
在启动容器 docker run 的时候，我们可以通过` -v `参数修改挂载点。
####EXPOSE
仅仅只是声明端口。
**作用**: 帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射。
在运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 EXPOSE 的端口。
```
格式：
EXPOSE <端口1> [<端口2>...]
```
#### WORKDIR
WORKDIR 指令为 Dockerfile 中跟随它的任何 RUN、CMD、ENTRYPOINT、COPY 和 ADD 指令设置工作目录。如果 WORKDIR 不存在，即使它没有在任何后续 Dockerfile 指令中使用，它也会被创建。
```
格式：
WORKDIR <工作目录路径>
```
WORKDIR 指令可以在 Dockerfile 中多次使用。如果提供了相对路径，它将相对于前一个 WORKDIR 指令的路径。例如：
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
此 Dockerfile 中最终 pwd 命令的输出将是 /a/b/c。
> 注意：为了清晰和可靠，应该始终为 WORKDIR 设置绝对路径。此外，应该使用 WORKDIR 而不是像 RUN cd … && do-some 这样的增殖指令，这些指令难以阅读、故障排除和维护。

####USER
用于指定执行后续命令的用户和用户组，这边只是切换后续命令执行的用户（用户和用户组必须提前已经存在）。
```
格式：
USER <用户名>[:<用户组>]
```

#### HEALTHCHECK
用于指定某个程序或者指令来监控 docker 容器服务的运行状态。
```
格式：
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
HEALTHCHECK [选项] CMD <命令> : 这边 CMD 后面跟随的命令使用，可以参考 CMD 的用法。

选项：
--interval=DURATION (default: 30s)
--timeout=DURATION (default: 30s)
--start-period=DURATION (default: 0s)
--retries=N (default: 3)
```
例如，每五分钟左右检查一次网络服务器是否能够在三秒钟内为网站的主页提供服务：
```
HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
```

#### ONBUILD
用于延迟构建命令的执行。简单的说，就是 Dockerfile 里用 ONBUILD 指定的命令，在本次构建镜像的过程中不会执行（假设镜像为 test-build）。当有新的 Dockerfile 使用了之前构建的镜像 FROM test-build ，这时执行新镜像的 Dockerfile 构建时候，会执行 test-build 的 Dockerfile 里的 ONBUILD 指定的命令。
```
格式：
ONBUILD <其它指令>
```
例如：
```
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
```

#### LABEL
LABEL 指令用来给镜像添加一些元数据（metadata），以键值对的形式，语法格式如下：
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
# 比如我们可以添加镜像的作者：
LABEL authors="lixu"
```