## 容器（Container）

对于容器这个词，大部分人第一时间想到的肯定是生活中常见瓶瓶罐罐，用来装水的东西。它给人的第一感觉就是能 “装”。

而在 IT 领域，`Container` 就被直译为容器，但 Container 本身是集装箱的意思，容器属于中国人的信雅达叫法。

基于集装箱的本质，大致可以得出容器的一些特征：

> 规格标准化，层层堆叠，互相隔离，将各类零散的货物分门别类，形成统一的形状，提升运输效率，降低管理成本，保护了货物的完整性。

在早期，IT 领域就是通过借鉴这一思想，研发出了 `hypervisor` 虚拟化，将不同操作系统的虚拟机通过 hypervisor（KVM、XEN 等）来衍生、运行、销毁。

但随着时间推移，用户也发现了 hypervisor 存在的问题：

> 每次部署发布都得搞一个完整操作系统和附带的依赖环境，而用户其实更关注自己部署的应用，这导致了任务变重和性能低下。

而容器的目的就在于实现底层操作系统和环境的复用，达到类似以下效果：

> 将一辆兰博基尼（应用），打包放到一个集装箱里（容器），通过货轮轻而易举的将它从上海码头（CentOS 环境）运送到纽约码头（Ubuntu 环境）。在运输期间，兰博基尼（应用）没有受到任何的损坏，在纽约码头卸货后依然可以完美风骚的飙车（启动正常）。

![image-20230420012826463](images/AboutContainer/image-20230420012826463.png)





## 容器的原理

Linux Container 容器技术，简称 `LXC`，是一种 `轻量级的操作系统层虚拟化技术`，它的诞生（2008 年）解决了 IT 世界里的 “集装箱运输” 问题。

LXC 可以提供轻量级的虚拟化，以便隔离进程和资源，而且不需要提供指令解释机制以及全虚拟化的其他复杂性。

与传统虚拟化技术相比，它的优势在于：

* 与宿主机使用同一个内核，性能损耗小。
* 不需要指令级模拟。
* 不需要即时（Just-in-time）编译。
* 容器可以在 CPU 核心的本地运行指令，不需要任何专门的解释机制。
* 避免了准虚拟化和系统调用替换中的复杂性。
* 轻量级隔离，在隔离的同时还提供共享机制，以实现容器与宿主机的资源共享。

<br>

为了实现容器进程对外界的隔离，容器底层主要运用了 `名称空间（Namespaces）`、`控制组（Cgroups）` 和 `切根（Chroot）`。

![image-20230420013257744](images/AboutContainer/image-20230420013257744.png)

<br>

**名称空间（Namespaces）**

每个运行的容器都有自己的名称空间（诞生于 2002 年），它主要用于实现资源的隔离。

Linux 操作系统默认提供了以下 6 个常用的 Namespace 的 API：

1. `PID Namespace`：提供进程隔离能力
   * 在 Linux 中，PID 为 1 的进程（init/systemd）是其他所有进程的父进程，容器内也要有一个父进程来管理其子进程。
   * 不同容器就是通过 PID Namespace 隔离的，不同的名称空间中可以有相同的 PID。
2. `MNT Namespace`：提供磁盘挂载点和文件系统的隔离能力
   * 每个容器都要有独立的根文件系统用户空间，以实现在容器里面启动服务并且使用容器的运行环境。
   * 在容器里面不能访问宿主机的资源，宿主机使用了 chroot 技术把容器锁定到一个指的运行目录里面。
3. `IPC Namespace`：提供进程间通信的隔离能力
   * 允许一个容器内的不同进程的（内存，缓存等）数据访问，但是不能跨容器访问其他容器的数据。
   * 容器中进程交互还是采用 Linux 常见的进程交互方法（interprocess communication，IPC）。
4. `Net Namespace`：提供网络隔离能力
   * 每一个容器都类似于虚拟机一样有自己的网络设备，IP地址，路由表，/proc/net 目录等。
   * 以 docker 为例，使用 network namespace 启动一个 vethX 接口，这样容器将拥有它自己的桥接 IP 地址，通常是 docker0。
5. `UTS Namespace`：提供主机名隔离能力
   * 允许容器拥有独立的 hostname 和 domain name，使其在网络上能被视作一个独立节点而非主机的一个进程。
6. `User Namespace`：提供用户隔离能力
   * 每个容器可以有不同的用户和组 id，也就是说可以在容器内用容器内部的用户执行程序而非主机上的用户。

<br>

**控制组（Control groups）**

简称 `Cgroups`，是 Linux 内核提供的一种可以限制、记录、隔离进程组的物理资源的机制，由 Google 贡献，2007 年合并到 Linux Kernel。

因为 Namespace 只能改变进程的视觉范围，不能真实地对资源做出限制，所以就需要用到 Cgroup 技术。以防止某个容器把宿主机资源全部用完导致其它容器也宕掉。

在 Linux 的 `/sys/fs/cgroup` 目录中有 cpu、memory、devices、net_cls 等目录，可以按需修改相应的配置文件来设置某个进程 ID 对物理资源的最大使用率。

<br>

**切根（Change to root）**

简称 `chroot`，用于改变一个程序运行时参考的根目录位置，让不同容器在不同的虚拟根目录下工作，从而相互不直接影响。

<br>

特别说明：

> Docker 并不是 LXC 替代品，Docker 只是在 LXC 的基础之上提供了一系列更强大的功能。





## 容器的特点

想要更好的理解容器的特点，就需要拿跟它跟硬件抽象层虚拟化 hypervisor 技术做对比。

![image-20230420015914877](images/AboutContainer/image-20230420015914877.png)

主要的区别如下：

1. 本质上来看，虚拟机是通过 Hypervisor 虚拟化硬件，然后在上安装不同的操作系统，而容器是宿主机上运行的不同进程。
2. 用户体验上来看，虚拟机是重量级的，系统本身占用的物理资源多，启动时间长，而容器则相反。
3. 隔离性上来看，虚拟机隔离的更彻底，容器则要差一些。

<br>

比较得出容器主要包含以下几个特点：

1. 极其轻量：只打包了必要的 bin/lib。
2. 秒级部署：根据镜像的不同，容器的部署大概在毫秒与秒之间。
3. 易于移植：`一次构建，到处运行`。





## 容器发展史

虽然现在提到容器就想到 Docker，但容器却是从 1979 年的 Chroot 开始的。而 Docker 是在 2013 年才开始推出第一个版本，是它带火了容器而已。

具体发展历程如下：

* 1979 年，`Chroot`：一套 Unix 操作系统，为每个进程提供一个隔离化的磁盘空间。
* 2000 年，FreeBSD Jails：与 chroot 类似，增加了进程的沙箱，对制作资源进行隔离。
* 2001 年，Linux Vserver：每个分区被称为一套安全上下文，其中虚拟化系统被称为一套虚拟私有服务器。
* 2004 年，Solaris 容器：将系统资源控制与分区提供的边界结合，各分区在单一的操作系统实例之内。
* 2005 年，OpenVZ：安装有补丁的 Linux 内核实现虚拟化，隔离能力，资源管理以及检查点交付。
* 2006 年，Process 容器：对整套进程集合中的资源使用量进行限制，分配与隔离。
* 2007 年，`Control Groups`：谷歌实现的 Cgroups，后被合并到 Linux 内核中。
* 2008 年，`LXC`：通过 liblxc 库交付，提供可与 Python，Lua，Go 等语言对接的 API。
* 2011 年，Warden：不与 Linux 紧密耦合，以后台进程方式运行，并提供 API 以实现容器管理。
* 2013 年，`Docker`：目前最流行的容器引擎，具备完整的生态系统。
* 2014 年，`Rocket`：由 CoreOS 开发，专门用于解决 Docker 中存在的缺陷。
* 2016 年，Windows 容器：docker 能够在 Windows 平台上运行。





## 容器标准化

> docker 是容器，但容器并不只有 docker。

任何技术出现都需要一个标准来规范它，不然各搞各的很容易导致技术实现的碎片化，出现大量的冲突和冗余。

所以，在 2015 年，由 Google，Docker、CoreOS、IBM、微软、红帽等厂商联合成立了 `OCP（开放容器项目）` 组织，并于 2016 年推出了第一个开放容器标准 `OCI（Open Container Initiative）`。

其主要包括：`runtime 运行时标准` 和 `image 镜像标准`。

<br>

**容器运行时标准**

1. creating：使用 create 命令创建容器，这个过程称为创建中。
2. created：容器创建出来，但是还没有运行，表示镜像和配置没有错误，容器能够运行在当前平台。
3. running：容器的运行状态，里面的进程处于 up 状态，正在执行用户设定的任务。
4. stopped：容器运行完成，或运行出错，或 stop 命令之后，容器处于暂停状态。该状态下，容器还有很多信息保存在平台中，并没有完全被删除。

<br>

**容器镜像标准**

1. 文件系统：以 layer 保存的文件系统，每个 layer 保存了和上层之间变化的部分。
2. config 文件：保存了文件系统的层级信息，以及容器运行时需要的一些信息，指定了镜像在某个特定平台和系统的配置。
3. manifest 文件：镜像的 config 文件索引，有哪些 layer，额外的 annotation 信息，还保存了很多和当前平台有关的信息。
4. index 文件：可选的文件，指向不同平台的 manifest 文件，该文件能保证一个镜像可跨平台使用，每个平台拥有不同的 manifest 文件。







