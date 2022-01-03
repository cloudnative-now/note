# Docker资源限制

> 官网：https://docs.docker.com/config/containers/resource_constraints/
## 资源限制介绍
默认情况下，容器没有资源限制，可以使用主机内核调度程序允许的尽可能多的给定资源，Docker 提供了控制容器可以限制容器使用多少内存或CPU 的方法，设置`docker run` 命令的运行时配置标志。

其中许多功能都要求宿主机的内核支持Linux 功能，要检查支持，可以使用`docker info` 命令，如果内核中禁用了某项功能，可能会在输出结尾处看到警告，
如下所示：

> WARNING: No swap limit support
## 内存资源限制介绍
  对于Linux 主机，如果没有足够的内存来执行其他重要的系统任务，将会抛出OOM (Out of Memory Exception,内存溢出、内存泄漏、内存异常), 随后系统会开始杀死进程以释放内存，凡是运行在宿主机的进程都有可能被kill，包括Dockerd和其它的应用程序，如果重要的系统进程被Kill,会导致和该进程相关的服务全部宕机。
![oom.png](https://upload-images.jianshu.io/upload_images/7897142-ee78685cc6c99e7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	产生OOM 异常时，Dockerd 尝试通过调整Docker 守护程序上的OOM 优先级来减轻这些风险，以便它比系统上的其他进程更不可能被杀死，但是容器的OOM优先级未调整，这使得单个容器被杀死的可能性比Docker 守护程序或其他系统进程被杀死的可能性更大， 不推荐通过在守护程序或容器上手动设置`--oom-score-adj` 为极端负数，或通过在容器上设置`--oom-kill-disable` 来绕过这些安全措施。

* `--oom-score-adj` ，宿主机kernel 对进程使用的内存进行评分，评分最高的将被宿
主机内核kill 掉，可以指定一个容器的评分制但是不推荐手动指定。
* ` --oom-kill-disable` ，对某个容器关闭oom 机制。
### OOM优先级机制
linux 会为每个进程算一个分数，最终它会将分数最高的进程kill。
* `/proc/PID/oom_score_adj` ，范围为-1000到1000，值越高越容易被宿主机kill掉，如果将该值设置为-1000，则进程永远不会被宿主机kernel kill。
* `/proc/PID/oom_adj` ，范围为-17 到+15，取值越高越容易被干掉，如果是-17，则表示不能被kill，该设置参数的存在是为了和旧版本的Linux 内核兼容。
* `/proc/PID/oom_score` ，这个值是系统综合进程的内存消耗量、CPU 时间(utime + stime)、存活时间(uptime - start time)和oom_adj 计算出的进程得分，消耗内存越多得分越高，越容易被宿主机kernel 强制杀死。
例如查看1019进程
```
root@ubuntu-xenial:~# cat /proc/1019/oom_score_adj 
0
root@ubuntu-xenial:~# cat /proc/1019/oom_adj 
0
root@ubuntu-xenial:~# cat /proc/1019/oom_score
0
```
## 容器的内存限制
​	Docker 可以强制执行硬性内存限制，即只允许容器使用给定的内存大小。Docker 也可以执行非硬性内存限制，即容器可以使用尽可能多的内存，除非内核检测到主机上的内存不够用了。这些选项中的大多数采用正整数，后跟 b、k、m、g 后缀，以表示字节、千字节、兆字节或千兆字节。

### 内存限制
* `-m or --memory` , 容器可以使用的最大内存量，如果设置此选项，则允许的最
小存值为4m （4 兆字节）。
* `--memory-swap` , 容器可以使用的交换分区大小，必须要在设置了物理内存限制的前提才能设置交换分区的限制
* `--memory-swappiness` , 设置容器使用交换分区的倾向性，值越高表示越倾向于使用swap 分区，范围为0-100，0 为能不用就不用，100 为能用就用。
* `--kernel-memory` , 容器可以使用的最大内核内存量，最小为4m，由于内核内存与用户空间内存隔离，因此无法与用户空间内存直接交换，因此内核内存不足的容器可能会阻塞宿主主机资源，这会对主机和其他容器或者其他服务进程产生影响，因此不要设置内核内存大小。
* `--memory-reservation` , 允许指定小于--memory 的软限制，当Docker 检测到主机上的争用或内存不足时会激活该限制，如果使用--memory-reservation，则必须将其设置为低于--memory 才能使其优先。因为它是软限制，所以不能保证容器不超过限制。
* `--oom-kill-disable` , 默认情况下，发生OOM 时，kernel 会杀死容器内进程，但是可以使用--oom-kill-disable 参数，可以禁止oom 发生在指定的容器上，即仅在已设置-m / - memory 选项的容器上禁用OOM，如果-m 参数未配置，产生OOM 时，主机为了释放内存还会杀死系统进程。
### swap限制
`--memory-swap` , 只有在设置了`--memory `后才会有意义。使用Swap,可以让容器将超出限制部分的内存置换到磁盘上，
> 注意：经常将内存交换到磁盘的应用程序会降低性能。

不同的`--memory-swap` 设置会产生不同的效果：
* `--memory-swap` 值为正数， 那么`--memory` 和`--memory-swap` 都必须要设置，`--memory-swap` 表示你能使用的内存和swap分区大小的总和，例如：
`--memory=300m, --memory-swap=1g`, 那么该容器能够使用300m内存和700m
swap，即`--memory` 是实际物理内存大小值不变，而swap 的实际大小计算方式为:
`(--memory-swap)-(--memory)=容器可用swap`。
* `--memory-swap` 设置为0，则忽略该设置，并将该值视为未设置，即未设置交换分区。
* `--memory-swap` 等于`--memory `的值，并且`--memory` 设置为正整数，容器无权访问swap即也没有设置交换分区。
* `--memory-swap` 设置为`unset`，如果宿主机开启了swap，则实际容器的swap值为`2*(--memory)`，即两倍于物理内存大小，但是并不准确(在容器中使用free 命令所看到的swap 空间并不精确，毕竟每个容器都可以看到具体大小，但是宿主机的swap 是有上限而且不是所有容器看到的累计大小)。
* `--memory-swap` 设置为`-1`，如果宿主机开启了swap，则容器可以使用主机上swap 的最大空间。
### 内存限制验证
假如一个容器未做内存使用限制，则该容器可以利用到系统内存最大空间，默认创建的容器没有做内存资源限制。
```
$ docker pull lorel/docker-stress-ng #测试镜像
$ docker run -it --rm lorel/docker-stress-ng --help #查看帮助信息
```
#### 1) 内存大小硬限制
启动两个工作进程，每个工作进程最大允许使用内存256M，且宿主机不限制当前容器最大内存：
```
$ docker run -it --name stree1  --rm lorel/docker-stress-ng -vm 2 --vm-bytes 256M
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
0813fad23ffa   stree1         198.66%   514.3MiB / 992MiB   51.84%    648B / 0B   414kB / 0B    5
```
接着宿主机限制容器最大内存使用：
```
$ docker run -it --name stree1 --memory 256m --rm lorel/docker-stress-ng -vm 2 --vm-bytes 256M
# 启动过程中会出现以下报错信息，由于两个工作进程启动所需内存超过宿主机限制的，所以一直会出现oom，容器进程会一直被kill
stress-ng: debug: [7] stress-ng-vm: child died: 9 (instance 0)
stress-ng: debug: [7] stress-ng-vm: assuming killed by OOM killer, restarting again (instance 0)
stress-ng: debug: [8] stress-ng-vm: child died: 9 (instance 1)
$ docker stats # 可以看到此时的USAGE / LIMIT
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
b630e0e65af5   stree1         177.26%   145.4MiB / 256MiB   56.81%    648B / 0B   246kB / 0B    5
```
宿主机cgroup 验证:
```
cat /sys/fs/cgroup/memory/docker/容器ID /memory.limit_imen_bytes
268435456 #宿主机基于cgroup 对容器进行内存资源的大小限制
```
> 注：通过echo 命令可以改内存限制的值，但是可以在原基础之上增大内存限制，缩小内存限制会报错write error: Device or resource busy
#### 2) 内存大小软限制
`--memory-reservation`
```
$ docker run -it --memory 256m --name stree1 --memory-reservation 127m --rm lorel/docker-stress-ng -vm 2 --vm-bytes 256M
```
宿主机cgroup 验证：
```
$ cat /sys/fs/cgroup/memory/docker/容器ID/memory.soft_limit_in_bytes
134217728 #返回的软限制结果
```
#### 3) 关闭OOM限制

```
$ docker run -it --memory 257m --name stree1 --oom-kill-disable --rm lorel/docker-stress-ng -vm 2 --vm-bytes 256M

$ docker stats # 此时可以看到容器进程不会因为oom被kill
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
0a865e5ac776   stree1         0.00%     257MiB / 257MiB     100.00%   648B / 0B   377kB / 0B    5
$ cat /sys/fs/cgroup/memory/docker/容器ID/memory.oom_control 
oom_kill_disable 1
under_oom 1
```
#### 4) 交换分区限制
```
$ docker run -it --rm -m 256m --memory-swap 512m centos bash
#宿主机cgroup 验证：
$ cat /sys/fs/cgroup/memory/docker/容器ID/memory.memsw.limit_in_bytes
536870912 #返回值
```
> K8S部署时，如果宿主机开启交换分区，会在安装之前的预检查环节提示相应错误信息

## 容器的CPU限制介绍
一个宿主机，有几十个核的CPU，但是宿主机上可以同时运行成百上千个不同的进程用以处理不同的任务，多进程共用一个CPU 的核心依赖计数就是为可压缩资源，即一个核心的CPU 可以通过调度而运行多个进程，但是同一个单位时间内只能有一个进程在CPU 上运行，那么这么多的进程怎么在CPU 上执行和调度的呢？
* 实时优先级：0-99
* 非实时优先级(nice)：-20-19，对应100-139 的进程优先级

Linux kernel 进程的调度基于CFS(Completely Fair Scheduler)，完全公平调度
* **CPU 密集型的场景**：优先级越低越好，计算密集型任务的特点是要进行大量的计算，消耗CPU 资源，比如计算圆周率、数据处理、对视频进行高清解码等等，全靠CPU 的运算能力。
* **IO 密集型的场景**：优先级值高点，涉及到网络、磁盘IO 的任务都是IO 密集型任务，这类任务的特点是CPU 消耗很少，任务的大部分时间都在等待IO 操作完成（因为IO 的速度远远低于CPU 和内存的速度），比如Web 应用，高并发、数据量大的动态网站来说，数据库应该为IO 密集型。
```
# 磁盘的调度算法
$ cat /sys/block/sda/queue/scheduler 
noop [deadline] cfq 
```
默认情况下，每个容器对主机CPU 周期的访问权限是不受限制的，但是我们可以设置各种约束来限制给定容器访问主机的CPU周期，大多数用户使用的是默认的CFS 调度方式，在Docker 1.13 及更高版本中，还可以配置实时优先级。
**参数**：
* `--cpus` ，指定容器可以使用多少可用CPU 资源，例如，如果主机有两个CPU，并且设置了`--cpus ="1.5"`，那么该容器将保证最多可以访问1.5 个的CPU(如果是4 核CPU，那么还可以是4 核心上每核用一点，但是总计是1.5 核心的CPU)，这相当于设置`--cpu-period ="100000"`和`--cpu-quota="150000"`，`--cpus` 主要在Docker 1.13 和更高版本中使用，目的是替代--cpu-period 和--cpu-quota 两个参数，从而使配置更简单，但是最大不能超出宿主机CPU 总核心数
```
# docker run -it --rm --cpus 2 centos bash
docker: Error response from daemon: Range of CPUs is from 0.01 to 1.00, as there are only 1 CPUs available.
See 'docker run --help'. #分配给容器的CPU 超出了宿主机CPU 总数。
```
### 测试CPU限制
#### 1) 未限制容器CPU
对于一台2核的服务器，如果不做限制，容器会把宿主机的CPU 全部占完：
```
#分配2核CPU 并启动2个工作线程
$ docker run -it --rm --name magedu-c1 lorel/docker-stress-ng --cpu 2 --vm 2
```
在宿主机使用`dokcer stats `命令查看容器运行状态：
```
#容器运行状态：
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
186306fafa33   stress1        195.97%   582.2MiB / 992MiB   58.69%    648B / 0B   307kB / 0B    7
```
在宿主机查看CPU 限制参数：
```
$ cat /sys/fs/cgroup/cpuset/docker/容器ID /cpuset.cpus
0-1
```
宿主机CPU 利用率：
```
top - 14:03:05 up  3:41,  2 users,  load average: 3.98, 2.25, 1.00
Tasks: 130 total,   5 running, 125 sleeping,   0 stopped,   0 zombie
%Cpu0  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 99.7 us,  0.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1015844 total,   157360 free,   692156 used,   166328 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   157200 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                
18003 root      20   0  268400 262244    296 R  49.8 25.8   1:54.42 stress-ng-vm                                                                                                                           
17998 root      20   0    6900   2096    268 R  49.5  0.2   1:54.64 stress-ng-cpu                                                                                                                          
18000 root      20   0    6900   2096    268 R  49.5  0.2   1:54.53 stress-ng-cpu                                                                                                                          
18002 root      20   0  268400 262244    296 R  49.5 25.8   1:54.63 stress-ng-vm        
```
####2) 限制容器CPU
只给容器分配最多1核宿主机CPU 利用率
```
$ docker run -it --rm --name stress1 --cpus 1 lorel/docker-stress-ng --cpu 2 --vm 2
```
宿主机cgroup 验证：
```
$ cat /sys/fs/cgroup/cpu,cpuacct/docker/容器ID/cpu.cfs_quota_us
100000
```
> 每核心CPU 会按照1000 为单位转换成百分比进行资源划分，2 个核心的CPU 就是200000/1000=200%，4 个核心400000/1000=400%，以此类推。

当前容器状态：
```
# 容器运行状态
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
b1206e802e6b   stress1        102.00%   582.2MiB / 992MiB   58.69%    648B / 0B   385kB / 0B    7
```
宿主机CPU利用率：
```
top - 14:14:18 up  3:52,  2 users,  load average: 2.78, 2.20, 1.62
Tasks: 129 total,   5 running, 124 sleeping,   0 stopped,   0 zombie
%Cpu0  : 51.2 us,  0.0 sy,  0.0 ni, 48.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 50.3 us,  0.0 sy,  0.0 ni, 49.3 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  1015844 total,   165048 free,   685704 used,   165092 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   164844 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                
18239 root      20   0    6900   2076    248 R  25.2  0.2   1:55.44 stress-ng-cpu                                                                                                                          
18241 root      20   0  268400 262192    244 R  25.2 25.8   1:55.42 stress-ng-vm                                                                                                                           
18237 root      20   0    6900   2076    248 R  24.9  0.2   1:55.33 stress-ng-cpu                                                                                                                          
18242 root      20   0  268400 262192    244 R  24.9 25.8   1:55.09 stress-ng-vm 
```
> 注：CPU 资源限制是将分配给容器的1核分配到了宿主机每一核CPU 上，也就是容器的总CPU 值是在宿主机的每一个核CPU 分配了部分比例。

####3) 将容器运行到指定的CPU 上
```
$ docker run -it --rm --name stress1 --cpus 1 --cpuset-cpus 0 lorel/docker-stress-ng --cpu 2 --vm 2
$ cat /sys/fs/cgroup/cpuset/docker/容器ID /cpuset.cpus
0
```
容器运行状态：
```
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
59370d489a91   stress1        100.50%   518.2MiB / 992MiB   52.24%    648B / 0B   319kB / 0B    7
```
宿主机CPU利用率：
```
top - 14:20:45 up  3:59,  2 users,  load average: 4.01, 2.73, 1.98
Tasks: 131 total,   5 running, 126 sleeping,   0 stopped,   0 zombie
%Cpu0  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  1015844 total,   128828 free,   690948 used,   196068 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   159620 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                
18491 root      20   0  268400 262152    208 R  25.2 25.8   0:38.27 stress-ng-vm                                                                                                                           
18492 root      20   0  268400 262152    208 R  25.2 25.8   0:38.26 stress-ng-vm                                                                                                                           
18487 root      20   0    6896   2048    228 R  24.8  0.2   0:38.26 stress-ng-cpu                                                                                                                          
18489 root      20   0    6896   2048    228 R  24.8  0.2   0:38.26 stress-ng-cpu
```
####4) 基于cpu-shares对cpu进行切分
启动两个容器，stress1 的 `--cpu-shares` 值为1000 ， stress2 的`--cpu-shares` 为500，观察最终效果，`--cpu-shares` 值为1000 的 stress1的CPU 利用率基本是`--cpu-shares` 为500 的 stress1的`2`倍：
```
$ docker run -it --rm --name stress1 --cpu-shares 1000 lorel/docker-stress-ng --cpu 2 --vm 2
$ docker run -it --rm --name stress2 --cpu-shares 500 lorel/docker-stress-ng --cpu 2 --vm 2
```
验证容器运行状态：
```
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
93c3ceaee639   stress2        56.60%    347.9MiB / 992MiB   35.07%    648B / 0B   26.8MB / 0B   7
d9923d15d05e   stress1        104.21%   173.6MiB / 992MiB   17.50%    648B / 0B   70.6MB / 0B   7
```
宿主机cgroups验证：
```
$ cat /sys/fs/cgroup/cpu,cpuacct/docker/容器ID/cpu.shares
1000
$ cat /sys/fs/cgroup/cpu,cpuacct/docker/容器ID/cpu.shares
500
```
**动态修改CPU shares的值**
`--cpu-shares` 的值可以在宿主机cgroup 动态修改，修改完成后立即生效，其值可以调大也可以减小。
```
# 将stress2的CPU shares设置为2000
$ echo 2000 > /sys/fs/cgroup/cpu,cpuacct/docker/容器ID/cpu.shares
```
验证修改后的容器运行状态：
```
$ docker stats
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O     PIDS
93c3ceaee639   stress2        131.96%   262.1MiB / 992MiB   26.43%    648B / 0B   244MB / 0B    7
d9923d15d05e   stress1        67.24%    262.4MiB / 992MiB   26.45%    648B / 0B   370MB / 0B    7
```