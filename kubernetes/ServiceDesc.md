## 服务说明

Kubernetes 简称 `K8S`，源于希腊语，意为”舵手”或“飞行员”，由 Google 在 2014 年将自己的 `Borg` 系统使用 `Go` 语言重写开源而来。

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230221184802981-975689453.png )

其 logo 类型轮船的方向盘，而 docker logo 类似载满集装箱的轮船，寓意可想而知。

在 Docker 带火了容器技术之前，Google 内部已经有多年的容器使用管理经验，所以尽管 Docker 后续推出了 swarm，compose 等容器编排服务与工具，但不管在服务的完整性和稳定性都不及 Kubernetes，而今 swarm 几乎已经没人使用。

> https://kubernetes.io/zh-cn/

Kubernetes 主要包含如下几个重要特征：

1. 自动上线回滚
   - 分步骤地对应用进行上线，同时监测程序运行状况，以确保不会同时终止所有实例。如果上线发生异常，会自动回滚所作更改。
2. 服务发现与负载均衡
   - 无需修改程序即可使用服务发现机制。同时为容器提供了自己的 IP 和 DNS 名称，并在它们之间实现负载均衡。
3. 存储编排
   - 能自动挂载所选择的存储系统，如本地存储，云服务商提供的存储，其它网络存储，如 NFS，Ceph 等。
4. Secret 和配置管理
   - 部署、更新 Secret 和应用程序的配置而不必重新构建容器镜像。也不需要将软件堆栈配置中的重要信息暴露出来，提升安全性。
5. 自动装箱
   - 根据资源需求和其他限制自动放置容器，同时避免影响可用性。将关键性的和尽力而为性质的工作负载混合放置，提高资源利用率。
6. 批量执行
   - 除了服务外，还可以管理批处理和 CI 工作负载，在期望时替换掉失效的容器。
7. IPv4/IPv6 双协议栈
   - 为 Pod 和 Service 分配 IPv4 和 IPv6 地址。
8. 水平扩缩
   - 使用一个简单的命令、一个 UI 或基于 CPU 使用情况自动对应用程序进行扩缩。
9. 自我修复
   - 自动重启失败的容器，在节点死亡时替换并重新调度容器。杀死不符合健康检查的容器，并在它们准备好服务之前不将它们公布给客户端。
10. 为扩展性设计
    - 无需更改上游源码即可扩展集群。





## 版本说明

Kubernetes 的版本号格式为：`x.y.z`

其中 x 为大版本号，y 为小版本号，z 为补丁版本号。Kubernetes 项目会维护最新的三个小版本分支。

Kubernetes 各个组件之间存在着版本偏差支持策略：

- 在高可用集群中，多个 API Server 的小版本号最多差 1，如最新的是 1.25，那么最旧的只能是 1.24。
- kubelet 的版本不能高于 API Server 的版本，最多比 API Server 低两个小版本。

虽然有这样的版本偏差兼容，但是生产环境一定要选择同一版本，避免出现未知 BUG。

2022 年 5 月 3 日， kubernetes 1.24 版本正式发布，正式移除了内嵌的 `dockershim`，若需要 1.24 及其以后的版本依然支持 docker，则需要引入第三方 cri-dockerd 项目进行支持。

在官方文档中有对大规模集群的特别说明：

> https://kubernetes.io/zh-cn/docs/setup/best-practices/cluster-large/

以 Kubernetes v1.25 版本为例：

- 每个节点的 Pod 数量不超过 110
- 节点数不超过 5000
- Pod 总数不超过 150000
- 容器总数不超过 300000





## 组件介绍

Kubernetes 核心组件示意图：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230221212142548-975977547.png)

Kubernetes 集群包含 `Master / Node（或 Worker）` 节点，所有操作都在 Master 上面执行。

<br>

**Master**

Master 节点是 Kubernetes 集群的控制节点，负责整个集群的管理和控制，其核心组件包含：

- `API Server`：集群网关，提供了 REST API 接口，所有组件之间的通讯都需要经过 API Server 转发，并且提供认证鉴权访问控制等。
- `Controller Manager`：用于实现 Kubernetes 集群故障检测和恢复的自动化工作。负责执行各种控制器：
  - Replication Controller：副本控制器。定期关联 Replication Controller (RC) 和 Pod，以保证集群中 RC 所关联的 Pod 副本数始终保持为与预设值一致。
  - Node Controller：节点控制器。Kubelet 在启动时会通过 API Server 注册自身的节点信息，并定时向 API Server 汇报状态信息。API Server 在接收到信息后将信息更新到 Etcd 中。Node Controller 通过 API Server 实时获取 Node 的相关信息，实现管理和监控集群中的各个 Node 节点的相关控制功能。
  - ResourceQuota Controller：资源配额管理控制器。用于确保指定的资源对象在任何时候都不会超量占用系统上物理资源。
  - Namespace Controller：命名空间控制器。用户通过 API Server 可以创建新的 Namespace 并保存在 Etcd 中，Namespace Controller 定时通过 API Server 读取这些 Namespace 信息来操作 Namespace。
  - Service Account Controller：服务账号控制器。主要在命名空间内管理 ServiceAccount，以保证名为 default 的 ServiceAccount 在每个命名空间中存在。
  - Token Controller：令牌控制器。作为 Controller Manager 的一部分，主要用作监听 serviceAccount 的创建和删除动作以及监听 secret 的添加、删除动作。
  - Service Controller：服务控制器。主要用作监听 Service 的变化。
  - Endpoint Controller：端点控制器。Endpoints 表示了一个 Service 对应的所有 Pod 副本的访问地址，而 Endpoints Controller 是负责生成和维护所有 Endpoints 对象的控制器。
- `Scheduler`：集群调度器，负责监视新创建的、未指定运行节点的 Pod，并选择合适的节点来运行 Pod。
- `Kubectl`：命令行执行工具，CLI 客户端，用于用户执行 Kubernetes 集群管理操作。

<br>

**Node / Worker**

Node 节点也叫 Worker 节点，是 Kubernetes 集群的工作节点，它的工作负载由 Master 调配，主要用于运行容器，其组件包含：

- `Kubelet`：负责 Pod 的创建，启动，监控，重启，销毁等操作，和 Master 节点协作管理集群。
- `Kube-proxy`：负责为 Service 提供 Cluster 内部的服务发现和负载均衡。

<br>

**其它组件**

除了 Kubernetes 本身的组件以外，还需要以下必不可少的组件：

- `ETCD`：分布式高性能键值存储数据库，用于存储集群信息，和 API Server 交互，所有的状态，操作都会记录在这里。
- `Container runtime`：容器运行时，1.24 版本移除 dockershim 后，目前主流的容器运行时为 containerd，CRI-O，RKT 等。
- `CoreDNS`：灵活的，可扩展的 DNS 服务器，可以为集群内的 Pod 提供 DNS 服务。
- `Calico / Flannel`：目前用的比较多的两个网络组件，用于实现容器跨主机之间的通信。

其它组件开源参考官方的网站：

> https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/





## 组件通信

通信示意图：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230221212842471-689507455.png)

从图中可以看出，集群的核心组件就是 API Server：

- API Server 负责 ETCD 的所有操作，也只有 API Server 才能直接操作 ETCD。
- 所有组件之间的请求都需要通过 API Server 帮忙转发，并且通过 API Server 将请求信息记录到 ETCD 中。





