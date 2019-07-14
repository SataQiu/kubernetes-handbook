# Kubelet 垃圾回收

Kubelet 垃圾回收（Garbage Collection）是一个非常有用的功能，它负责自动清理节点上的无用镜像和容器。Kubelet 每隔 1 分钟进行一次容器清理，每隔 5 分钟进行一次镜像清理（截止到 v1.15 版本，垃圾回收间隔时间还都是在源码中固化的，不可自定义配置）。如果节点上已经运行了 Kubelet，不建议再额外运行其它的垃圾回收工具，因为这些工具可能错误地清理掉 Kubelet 认为本应保留的镜像或容器，从而可能造成不可预知的问题。

<!-- more -->

## 镜像回收

Kubernetes 对节点上的所有镜像提供生命周期管理服务，这里的『所有镜像』是真正意义上的所有镜像，而不仅仅是通过 Kubelet 拉取的镜像。当磁盘使用率超过设定上限（`HighThresholdPercent`）时，Kubelet 就会按照 LRU 清除策略逐个清理掉那些没有被任何 Pod 容器（包括那些已经死亡的容器）所使用的镜像，直到磁盘使用率降到设定下限（`LowThresholdPercent`）或没有空闲镜像可以清理。此外，在进行镜像清理时，会考虑镜像的生存年龄，对于年龄没有达到最短生存年龄（`MinAge`）要求的镜像，暂不予以清理。

### 主体流程

![](/assets/image-gc-workflow.svg)

如上图所示，Kubelet 对于节点上镜像的回收流程还是比较简单的，在磁盘使用率超出设定上限后：首先，通过 CRI 容器运行时接口读取节点上的所有镜像以及 Pod 容器；然后，根据现有容器列表过滤出那些已经不被任何容器所使用的镜像；接着，按照镜像最近被使用时间排序，越久被用到的镜像越会被排在前面，优先清理；最后，就按照排好的顺序逐个清理镜像，直到磁盘使用率降到设定下限（或者已经没有空闲镜像可以清理）。

需要注意的是，Kubelet 读取到的镜像列表是节点镜像列表，而读取到的容器列表却仅包括由其管理的容器（即 Pod 容器，包括 Pod 内的死亡容器）。因此，那些用户手动 `run` 起来的容器，对于 Kubelet 垃圾回收来说就是不可见的，也就不能阻止对相关镜像的垃圾回收。当然，Kubelet 的镜像回收不是 force 类型的回收，虽然会对用户手动下载的镜像进行回收动作，但如果确实有运行的（或者停止的任何）容器与该镜像关联的话，删除操作就会失败（被底层容器运行时阻止删除）。

### 用户配置

通过上面的分析，我们知道影响镜像垃圾回收的关键参数有：

`image-gc-high-threshold`：磁盘使用率上限，有效范围 [0-100]，默认 `85`

`image-gc-low-threshold`：磁盘使用率下限，有效范围 [0-100]，默认 `80`

`minimum-image-ttl-duration`：镜像最短应该生存的年龄，默认 `2` 分钟

### 实验环节

本节我们通过实验来验证镜像垃圾回收（基于 Kubelet 1.15 版本）。

实验前，需要配置 Kubelet 启动参数，降低磁盘使用率上限，以便能够直接触发镜像回收。

```sh
# vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
...
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --image-gc-high-threshold=2 --image-gc-low-threshold=1
...
```

我们在 Kubelet 启动参数的最后追加了 `--image-gc-high-threshold=2 --image-gc-low-threshold=1`，这么低的配置，Kubelet 应该会一直忙于进行镜像回收了，生产环境可不能这么配置！

执行以下命令使得配置生效：

```sh
# systemctl daemon-reload
# systemctl restart kubelet
```

首先，看下本地都有哪些镜像：

```sh
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```

接下来，我们运行一个 `nginx` 程序，让 Kubelet 自动拉取镜像。

```sh
root@shida-machine:~# kubectl run nginx --image=nginx
deployment.apps/nginx created
root@shida-machine:~# kubectl get deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           62s
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
nginx                                latest              f68d6e55e065        12 days ago         109MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```
可以看到，`nginx` 镜像已经被自动 `pull` 到本地了，ID 为 `f68d6e55e065`。

然后，删除 `nginx` Deployment：

```sh
root@shida-machine:~# kubectl delete deployment nginx
deployment.extensions "nginx" deleted
```

过大概 5 分钟后，再次检查本地镜像列表，发现 `nginx` 镜像已被清理！

```sh
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```

通过以下命令查看镜像垃圾回收日志：

```sh
root@shida-machine:~# journalctl -u kubelet -o cat | grep imageGCManager
...
I0714 18:03:20.883489   51179 image_gc_manager.go:300] [imageGCManager]: Disk usage on image filesystem is at 24% which is over the high threshold (2%). Trying to free 72470076620 bytes down to the low threshold (1%).
I0714 18:03:20.899370   51179 image_gc_manager.go:371] [imageGCManager]: Removing image "sha256:f68d6e55e06520f152403e6d96d0de5c9790a89b4cfc99f4626f68146fa1dbdc" to free 109357355 bytes
```
可以看到，日志中记录的删除镜像 ID 与 `nginx` 镜像的 ID 是一致的（均为 `f68d6e55e065`）。

继续验证用户手动拉取的镜像是否会被清理，手动运行 `nginx` 程序：

```sh
root@shida-machine:~# docker run --name nginx -d nginx 
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
fc7181108d40: Pull complete 
d2e987ca2267: Pull complete 
0b760b431b11: Pull complete 
Digest: sha256:48cbeee0cb0a3b5e885e36222f969e0a2f41819a68e07aeb6631ca7cb356fed1
Status: Downloaded newer image for nginx:latest
2fc8a836ba3c7cbd488c7fd4f2ffa7287b709abf1b7701685291c3b1e5df3472
```

通过查看镜像 GC 日志，会发现 GC 会尝试清理用户自己手动拉取的 `nginx` 镜像，但因为该镜像被使用中，所以这次删除操作不会成功：

```sh
root@shida-machine:~# journalctl -u kubelet -o cat | grep imageGCManager
...
I0714 18:28:23.015586   51179 image_gc_manager.go:300] [imageGCManager]: Disk usage on image filesystem is at 24% which is over the high threshold (2%). Trying to free 72501525708 bytes down to the low threshold (1%).
I0714 18:28:23.306696   51179 image_gc_manager.go:371] [imageGCManager]: Removing image "sha256:f68d6e55e06520f152403e6d96d0de5c9790a89b4cfc99f4626f68146fa1dbdc" to free 109357355 bytes
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
nginx                                latest              f68d6e55e065        12 days ago         109MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```

将该容器停止，继续观察回收动作：

```sh
root@shida-machine:~# docker stop nginx
nginx
root@shida-machine:~# journalctl -u kubelet -o cat | grep imageGCManager
...
I0714 18:53:23.579629   51179 image_gc_manager.go:300] [imageGCManager]: Disk usage on image filesystem is at 24% which is over the high threshold (2%). Trying to free 72549280972 bytes down to the low threshold (1%).
I0714 18:53:23.629492   51179 image_gc_manager.go:371] [imageGCManager]: Removing image "sha256:f68d6e55e06520f152403e6d96d0de5c9790a89b4cfc99f4626f68146fa1dbdc" to free 109357355 bytes
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
nginx                                latest              f68d6e55e065        12 days ago         109MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```

可以看到，对于已经停止的容器，Kubelet 也是会尝试删除，但删除操作依然不会成功（存在死亡容器对该镜像的引用）。

彻底删除 `nginx` 容器，此时就没有任何容器继续使用该镜像，经过 1 次 GC 后，`nginx` 镜像就会被清理。

```sh
root@shida-machine:~# docker rm nginx
nginx
root@shida-machine:~# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.4             5f2081c22306        6 days ago          82.1MB
k8s.gcr.io/kube-apiserver            v1.14.4             f3171d49fa9b        6 days ago          210MB
k8s.gcr.io/kube-controller-manager   v1.14.4             35f0904dc8fa        6 days ago          158MB
k8s.gcr.io/kube-scheduler            v1.14.4             ee080c083e45        6 days ago          81.6MB
calico/node                          v3.7.3              bf4ff15c9db0        4 weeks ago         156MB
calico/cni                           v3.7.3              1a6ade52d471        4 weeks ago         135MB
calico/kube-controllers              v3.7.3              283860d96794        4 weeks ago         46.8MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        19 months ago       742kB
```

## 容器回收

了解了镜像回收的基本原理，我们再来看看容器回收。容器在停止运行（比如出错退出或者正常结束）后会残留一系列的垃圾文件，一方面会占据磁盘空间，另一方面也会影响系统运行速度。此时，就需要 Kubelet 容器回收了。要特别注意的是，Kubelet 回收的容器是指那些由其管理的的容器（也就是 Pod 容器），用户手动运行的容器不会被 Kubelet 进行垃圾回收。

与容器垃圾回收相关的控制参数主要有 3 个：

`MinAge`：容器可以被执行垃圾回收的最小年龄

`MaxPerPodContainer`：每个 pod 内允许存在的死亡容器的最大数量

`MaxContainers`：节点上全部死亡容器的最大数量

>注意：当 `MaxPerPodContainer` 与 `MaxContainers` 发生冲突时，Kubelet 会自动调整 `MaxPerPodContainer` 的取值以满足 `MaxContainers` 要求。

### 主体流程

![](/assets/container-gc-workflow.svg)

容器回收主要针对三个目标资源：普通容器、sandbox 容器以及容器日志目录。

对于普通容器，主要根据 `MaxPerPodContainer` 与 `MaxContainers` 的设置，按照 LRU 策略，从 Pod 的死亡容器列表删除一定数量的容器，直到满足配置需求；对于 `sandbox` 容器，按照每个 Pod 保留一个的原则清理多余的死亡 `sandbox`；对于日志目录，只要没有 Pod 与之关联了就将其删除。

Kubelet 的容器垃圾回收只针对 Pod 容器，非 Kubelet Pod 容器（比如通过 `docker run` 启动的容器）不会被主动清理。

### 用户配置

影响容器垃圾回收的关键参数有：

`minimum-container-ttl-duration`：容器可被回收的最小生存年龄，默认是 `0` 分钟，这意味着每个死亡容器都会被立即执行垃圾回收

`maximum-dead-containers-per-container`：每个 Pod 要保留的死亡容器的最大数量，默认值为 `1`

`maximum-dead-containers`：节点可保留的死亡容器的最大数量，默认值是 `-1`，这意味着节点没有限制死亡容器数量


### 实验环节

还是以 `nginx` 为例，创建一个 `nginx` 服务：

```sh
root@shida-machine:~# kubectl run nginx --image nginx
deployment.apps/nginx created
root@shida-machine:~# docker ps -a | grep nginx
7bef0308d9ea        nginx                     "nginx -g 'daemon of…"   16 seconds ago      Up 14 seconds                                 k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
7e65e0db52c2        k8s.gcr.io/pause:3.1      "/pause"                 2 minutes ago       Up 2 minutes                                  k8s_POD_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
```

可以看到，Kubelet 启动了一个 `sandbox` 以及一个 `nginx` 实例。

手动杀死 `nginx` 实例，模拟容器异常退出：

```sh
root@shida-machine:~# docker kill 7bef0308d9ea
7bef0308d9ea
root@shida-machine:~# docker ps -a | grep nginx
408b23b2b72a        nginx                     "nginx -g 'daemon of…"   3 seconds ago       Up 2 seconds                                      k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_1
7bef0308d9ea        nginx                     "nginx -g 'daemon of…"   2 minutes ago       Exited (137) 15 seconds ago                       k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
7e65e0db52c2        k8s.gcr.io/pause:3.1      "/pause"                 5 minutes ago       Up 5 minutes                                      k8s_POD_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
```

可以看到 Kubelet 重新拉起了一个新的 `nginx` 实例。

等待几分钟，发现 Kubelet 并未清理异常退出的 `nginx` 容器（因为此时仅有一个 dead container）。

```sh
root@shida-machine:~# docker ps -a | grep nginx
408b23b2b72a        nginx                     "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes                                     k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_1
7bef0308d9ea        nginx                     "nginx -g 'daemon of…"   5 minutes ago       Exited (137) 3 minutes ago                       k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
7e65e0db52c2        k8s.gcr.io/pause:3.1      "/pause"                 8 minutes ago       Up 8 minutes                                     k8s_POD_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
```

继续杀死当前 `nginx` 实例：

```sh
root@shida-machine:~# docker kill 408b23b2b72a
408b23b2b72a
root@shida-machine:~# docker ps -a | grep nginx
e064e376819f        nginx                     "nginx -g 'daemon of…"   9 seconds ago       Up 7 seconds                                      k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_2
408b23b2b72a        nginx                     "nginx -g 'daemon of…"   5 minutes ago       Exited (137) 40 seconds ago                       k8s_nginx_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_1
7e65e0db52c2        k8s.gcr.io/pause:3.1      "/pause"                 10 minutes ago      Up 10 minutes                                     k8s_POD_nginx-7db9fccd9b-p2p2t_default_69c38c2b-a64e-11e9-94bd-000c29ce064a_0
```

这下看到效果了，仍然只有一个退出的容器被保留，而且被清理掉的是最老的死亡容器，这与之前的分析是一致的！

删除这个 `nginx` Deployment，会发现所有的 `nginx` 容器都会被清理：

```sh
root@shida-machine:~# kubectl delete deployment nginx
deployment.extensions "nginx" deleted
root@shida-machine:~# docker ps -a | grep nginx
root@shida-machine:~# 
```

进一步，我们修改 Kubelet 参数，设置 `maximum-dead-containers` 为 `0`，这就告诉 Kubelet 清理所有死亡容器。

重复前边的实验步骤：

```sh
root@shida-machine:~# kubectl run nginx --image nginx
deployment.apps/nginx created
root@shida-machine:~# docker ps -a | grep nginx
8de9ae8e2c9b        nginx                     "nginx -g 'daemon of…"   33 seconds ago      Up 32 seconds                                   k8s_nginx_nginx-7db9fccd9b-jl2xn_default_0cd67a29-a6a2-11e9-94bd-000c29ce064a_0
d2cdfafdbe50        k8s.gcr.io/pause:3.1      "/pause"                 41 seconds ago      Up 38 seconds                                   k8s_POD_nginx-7db9fccd9b-jl2xn_default_0cd67a29-a6a2-11e9-94bd-000c29ce064a_0
root@shida-machine:~# docker kill 8de9ae8e2c9b
8de9ae8e2c9b
root@shida-machine:~# docker ps -a | grep nginx
95ee5bd2cab2        nginx                     "nginx -g 'daemon of…"   About a minute ago   Up About a minute                             k8s_nginx_nginx-7db9fccd9b-jl2xn_default_0cd67a29-a6a2-11e9-94bd-000c29ce064a_1
d2cdfafdbe50        k8s.gcr.io/pause:3.1      "/pause"                 2 minutes ago        Up About a minute                             k8s_POD_nginx-7db9fccd9b-jl2xn_default_0cd67a29-a6a2-11e9-94bd-000c29ce064a_0
```

结果显示，`nginx` Pod 的所有死亡容器都会被清理，因为我们已经强制要求节点不保留任何死亡容器，与预期一致！

那对于手动运行的容器呢？我们通过 `docker run` 运行 `nginx`：

```sh
root@shida-machine:~# docker run --name nginx -d nginx
46ebb365f6be060a6950f44728e4f11e4666bf2fb007cad557ffc65ecf8aded8
root@shida-machine:~# docker ps | grep nginx
46ebb365f6be        nginx                     "nginx -g 'daemon of…"   9 seconds ago       Up 6 seconds        80/tcp              nginx
```

杀死该容器：

```sh
root@shida-machine:~# docker kill 46ebb365f6be
46ebb365f6be
root@shida-machine:~# docker ps -a | grep nginx
46ebb365f6be        nginx                     "nginx -g 'daemon of…"   About a minute ago   Exited (137) 18 seconds ago                       nginx
```

经过几分钟，我们发现该死亡容器还是会存在的，Kubelet 不会清理这类容器！

## 小结

Kubelet 每 5 分钟进行一次镜像清理。当磁盘使用率超过上限阈值，Kubelet 会按照 LRU 策略逐一清理没有被任何容器所使用的镜像，直到磁盘使用率降到下限阈值或没有空闲镜像可以清理。Kubelet 认为镜像可被清理的标准是未被任何 Pod 容器（包括那些死亡了的容器）所引用，那些非 Pod 容器（如用户通过 `docker run` 启动的容器）是不会被用来计算镜像引用关系的。也就是说，即便用户运行的容器使用了 A 镜像，只要没有任何 Pod 容器使用到 A，那 A 镜像对于 Kubelet 而言就是可被回收的。但是我们无需担心手动运行容器使用的镜像会被意外回收，因为 Kubelet 的镜像删除是非 force 类型的，底层容器运行时会使存在容器关联的镜像删除操作失败（因为 Docker 会认为仍有容器使用着 A 镜像）。

Kubelet 每 1 分钟执行一次容器清理。根据启动配置参数，Kubelet 会按照 LRU 策略依次清理每个 Pod 内的死亡容器，直到达到死亡容器限制数要求，对于 `sandbox` 容器，Kubelet 仅会保留最新的（这不受 GC 策略的控制）。对于日志目录，只要已经没有 Pod 继续占用，就将其清理。对于非 Pod 容器（如用户通过 `docker run` 启动的容器）不会被 Kubelet 垃圾回收。