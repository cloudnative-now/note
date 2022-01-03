## dockerd、containerd、containerd-shim关系

> 参考：https://blog.kelu.org/tech/2020/10/09/the-diff-between-docker-containerd-runc-docker-shim.html

不同版本的docker，关系也不同，本机安装的是docker 20.10.7
```
$ docker info
...
 Server Version: 20.10.7
...

```
例如，启动一个nginx容器，将端口映射到80
```
$ docker run -d -p 80:80 nginx
$  docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS         PORTS                               NAMES
31821dbe5898   nginx     "/docker-entrypoint.…"   10 minutes ago   Up 9 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp   epic_babbage
```
用pstree查看进程树
```
$ pstree -a
systemd─
  ├─containerd
  │   └─10*[{containerd}]
  ├─containerd-shim -namespace moby -id 31821dbe5898d02c6543fb1d8e26e1b57fbe888a6ee06af82e31620cbbc32b20 -address /run/containerd/containerd.sock
  │   ├─nginx
  │   │   ├─nginx
  │   │   └─nginx
  │   └─12*[{containerd-shim}]
  ├─dockerd -H fd:// --containerd=/run/containerd/containerd.sock
  │   ├─docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.17.0.2 -container-port 80
  │   │   └─6*[{docker-proxy}]
  │   ├─docker-proxy -proto tcp -host-ip :: -host-port 80 -container-ip 172.17.0.2 -container-port 80
  │   │   └─5*[{docker-proxy}]
  │   └─12*[{dockerd}]

```
* `docker`，一个客户端工具，用来把用户的请求发送给 docker daemon(dockerd)。
* `dockerd, docker daemon`，一般也会被称为 docker engine。dockerd 启动时会启动 containerd 子进程。
* `Containerd` 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性，几乎囊括了单机运行一个容器运行时所需要的一切：执行，分发，监控，网络，构建，日志等。主要作用是：
  1. 管理容器的生命周期(从创建容器到销毁容器)
  2. 拉取/推送容器镜像
  3. 存储管理(管理镜像及容器数据的存储)
  4. 调用 runC 运行容器(与 runC 等容器运行时交互)
  5. 管理容器网络接口及网络
*  `containerd-shim`，为了能够支持多种 OCI Runtime，containerd 内部使用 containerd-shim，每启动一个容器都会创建一个新的 containerd-shim 进程，指定容器 ID，Bundle 目录，运行时的二进制（比如 runc）。
*  `RunC` 是一个轻量级的工具，用来运行容器的，我们可以不用通过 docker 引擎，直接运行容器。事实上，runC 是标准化的产物，它根据 OCI 标准来创建和运行容器。