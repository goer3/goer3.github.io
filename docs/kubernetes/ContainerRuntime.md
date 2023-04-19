## Docker

Docker 从 1.11 版本开始，其容器的运行就不再是简单的通过 Docker Daemon 来启动，而是通过集成 containerd、runc 等多个组件来协同完成。

* Docker Daemon：守护进程，属于 CS 架构，负责和 Docker Client 端交互，并管理 Docker 镜像和容器。

* Containerd：负责对集群节点上的容器生命周期进行管理，并为 Docker Daemon 提供 gRPC 接口。

工作原理架构图：

![img](images/ContainerRuntime/979767-20230111220608799-406970549.png)

<br>

目前 Docker 创建容器的流程主要包含以下 4 步：

1. 用户通过 Docker Daemon 提交容器创建请求。
2. Docker Daemon 请求 Containerd 发起创建。
3. Containerd 收到创建请求后，创建一个 Containerd-shim 进程去操作容器。这样做的原因在于：
   * 容器进程需要一个父进程做状态收集、维持 stdin 等工作，如果父进程直接使用 Containerd 进程，可能会出现因为 Containerd 进程挂掉，整个宿主机所有容器都跟着挂的情况。
   * 而 Containerd-shim 进程就是用来规避这个问题的父进程。
4. Containerd-shim 再通过 RunC 来进行容器的创建。使用 RunC 的原因在于：
   * 容器的创建需要遵循开放容器标准（`OCI`，规定了容器镜像的结构、以及容器需要接收哪些操作指令）做一些 Namespaces，Cgroups 以及挂载 root 文件系统等配置操作。
   * 而 RunC（Docker 捐献的 `libcontainer` 改名而来）则是该标准的一个参考实现。通过 RunC 可以创建一个符合规范的容器，除此之外，同类产品还有 `Kata`、`gVisor` 等。

<br>

整个过程总结起来其实就是：

> Containerd-shim 调用 RunC 创建容器，完成后 RunC 退出，Containerd-shim 进程成为容器的父进程（该进程在宿主机可以直接通过 ps -ef 看到），负责收集容器状态上报给 Containerd。并在容器中 PID 为 1 的进程退出后对容器子进程进行清理，避免出现僵尸进程。

这样看起来 Docker Daemon 完全是多余的东西，还增加了链路长度。

Docker 为了做 Swam 进军 PaaS 市场，将容器操作都迁移到 Containerd。又把 Containerd 项目捐献给了 `CNCF` 基金会（Cloud Native Computing Foundation，云原生计算基金会）。这也导致了因为 Docker Swarm 的失败，Docker 也逐渐被淘汰。



## CRI

Kubernetes 提供了一个叫做 CRI 的容器运行时接口，那么这个 CRI 到底是啥？

在 Kubernetes 早期，当时主流的容器就只有 Docker，所以 Kubernetes 是通过硬编码的方式直接调用 Docker API。

但随着容器技术的不断发展以及 Google 的主导，出现了更多的容器运行时。Kubernetes 为了支持更多更精简的容器运行时，Google 就和红帽主导推出了 CRI，用于将 Kubernetes 平台和特定的容器运行时解耦。

简单的归纳就是：

Google 和红帽联合发布了一个接口标准，大家自己的容器运行时如果想要接入 Kubernetes，就需要按照这个标准提供对应的接口。

<br>

`CRI`（Container Runtime Interface，容器运行时接口），就是 Kubernetes 定义的一组与容器运行时进行交互的接口。

CRI 的发布，意味着 Kubernetes 的角色由：去适配其它容器运行时的接口，到让别的容器运行时去遵循自己的接口标准的转变。

不过 CRI 发布的时候 Kubernetes 还没有现在的统治地位，很多容器运行时自身可能并不会去适配这些 CRI 接口，于是就有了各种各样 shim 垫片。其作用在于适配容器运行时本身的接口到 Kubernetes 的 CRI 接口上。

日常听到比较多的就是 `dockershim`，用于 Kubernetes 和 Docker 进行适配的 `shim 垫片`。其结构如下：

![img](images/ContainerRuntime/979767-20230111220641822-629660686.png)

<br>

CRI 定义的 API 主要包含两个 gRPC 服务：

* `ImageService`：镜像管理
* `RuntimeService`：容器运行时管理（容器和 Pod）

<br>

在 2022 年 5 月 3 日，`Kubernetes 1.24` 版本发布，正式移除了内置的 dockershim。从此时开始，新版本的 Kubernetes 还想要以 Docker 作为容器运行时，就需要单独安装适配器垫片 `cri-dockerd` 了。

下面是 1.24 版本之前的 Kubernetes 以 Docker 作为容器运行时的调用流程：

![img](images/ContainerRuntime/979767-20230111220730778-252260409.png)

可以发现整个调用链路是很长的，当在 Kubernetes 中想要创建一个 Pod 时：

1. 首先需要 kubelet 通过 CRI 接口调用自身集成的 dockershim。
2. dockershim 在收到请求后，转化成 Docker Daemon 能识别的请求并发给 Docker Daemon。
3. Docker Daemon 收到请求后，再执行上面提到的创建容器流程。

<br>

说到底，通过 Docker 转换这个流程完全就是多余，因为 Containerd 本身就是实现了 CRI 的服务。只是 Docker 在用户交互上做的跟友好而已。但其实在使用 Kubernetes 的时候，直接对于 Docker 的操作几乎很少。所以完全可以不需要它。

所以慢慢的整个创建流程就被精简了：

![img](images/ContainerRuntime/979767-20230111220741228-1735997539.png)

<br>

在 Containerd 1.1 之后，已经完全消除了中间环节，适配器 shim 也被继承到了 Contained 的主进程中，调用变得更加简洁。

当然用户依旧还是能够使用 Docker 镜像的，但前提是这些镜像必须都是镜像仓库中的。直接使用 Docker 打包在本地的镜像对于 Contained 来说是无法访问到的。

同时，Kubernetes 社区也做了个专门用于 Kubernetes 的容器运行时 `CRI-O`，直接兼容 CRI 和 OCI 规范。但是对于用户来说，Docker 还是更为熟悉，所以更多的还是选择 Containerd 作为容器运行时。

![img](images/ContainerRuntime/979767-20230111220753696-415289604.png)



## Containerd

Containerd 是从 Docker Engine 里分离出来的一个独立的开源项目，目标是提供一个更加开放、稳定的容器运行基础设施。分离出来的 Containerd 具有更多的功能，涵盖整个容器运行时管理的所有需求，提供更强大的支持。

作为一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性，可以负责下面这些事情：

* 管理容器的生命周期（从创建容器到销毁容器）
* 拉取/推送容器镜像
* 存储管理（管理镜像及容器数据的存储）
* 调用 RunC 运行容器（与 RunC 等容器运行时交互）
* 管理容器网络接口及网络

<br>

官方架构图：

![img](images/ContainerRuntime/979767-20230112213118576-1517267591.png)

可以看出 Containerd 采用的也是 C/S 架构，服务端通过 unix domain socket 暴露低层的 gRPC API 接口出去，客户端通过这些 API 管理节点上的容器，每个 Containerd 只负责一台机器。

为了解耦，Containerd 将系统划分成了不同的组件，每个组件都由一个或多个模块协作完成，每一种类型的模块都以插件的形式集成到 Containerd 中，而且插件之间是相互依赖的。

总体来看，Containerd 可以分为三个大块：`Storage`、`Metadata` 和 `Runtime`。

![img](images/ContainerRuntime/979767-20230112213137411-1462940565.png)





## 安装 Containerd（了解）

由于 Containerd 需要调用 RunC，所以需要先安装 RunC，不过 Containerd 提供了一个包含相关依赖的压缩包 cri-containerd-cni，可以直接使用这个包来进行安装。

本文安装所使用的是 `CentOS 7.9` 的 VM Ware 虚拟机。Containerd 最新版下载地址：

> https://github.com/containerd/containerd/releases

当前 cri-containerd-cni 安装包的最新版本是 `1.6.20`。

```bash
# 版本变量
CRI_CONTAINERD_CNI_VERSION="1.6.20"

# 安装依赖
yum -y install vim lrzsz wget zip unzip tree

# 更新 libseccomp，解决容器启动报错
rpm -e libseccomp-2.3.1-4.el7.x86_64 --nodeps
wget https://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm 

# 下载安装包
https://github.com/containerd/containerd/releases/download/v${CRI_CONTAINERD_CNI_VERSION}/cri-containerd-cni-${CRI_CONTAINERD_CNI_VERSION}-linux-amd64.tar.gz

# 直接解压到系统目录
tar -C / -zxf cri-containerd-cni-${CRI_CONTAINERD_CNI_VERSION}-linux-amd64.tar.gz

# 创建配置文件目录，并生成默认配置
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml

# 启动 Containerd
systemctl enable containerd --now

# 查看安装信息
ctr version
```

可以看到它的用法和 Docker 是有些类似的，但是在生产环境中一般不单独这样安装，而是选择功能更全的 `nerdctl`。









