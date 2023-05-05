## OpenEBS

OpenEBS 是一种模拟了 AWS 的 EBS、阿里云的云盘等块存储实现的基于容器的存储开源软件。是一种基于 CAS（Container Attached Storage）理念的容器解决方案。其核心理念是存储和应用一样采用微服务架构，并通过 Kubernetes 来做资源编排。其架构实现上，每个卷的 Controller 都是一个单独的 Pod，且与应用 Pod 在同一个节点，卷的数据使用多个 Pod 进行管理。

![image-20230505180057293](images/OpenEBS/image-20230505180057293.png)

OpenEBS 的组件可以分为以下几类：

- 控制平面组件：管理 OpenEBS 卷容器，通常会用到容器编排软件的功能。
- 数据平面组件：为应用程序提供数据存储，包含 Jiva 和 cStor 两个存储后端。
- 节点磁盘管理器：发现、监控和管理连接到 Kubernetes 节点的媒体。
- 整合其它云原生工具：与 Prometheus、Grafana、Fluentd 和 Jaeger 进行整合。



## 控制平面

OpenEBS 集群的控制平面（Control Plane）被称为 `Maya`，负责供应和操作数据卷，如快照、克隆、创建执行存储策略等。

控制平面 Maya 实现了创建超融合的 OpenEBS，并将其挂载到 Kubernetes 调度引擎上，用来扩展系统的存储功能。

控制平面也是基于微服务的，通过不同的组件实现存储管理功能、监控、容器编排插件等功能。如下图所示：

![image-20230505181727613](images/OpenEBS/image-20230505181727613.png)





### PV Provisioner

该组件作为一个 Pod 运行，主要用于做供应决策。它是一个动态供应器，属于标准的 Kubernetes 外部存储插件。

![image-20230505190612264](images/OpenEBS/image-20230505190612264.png)

开发者用所需的卷参数构建一个请求，选择合适的存储类，然后 OpenEBS PV 动态供应器通过 `Maya-apiserver` 与集群的 API Server 进行交互，最终通过 Kubelet，在适当的节点上为卷控制器 Pod 和卷复制 Pod 创建部署规范。同时，用户可以使用 PVC 规范中的注解来控制容量 Pod 的调度。

目前，OpenEBS PV Provisioner 只支持一种类型的绑定，即 `iSCSI`。





### Maya Apiserver

Maya-apiserver 也叫 `m-apiserver`，作为一个 Pod 运行，主要用于暴露了存储 REST API。

![image-20230505191826673](images/OpenEBS/image-20230505191826673.png)

除此之外，m-apiserver 还负责以下几个工作：

* 创建创建卷 Pod 所需的部署规范文件。
  * 在生成这些规范文件后，调用 API Server 来相应地调度 Pod。
* 在 OpenEBS PV Provisioner 的卷供应结束时，创建 Kubernetes PV 对象，并挂载到应用 Pod 上。
  * 该 PV 对象由控制器 Pod 托管，控制器 Pod 又由一组位于不同节点的副本 Pod 支持，控制器 Pod 和副本 Pod 也是数据平面的一部分。
* 卷策略管理。
  * OpenEBS 提供了规范用于表达策略，m-apiserver 会解释 YAML 规范，将其转换为可执行组件，通过卷管理 Sidecar 来执行。





### Exporter  Sidecar

Exporter  是每个存储控制器 Pod（cStor/Jiva）的 Sidecar。这些 Sidecar 将控制平面与数据平面连接起来，以获取统计数据，比如：

* volume 读/写延迟
* 读/写 IOPS
* 读/写块大小
* 容量统计

![image-20230505194645727](images/OpenEBS/image-20230505194645727.png)





### Volume Mgmt Sidecar

Volume Mgmt  Sidecar 用于将控制器配置参数和卷策略传递给数据平面的卷控制器 Pod，以及将副本配置参数和数据保护参数传递给卷副本 Pod。

![image-20230505194522317](images/OpenEBS/image-20230505194522317.png)





## 数据平面

OpenEBS 持久化存储卷通过 Kubernetes 的 PV 来创建，使用 iSCSI 来实现，数据保存在节点上或者云存储中。数据卷完全独立于用户的应用的生命周期来管理，和 Kuberentes 中 PV 的思路一致。

目前，OpenEBS 提供了两个可以轻松插入的存储引擎： `Jiva` 和 `cStor`。这两个存储引擎都完全运行在 Linux 用户空间中，并且基于微服务架构。



















































