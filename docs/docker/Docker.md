## 安装准备

准备一台测试机器用于安装 docker，配置如下：

| IP            | 系统                          | CPU  | 内存 | 硬盘 |
| ------------- | ----------------------------- | ---- | ---- | ---- |
| 192.168.2.100 | CentOS Linux release 7.9.2009 | 4    | 4    | 30   |





## 系统初始化

### 防火墙

关闭防火墙，方便使用，生产云服务器一般使用云平台安全组提供防火墙功能：

```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
```



### Selinux

关闭 Selinux，避免出现位置 BUG：

```bash
# 关闭 Selinux
sed -i "s#^SELINUX=.*#SELINUX=disabled#g" /etc/selinux/config
setenforce 0
```



### Swap

关闭 Swap，提升性能：

```bash
# 关闭 swap 分区
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```



### Yum（云不需要）

本地配置国内 yum 源，加速依赖下载，云服务器已经自带了，不需要配置：

```bash
# 备份旧的 yum 源
cd /etc/yum.repos.d/
mkdir backup-$(date +%F)
mv *repo backup-$(date +%F)

# 添加阿里云 yum 源
curl http://mirrors.aliyun.com/repo/Centos-7.repo -o ali.repo

# 安装 epel 源
yum -y install epel-release
yum clean all
yum makecache
```



### 依赖安装

安装常用的依赖包和工具，更新系统补丁：

```bash
# 安装常用依赖
yum -y install gcc glibc gcc-c++ make cmake net-tools screen vim lrzsz tree dos2unix lsof \
    tcpdump bash-completion wget ntp setuptool openssl openssl-devel bind-utils traceroute \
    bash-completion bash-completion-extras glib2 glib2-devel unzip bzip2 bzip2-devel libevent libevent-devel \
    ntp expect pcre pcre-devel zlib zlib-devel jq psmisc tcping yum-utils device-mapper-persistent-data \
    lvm2 git device-mapper-persistent-data bridge-utils container-selinux binutils-devel \
    ncurses ncurses-devel elfutils-libelf-devel

# 升级服务器
yum update
```



### 时间同步（云不需要）

使用 `chrony` 替代以前的 `ntpdate`，云服务器有自己的同步机制：

```bash
# 安装服务
yum -y install chrony

# 设置时区
timedatectl set-timezone Asia/Shanghai

# 注释默认同步源
sed -i "s/^server.*$/# &/g" /etc/chrony.conf

# 配置阿里云时间同步源
cat >> /etc/chrony.conf << EOF
server ntp1.aliyun.com
server time1.aliyun.com
EOF

# 启动服务
systemctl start chronyd
systemctl enable chronyd
systemctl status chronyd

# 同步时间
chronyc makestep

# 查看系统时间
timedatectl
```



### 打开文件数优化

优化打开文件数，提升并发性能：

```bash
cat >> /etc/security/limits.conf << EOF
# 打开文件优化配置
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```



### 内核优化

主要包括网络优化和安全优化两个方面：

```bash
cat > /etc/sysctl.d/user.conf << EOF
# 内核调优
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
```





### 内核升级

docker 对于 CentOS 的要求为内核版本不低于 `3.10`，尽管 7.9 已经是 3.10 版本，但还是有升级的必要。在官方文档中有提到相关需求：

> https://docs.docker.com/engine/install/binaries/

同时作为生产环境，一般不会选择安装最新版本。但官方仓库中提供的 rpm 一般只会有一个 lt 和一个 ml 版本，所以需要从第三方下载想要的版本，统一服务器的内核版本，可以避免出现未知 BUG。

内核安装包下载地址：

> http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/

下载安装：

```bash
# 下载内核 rpm 包
mkdir /usr/local/src/kernel
cd /usr/local/src/kernel
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-6.2.10-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-6.2.10-1.el7.elrepo.x86_64.rpm

# 安装内核
yum -y localinstall kernel-ml-*

# 查看顺序
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

# 设置默认启动内核，上面的命令可以看到最新内核的序号是 0
grub2-set-default 0

# 重启系统
reboot
```

重启完成后查看：

```bash
uname -a
```





## 服务安装

docker 目前有三种版本可供选择：`nightly（开发版）`，`test（测试版）`，`stable（稳定版）`。



### 安装包

对于生产环境，为了避免因为版本不同导致集群出现未知 BUG，建议手动下载 rpm 包安装。本次安装版本为 `23.0.4`,下载地址：

> https://download.docker.com/linux/centos/7/x86_64/stable/Packages/

如果觉得下载慢，可以使用国内的镜像地址：

> http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/

所需安装包列表如下：

* [docker-ce](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-23.0.4-1.el7.x86_64.rpm)
* [docker-ce-cli](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-23.0.4-1.el7.x86_64.rpm)
* [containerd.io](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/containerd.io-1.6.20-3.1.el7.x86_64.rpm)
* [docker-ce-rootless-extras](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-rootless-extras-23.0.4-1.el7.x86_64.rpm)
* [docker-buildx-plugin](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-buildx-plugin-0.10.4-1.el7.x86_64.rpm)
* [docker-scan-plugin](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-scan-plugin-0.23.0-1.el7.x86_64.rpm)
* [docker-compose-plugin](http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-compose-plugin-2.17.2-1.el7.x86_64.rpm)



### 安装

```bash
# 卸载可能存在的旧版本 docker
yum remove docker \
	docker-client \
	docker-client-latest \
	docker-common \
	docker-latest \
	docker-latest-logrotate \
	docker-logrotate \
	docker-engine

# 创建安装包存放目录
mkdir /usr/local/src/docker
cd /usr/local/src/docker

# 下载安装包
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-23.0.4-1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-23.0.4-1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/containerd.io-1.6.20-3.1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-rootless-extras-23.0.4-1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-buildx-plugin-0.10.4-1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-scan-plugin-0.23.0-1.el7.x86_64.rpm
wget http://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-compose-plugin-2.17.2-1.el7.x86_64.rpm

# 安装
yum -y localinstall docker* containerd*

# 启动 docker
systemctl enable docker --now
systemctl status docker

# 查看 docker 信息
docker version
```





## 配置优化

docker 在初始化安装完成之后需要进行简单的优化配置，其中常见的包含以下几个方面：

**registry**

docker 默认的 registry 是 docker hub，访问有时会很慢，可以将它改成阿里云的 registry（需要注册用户）。

镜像加速获取地址：

> https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

最终的加速地址格式：``https://<你的ID>.mirror.aliyuncs.com``

除此之外还有其它加速地址：

- 网易云：`https://hub-mirror.c.163.com`
- 百度云：`https://mirror.baidubce.com`

<br>

**数据目录**

docker 的默认数据都存在 `/var/lib/docker` 下面，但是这是系统目录，如果磁盘很小，可能会因为数据量增多导致系统盘写满。

所有建议将 docker 的数据目录迁移到挂载的分区上。本文只是模拟，假设 `/devops` 目录是挂载的外部分区：

```bash
# 创建相关目录
mkdir -p /devops/docker/{run,lib}
```

<br>

**网络调整**

docker 在启动之后默认会在宿主机创建 `docker0` 网桥，分配的网段是 `172.17.0.1/16`。如果该网段和服务器的网段发生冲突，就需要对其进行修改。

如果不清楚网络划分，可以使用工具查看：

> http://tools.jb51.net/aideddesign/ip_net_calc/

<br>

**最终配置**

基于上面几个方面的调优，可以将优化后的配置更新到 `/etc/docker/daemon.json` 文件中。

```bash
cat > /etc/docker/daemon.json << EOF
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"],
	"bip": "172.16.0.1/16",
	"exec-root": "/devops/docker/run",
	"data-root": "/devops/docker/lib"
}
EOF
```

参数说明：

* `exec-opts`：调整 docker 为 systemd 管理。
* `registry-mirrors`：设置注册点，可以设置多个，这里的阿里云加速地址，需要换成自己的。
* `bip`：修改 docker 的网段。
* `exec-root`：重新定义 docker 的运行目录。
* `data-root`：重新定义 docker 的数据目录。

完成之后需要重启 docker：

```bash
# 重启 docker
systemctl restart docker

# 查看配置是否生效
docker info

# 查看网络是否生效
ip a
```

可以看到数据目录，registry，docker0 网桥的 IP 都已经得到了更新。





## 故障排查

下面是 docker 安装配置优化时常发生的一些错误和解决方式。

`错误 1`：docker 在停止的时候可能会出现提醒：

> Warning: Stopping docker.service, but it can still be activated by: docker.socket

告警的意思为：如果你试图连接到 docker socket，而 docker 服务没有运行，系统将自动启动 docker。

该错误可以直接忽略，如果有强迫症，则可以修改启动文件 `/lib/systemd/system/docker.service`：

```bash
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

删除掉上面启动参数中的 `-H fd://`，重启即可：

```bash
sed -i "s#-H fd://##g" /lib/systemd/system/docker.service
systemctl daemon-reload 
systemctl stop docker
systemctl start docker
```

<br>

`错误 2`：重启失败，大概可能是 /etc/docker/daemon.json 存在报错。

通过查看启动日志查找原因：

```bash
tail -1000f /var/log/messages
```

<br>

`错误情况 1`：

> unable to configure the Docker daemon with file /etc/docker/daemon.json: invalid character 'Â' looking for beginning of value

查看配置文件没有问题，但是一直这个错误，可能是该 json 文件 tab 或者空格存在问题，通过执行命令：

```bash
cat -A /etc/docker/daemon.json
```

可以发现文件中可能存在很多特殊的字符，这种问题常发生在网上复制配置的情况。通过将里面空格和 tab 替换成空格之后故障解决。

<br>

`错误情况 2`：

> unable to configure the Docker daemon with file /etc/docker/daemon.json: invalid character '}' looking for beginning of object key string

json 表面看起来没有问题，但是可能会因为 json 最后一个值后面也加了 `,` 符号的缘故。





## daemon.json

配置文件 /etc/docker/daemon.json 常见的配置项：

```bash
{
    # 在引擎 API 中设置 CORS 标头
    "api-cors-header": "",
    # 要加载的授权插件
    "authorization-plugins": [],
    # 将容器附加到网桥，不一定使用 docker0，也可以是自建的网桥
    "bridge": "",
    # 指定网桥 IP
    "bip": "172.16.0.0/16",
    # 为所有容器设置父 cgroup
    "cgroup-parent": "",
    # 分布式存储后端的 URL
    "cluster-store": "",
    # 设置集群存储选项（默认map []）
    "cluster-store-opts": {},
    # 要通告的地址或接口名称
    "cluster-advertise": "",
    # 启用调试模式，启用后，可以看到很多的启动信息。默认 false
    "debug": true,
    # 容器默认网关 IPv4 地址
    "default-gateway": "", 
    # 容器默认网关 IPv6 地址
    "default-gateway-v6": "",
    # 容器的默认 OCI 运行时，默认为 runc
    "default-runtime": "runc",
    # 容器的默认 ulimit（默认[]）
    "default-ulimits": {},
    # 设定容器 DNS 的地址，在容器的 /etc/resolv.conf 文件中可查看
    "dns": ["192.168.1.1"],
    # 容器 /etc/resolv.conf 文件，其他设置
    "dns-opts": [],
    # 设定容器的搜索域。
    "dns-search": [],
    # 运行时执行选项
    "exec-opts": [],
    # 执行状态文件的根目录，默认 /var/run/docker
    "exec-root": "",
    # 固定 IP 的 IPv4 子网
    "fixed-cidr": "",
    # 固定 IP 的 IPv6 子网
    "fixed-cidr-v6": "",
    # docker 运行时使用的根路径，默认 /var/lib/docker
    "data-root": "/var/lib/docker",
    # UNIX 接字的组，默认 docker
    "group": "",
    # 设置容器 hosts
    "hosts": [],
    # 启用容器间通信，默认 true
    "icc": false,
    # 绑定容器端口时的默认 IP，默认 0.0.0.0
    "ip": "0.0.0.0",
    # 启用 iptables 规则添加，默认 true
    "iptables": false,
    # 启用IPv6网络
    "ipv6": false,
    # 默认 true, 启用 net.ipv4.ip_forward
    "ip-forward": false,
    # 启用 IP 伪装，默认 true
    "ip-masq": false,
    # 设置私有仓库地址可以设为 http
    "insecure-registries": ["xxxx"],
    # docker 主机的标签
    "labels": ["nodeName=xxx"],
    # 在容器仍在运行时启用 docker 的实时还原
    "live-restore": true,
    # 容器日志的默认驱动程序，默认 json-file
    "log-driver": "",
    # 设置日志记录级别："调试"，"信息"，"警告"，"错误"，"致命"
    "log-level": "",
    # 设置每个请求的最大并发下载量，默认 3
    "max-concurrent-downloads": 3,
    # 设置每次推送的最大同时上传数，默认 5
    "max-concurrent-uploads": 5,
    # 设置容器网络 MTU
    "mtu": 0, 
    # 设置守护程序的 oom_score_adj，默认 -500
    "oom-score-adjust": -500,
    # Docker 守护进程的 PID 文件
    "pidfile": "",
    # 全时间戳机制
    "raw-logs": false,
    # 设置镜像加速
    "registry-mirrors": ["https://xxxx"],
    # 启用 selinux 支持，默认 false
    "selinux-enabled": false,
    # 要使用的存储驱动程序
    "storage-driver": "",
    # 设置默认地址或群集广告地址的接口
    "swarm-default-advertise-addr": "",
    # 启动 TLS 认证开关，默认 false
    "tls": true,
    # 通过 CA 认证过的 certificate 文件路径，默认 ~/.docker/ca.pem
    "tlscacert": "",
    # TLS 的 certificate 文件路径，默认 ~/.docker/cert.pem
    "tlscert": "",
    # TLS 的 key 文件路径，默认 ~/.docker/key.pem
    "tlskey": "",
    # 使用 TLS 并做后台进程与客户端通讯的验证，默认 false
    "tlsverify": true,
    # 使用 userland 代理进行环回流量，默认 true
    "userland-proxy": false,
    # 用户名称空间的用户/组设置
    "userns-remap": "",
    # 存储驱动程序选项
    "storage-opts": [
        "overlay2.override_kernel_check=true",
        "overlay2.size=15G"
    ],
    # 容器默认日志驱动程序选项
    "log-opts": {
        "max-file": "3",
        "max-size": "10m",
    },
}
```



