## 安装准备

区别于使用 kubeadm 部署集群时所有核心组件都托管在集群上。二进制安装则采用守护进程的方式直接将各个组件运行在宿主机，生产环境更为推荐。

服务还是那些，只需要将服务器准备到初始化状态即可，即：

> https://www.cnblogs.com/Dy1an/p/17142958.html

对于生产环境，不推荐在 Master 节点再安装 kubelet、kube-proxy 以及容器运行时 Containerd，使得它同时也是 Worker 节点运行。但是测试环境由于机器有限，为了使得集群看起来节点不那么少，可以也安装上用于测试。

具体规划如下：

| 主机      | IP            | 系统       | 配置      | 初始化安装服务                                               |
| --------- | ------------- | ---------- | --------- | ------------------------------------------------------------ |
| ops       | 192.168.2.40  | CentOS 7.9 | 4C/4G/20G | cfssl，kubernetes                                            |
| slb-01    | 192.168.2.11  | CentOS 7.9 | 4C/2G/20G | nginx，keepalived                                            |
| slb-02    | 192.168.2.12  | CentOS 7.9 | 4C/2G/20G | nginx，keepalived                                            |
| master-01 | 192.168.2.21  | CentOS 7.9 | 4C/3G/20G | containerd，nerdctl，etcd，apiserver，controller-manager，scheduler，kubectl，kubelet，kube-proxy，calico |
| master-02 | 192.168.2.22  | CentOS 7.9 | 4C/3G/20G | containerd，nerdctl，etcd，apiserver，controller-manager，scheduler，kubectl，kubelet，kube-proxy，calico |
| worker-01 | 192.168.2.31  | CentOS 7.9 | 4C/4G/20G | containerd，nerdctl，kubelet，kube-proxy，calico             |
| worker-02 | 192.168.2.32  | CentOS 7.9 | 4C/4G/20G | containerd，nerdctl，kubelet，kube-proxy，calico             |
| worker-03 | 192.168.2.33  | CentOS 7.9 | 4C/4G/20G | containerd，nerdctl，kubelet，kube-proxy，calico             |
| VIP       | 192.168.2.100 | /          | /         | keepalived 提供                                              |

CoreDNS 和 Dashboard 属于托管在集群内部，不确定节点。



## 安装证书签发环境 cfssl

Kubernetes 集群各组件之间的通信都会涉及证书验证，在使用 Kubeadm 安装集群的时候就遇到因为证书有效期问题，各种替换处理。所以在二进制安装签发证书的时候，尽可能证书做到一劳永逸。同时，证书的签发也是整个 Kubernetes 集群二进制安装中最难的部分。

为此，准备了 ops 这台服务器专门用于做源码编译，证书签发这类独立于集群的工作。

下载 cfssl 相关文件，具体不同版本可以去 Github 下载，本文编写的时候最新版本为 `1.6.3` ：

> https://github.com/cloudflare/cfssl/releases

<br>

执行服务器：`ops`

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_linux_amd64 -O cfssl-json
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl-certinfo_1.6.3_linux_amd64 -O cfssl-certinfo
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_linux_amd64 -O cfssl
mv cfssl* /usr/bin/
chmod 755 /usr/bin/cfssl*
```

文件说明：

- cfssl：证书签发工具。
- cfssl-json：将 json 格式的证书信息转换成文件格式。
- cfssl-certinfo：验证证书信息。





## 创建免密认证

由于 OPS 服务器需要经常传输文件到其它服务器，所以其它服务器要对 OPS 配置免密登录。

执行服务器：`ops`

```bash
# 创建公钥和私钥
ssh-keygen

# 上传公钥到其它服务器，需要一台一台的验证
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.2.21
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.2.22
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.2.31
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.2.32
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.2.33

# 测试免密登录
ssh root@192.168.2.21

# 退出登录
exit
```





## 签发 CA 证书

CA 根证书是签发其他证书的基础，在使用 kubeadm 安装集群查看证书的有效期的时候是能看到有三个 CA 证书：

- ca
- etcd-ca
- front-proxy-ca

它们的有效期都是 10 年。为了做到二进制安装也更贴近官方最佳实践，本文的规划也是如此。

同时，对于证书的签发，有两个非常重要的注意事项：

1. 证书有效期：二进制安装由于证书签发是有用户自己决定的，所以推荐尽可能签长一点，一劳永逸。
2. IP 白名单：某些证书是需要绑定 IP 地址的，为了方便后期的节点扩容，可以在 `hosts` 字段多配置一些 IP 用作备用。

<br>

不管是 Master 节点还是 Worker 节点都是需要保存证书的，在 kubeadm 安装的集群中，证书都被保存在了 /etc/kubernetes/pki 下面。

二进制安装环境为了对证书有更好的管理，使用自定义的目录用于存储所需的证书。

执行服务器：`所有 Master、Worker、ops 节点`

```bash
# 创建证书存储目录，服务目录，安装包目录
mkdir -p /opt/certs/kubernetes
mkdir -p /opt/package
mkdir -p /opt/service
cd /opt/certs/kubernetes
```

<br>

**Kubernetes CA**

创建生成 Kubernetes 集群 CA 证书请求的 Json 文件。

执行服务器：`ops`

```bash
cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen",
            "O": "kubernetes",
            "OU": "ops"
        }
    ],
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

生成证书：

```bash
cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
```

<br>

**ETCD CA**

创建生成 ETCD 集群 CA 证书请求的 Json 文件。

执行服务器：`ops`

```bash
cat > etcd-ca-csr.json << EOF
{
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen",
            "O": "etcd",
            "OU": "ops"
        }
    ],
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

生成证书：

```bash
cfssl gencert -initca etcd-ca-csr.json | cfssl-json -bare etcd-ca
```

<br>

**front-proxy CA**

创建生成 front-proxy 的 CA 证书请求的 Json 文件。

执行服务器：`ops`

```bash
cat > front-proxy-ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

生成证书：

```bash
cfssl gencert -initca front-proxy-ca-csr.json | cfssl-json -bare front-proxy-ca
```

<br>

在请求 Json 中有几个字段需要特别关注一下：

- `CN`：Common Name，所有 csr 文件都必须有字段，对于不同证书，一般拥有不同的含义。
  - 普通 SSL 证书，一般为网站域名。
  - 代码签名证书，一般为申请单位名称。
  - 客户端证书，一般为证书申请者的姓名。
  - 在 Kubernetes 集群中，apiserver 会从证书中提取该字段作为请求的用户名，所以在定义的时候需要注意。
- `O`：Organization，apiserver 会从证书中提取该字段作为请求用户所属的组。
- `expiry`：证书有效期，`876000h` 表示 100 年。

<br>

创建证书通用配置，该配置可以简化后面生成证书请求的 Json，算是公共配置。

执行服务器：`ops`

```bash
cat > ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "876000h"
            }
        }
    }
}
EOF
```

<br>

CA 证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222201626364-1747728140.png "bg-black")





## 签发 ETCD 证书

ETCD 集群作为 Kubernetes 集群运行的基础，独立于 Kubernetes 集群之外。可以先给它签发证书并进行安装部署。

值得注意的是，ETCD 证书签发是需要绑定 ETCD 集群节点部署机器的 IP 地址的，为了方便后期的扩容迁移，可以配置一些备份 IP。

创建生成证书的请求 Json 文件。

执行服务器：`ops`

```bash
cat > etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "192.168.2.21",
        "192.168.2.22",
        "192.168.2.23",
        "192.168.2.24",
        "192.168.2.25"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "etcd",
        "OU": "ops"
    }]
}
EOF
```

生成证书：

```bash
cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssl-json -bare etcd
```

这里就会用到证书生成的通用配置 `ca-config.json` 文件并获取了它的 `kubernetes` 字段定义的信息。

<br>

ETCD 证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222203211755-1336221317.png "bg-black")

<br>

签发完成后查询证书信息：

```bash
cfssl-certinfo -cert etcd.pem
```

其中 `not_after` 字段定义了证书的有效期，可以看到是 100 年以后。

当然，也可以使用 openssl 直接看证书有效期：

```bash
openssl x509 -in etcd.pem -noout -dates
```

<br>

将 ETCD 相关证书分发到两台 Master 节点：

```bash
scp etcd* root@192.168.2.21:/opt/certs/kubernetes/
scp etcd* root@192.168.2.22:/opt/certs/kubernetes/
```





## 部署 ETCD 集群

从 Kubernetes v1.26 官方 CHANGELOG 中可以看到它已经支持 ETCD v3.5.5 版本了：

> https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.26.md

在所有 Master 服务器上面进行 ETCD 的安装。

执行服务器：`所有 Master`

```bash
cd /opt/package

# 下载解压安装
wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz
tar -zxf etcd-v3.5.5-linux-amd64.tar.gz
mv etcd-v3.5.5-linux-amd64 /opt/service/etcd

# 创建服务所需目录并整理文件
cd /opt/service/etcd/
mkdir data conf logs docs bin
mv *md Documentation docs/
mv etcd* bin/

# 添加环境变量
cat >> /etc/profile << EOF
# ETCD 环境变量
export ETCD_HOME="/opt/service/etcd"
export PATH=\$PATH:\$ETCD_HOME/bin
EOF

# 生效
source /etc/profile

# 查看版本
etcd --version
```

<br>

配置主配置文件。

执行服务器：`所有 Master`

```bash
# 本机 IP
IP=$(ip a | grep "192.168.2" | awk '{print $2}' | cut -d "/" -f 1)

cat > /opt/service/etcd/conf/etcd.yml << EOF
name: etcd-$(hostname)
data-dir: /opt/service/etcd/data
listen-client-urls: https://${IP}:2379,https://127.0.0.1:2379
advertise-client-urls: https://${IP}:2379,https://127.0.0.1:2379
listen-peer-urls: https://${IP}:2380
initial-advertise-peer-urls: https://${IP}:2380
initial-cluster: etcd-master-01=https://192.168.2.21:2380,etcd-master-02=https://192.168.2.22:2380
initial-cluster-token: KubernetesToken
initial-cluster-state: new
client-transport-security:
  cert-file: /opt/certs/kubernetes/etcd.pem
  key-file: /opt/certs/kubernetes/etcd-key.pem
  trusted-ca-file: /opt/certs/kubernetes/etcd-ca.pem
  client-cert-auth: true
peer-transport-security:
  cert-file: /opt/certs/kubernetes/etcd.pem
  key-file: /opt/certs/kubernetes/etcd-key.pem
  trusted-ca-file: /opt/certs/kubernetes/etcd-ca.pem
  client-cert-auth: true
EOF
```

有几个值得注意的地方：

- `name`：节点在集群中的名称，和 `initial-cluster` 字段中的名称要对应，而且在集群中要唯一。
- `initial-cluster`：ETCD 集群节点属于静态发现，所以所有节点都要写上去。
- IP：注意不同节点监听的 IP 是不同的，根据自己需求改为自己的 IP。
- 证书：需要配置 ETCD 自己的证书和这个证书对应的 CA 证书。

<br>

配置启动文件。

执行服务器：`所有 Master`

```bash
cat > /etc/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/service/etcd
ExecStart=/opt/service/etcd/bin/etcd --config-file=/opt/service/etcd/conf/etcd.yml
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动服务：

```bash
systemctl daemon-reload
systemctl enable --now etcd
systemctl status etcd
```

注意，如果只启动一个 etcd 节点服务是无法启动的，会一直卡住，至少启动两个节点。

<br>

配置命令别名，用于简化用户从客户端操作 ETCD。

执行服务器：`所有 Master`

```bash
cat >> /etc/profile << EOF
# ETCD 变量
export ETCDCTL_API=3
alias etcdctl='etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/opt/certs/kubernetes/etcd-ca.pem --cert=/opt/certs/kubernetes/etcd.pem --key=/opt/certs/kubernetes/etcd-key.pem'
EOF

source /etc/profile
```

<br>

查看集群节点状态：

```bash
etcdctl endpoint status --cluster -w table | grep -v "127.0.0.1"
```

如图所示，可以查看到集群中谁是 Leader：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222205638028-685431785.png "bg-black")





## 安装 Kubernetes

Kubernetes 本身是 Go 语言开发，所以只需要直接下载安装包解压配置后就能直接使用。

可以去 Github CHANGELOG 中找到对应版本提供的二进制下载地址：

> https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.26.md#server-binaries-2

由于 Kubernetes 二进制包中包含了所有 Master 和 Worker 的核心组件，所以在所有 Master 和 Worker 节点都安装。

<br>

下载安装 Kubernetes。

执行服务器：`所有 Master、Worker、ops 节点`

```bash
# 下载安装包
cd /opt/package/
wget https://dl.k8s.io/v1.26.0/kubernetes-server-linux-amd64.tar.gz

# 解压安装
tar -zxf kubernetes-server-linux-amd64.tar.gz
mv kubernetes /opt/service/

# 删除无用文件
cd /opt/service/kubernetes/
rm -rf kubernetes-src.tar.gz LICENSES/
cd server/bin/
rm -f *_tag *tar

# 创建相关目录
mkdir -p /opt/service/kubernetes/server/{logs,conf}
mkdir -p /opt/service/kubernetes/manifests

# 添加环境变量
cat >> /etc/profile << EOF
# Kubernetes
export KUBERNETES_HOME=/opt/service/kubernetes
export PATH=\$KUBERNETES_HOME/server/bin:\$PATH
EOF

# 配置生效
source /etc/profile

# 查看安装结果
kubectl version
```





## 签发 apiserver 证书

apiserver 是集群各个组件交互的核心组件，也是需要绑定对应节点 IP 地址的。

创建生成证书的请求 Json 文件。

执行服务器：`ops`

```bash
cd /opt/certs/kubernetes

# 创建文件
cat > kube-apiserver-csr.json << EOF
{
    "CN": "kube-apiserver",
    "hosts": [
        "127.0.0.1",
        "192.168.2.100",
        "192.168.2.21",
        "192.168.2.22",
        "192.168.2.23",
        "192.168.2.24",
        "192.168.2.25",
        "10.10.0.1",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "kubernetes",
        "OU": "ops"
    }]
}
EOF
```

apiserver 证书的 hosts 需要包含：

- Master 节点和备用 IP。
- Service 网段的第一个 IP。
- Kubernetes 自带的一些解析地址。

<br>

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssl-json -bare kube-apiserver
```

<br>

kube-apiserver 证书生成结果，如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222211253951-352713575.png "bg-black")





## 签发 apiserver 聚合证书

该证书用于控制第三方组件使用集群的时候的权限管理。

创建生成证书的请求 Json 文件。

执行服务器：`ops`

```bash
cat > front-proxy-client-csr.json << EOF
{
    "CN": "front-proxy-client",
    "key": {
        "algo": "rsa",
        "size": 2048
    }
}
EOF
```

生成证书，注意聚合证书的 CA 是之前单独生成的：

```bash
cfssl gencert -ca=front-proxy-ca.pem -ca-key=front-proxy-ca-key.pem -config=ca-config.json -profile=kubernetes front-proxy-client-csr.json | cfssl-json -bare front-proxy-client
```

由于没配置 hosts 字段，在生成证书的时候会提示，忽略即可：

> 2023/02/22 21:15:13 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
> websites. For more information see the Baseline Requirements for the Issuance and Management
> of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
> specifically, section 10.2.3 ("Information Requirements").

<br>

证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222211608467-223867044.png "bg-black")





## 签发 controller-manager 证书

创建生成证书的请求 Json 文件。

执行服务器：`ops`

```bash
cat > kube-controller-manager-csr.json << EOF
{
    "CN": "system:kube-controller-manager",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "system:kube-controller-manager",
        "OU": "ops"
    }]
}
EOF
```

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssl-json -bare kube-controller-manager
```

<br>

该证书涉及到高可用集群的角色等配置，需要生成额外的配置，这就是为何 ops 节点也需要安装 Kubernetes 的原因：

```bash
# set-cluster：设置一个集群项，高可用集群使用 VIP 代理的 API Server 地址
kubectl config set-cluster kubernetes --certificate-authority=/opt/certs/kubernetes/ca.pem --embed-certs=true --server=https://192.168.2.100:6443 --kubeconfig=/opt/certs/kubernetes/kube-controller-manager.kubeconfig

# set-credentials 设置一个用户项
kubectl config set-credentials system:kube-controller-manager --client-certificate=/opt/certs/kubernetes/kube-controller-manager.pem --client-key=/opt/certs/kubernetes/kube-controller-manager-key.pem --embed-certs=true --kubeconfig=/opt/certs/kubernetes/kube-controller-manager.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-context system:kube-controller-manager@kubernetes --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=/opt/certs/kubernetes/kube-controller-manager.kubeconfig

# 使用某个环境当做默认环境
kubectl config use-context system:kube-controller-manager@kubernetes --kubeconfig=/opt/certs/kubernetes/kube-controller-manager.kubeconfig
```

<br>

证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222212505270-1890675430.png "bg-black")





## 签发 scheduler 证书

创建生成证书的请求 Json 文件。

执行服务器：`ops`

```bash
cat > kube-scheduler-csr.json << EOF
{
    "CN": "system:kube-scheduler",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "system:kube-scheduler",
        "OU": "ops"
    }]
}
EOF
```

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssl-json -bare kube-scheduler
```

<br>

该证书涉及到高可用集群的角色等配置，需要生成额外的配置：

```bash
# set-cluster：设置一个集群项，高可用集群使用 VIP 代理的 API Server 地址
kubectl config set-cluster kubernetes --certificate-authority=/opt/certs/kubernetes/ca.pem --embed-certs=true --server=https://192.168.2.100:6443 --kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig

# set-credentials 设置一个用户项
kubectl config set-credentials system:kube-scheduler --client-certificate=/opt/certs/kubernetes/kube-scheduler.pem --client-key=/opt/certs/kubernetes/kube-scheduler-key.pem --embed-certs=true  --kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-context system:kube-scheduler@kubernetes --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig

# 使用某个环境当做默认环境
kubectl config use-context system:kube-scheduler@kubernetes --kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig
```

<br>

证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222212814389-377993294.png "bg-black")





## 签发 admin 证书

该证书用于生成管理员权限的 kubeconfig，这里主要是给 kubectl 使用。

执行服务器：`ops`

```bash
cat > admin-csr.json << EOF
{
    "CN": "admin",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "system:masters",
        "OU": "ops"
    }]
}
EOF
```

apiserver 使用 RBAC 对客户端授权时内部定义了一些 RoleBindings，如 system:masters 绑定 cluster-admin，该角色拥有 apiserver 的所有权限。通过 O 指定了 Group，由于都是被同一个 CA 签名，所以访问 apiserver 是认证通过的，然后根据所属组的角色权限绑定就能获得了 apiserver 的所有权限。

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssl-json -bare admin
```

<br>

该证书涉及到高可用集群的角色等配置，需要生成额外的配置：

```bash
# set-cluster：设置一个集群项，高可用集群使用 VIP 代理的 API Server 地址
kubectl config set-cluster kubernetes --certificate-authority=/opt/certs/kubernetes/ca.pem --embed-certs=true --server=https://192.168.2.100:6443 --kubeconfig=/opt/certs/kubernetes/admin.kubeconfig

# set-credentials 设置一个用户项
kubectl config set-credentials kubernetes-admin --client-certificate=/opt/certs/kubernetes/admin.pem --client-key=/opt/certs/kubernetes/admin-key.pem --embed-certs=true --kubeconfig=/opt/certs/kubernetes/admin.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --user=kubernetes-admin --kubeconfig=/opt/certs/kubernetes/admin.kubeconfig

# 使用某个环境当做默认环境
kubectl config use-context kubernetes-admin@kubernetes --kubeconfig=/opt/certs/kubernetes/admin.kubeconfig
```

<br>

证书生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222213123881-1256768039.png "bg-black")





## 创建 SA Key 和分发证书

生成公钥私钥。

执行服务器：`ops`

```bash
openssl genrsa -out /opt/certs/kubernetes/sa.key 2048
openssl rsa -in /opt/certs/kubernetes/sa.key -pubout -out /opt/certs/kubernetes/sa.pub
```

生成结果如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222213244300-303515530.png "bg-black")

<br>

将 ops 目录下生成的所有证书分发到所有 Master 节点：

```bash
scp * root@192.168.2.21:/opt/certs/kubernetes/
scp * root@192.168.2.22:/opt/certs/kubernetes/
```





## 配置 apiserver

配置 apiserver 启动文件。

执行服务器：`所有 Master 节点`

```bash
# 本机 IP
IP=$(ip a | grep "192.168.2" | awk '{print $2}' | cut -d "/" -f 1)

cat > /opt/service/kubernetes/server/conf/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/service/kubernetes/server/bin/kube-apiserver \\
  --v=2 \\
  --allow-privileged=true \\
  --bind-address=0.0.0.0 \\
  --secure-port=6443 \\
  --advertise-address=${IP} \\
  --service-cluster-ip-range=10.10.0.0/16 \\
  --service-node-port-range=30000-50000 \\
  --etcd-servers=https://192.168.2.21:2379,https://192.168.2.22:2379 \\
  --etcd-cafile=/opt/certs/kubernetes/etcd-ca.pem \\
  --etcd-certfile=/opt/certs/kubernetes/etcd.pem \\
  --etcd-keyfile=/opt/certs/kubernetes/etcd-key.pem \\
  --client-ca-file=/opt/certs/kubernetes/ca.pem \\
  --tls-cert-file=/opt/certs/kubernetes/kube-apiserver.pem \\
  --tls-private-key-file=/opt/certs/kubernetes/kube-apiserver-key.pem \\
  --kubelet-client-certificate=/opt/certs/kubernetes/kube-apiserver.pem \\
  --kubelet-client-key=/opt/certs/kubernetes/kube-apiserver-key.pem \\
  --service-account-key-file=/opt/certs/kubernetes/sa.pub \\
  --service-account-signing-key-file=/opt/certs/kubernetes/sa.key \\
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota \\
  --authorization-mode=Node,RBAC \\
  --enable-bootstrap-token-auth=true \\
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \\
  --enable-aggregator-routing=true \\
  --requestheader-client-ca-file=/opt/certs/kubernetes/front-proxy-ca.pem \\
  --proxy-client-cert-file=/opt/certs/kubernetes/front-proxy-client.pem \\
  --proxy-client-key-file=/opt/certs/kubernetes/front-proxy-client-key.pem \\
  --requestheader-allowed-names=aggregator \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-username-headers=X-Remote-User
Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

<br>

启动服务：

```bash
# 创建启动文件软连接
ln -s /opt/service/kubernetes/server/conf/kube-apiserver.service /etc/systemd/system/kube-apiserver.service

# 启动服务
systemctl daemon-reload
systemctl enable --now kube-apiserver
systemctl status kube-apiserver
```





## 配置 controller-manager

配置 controller-manager 启动文件。

执行服务器：`所有 Master 节点`

```bash
cat > /opt/service/kubernetes/server/conf/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/service/kubernetes/server/bin/kube-controller-manager \\
  --cluster-signing-duration=876000h0m0s \\
  --feature-gates=RotateKubeletServerCertificate=true \\
  --bind-address=0.0.0.0 \\
  --root-ca-file=/opt/certs/kubernetes/ca.pem \\
  --cluster-signing-cert-file=/opt/certs/kubernetes/ca.pem \\
  --cluster-signing-key-file=/opt/certs/kubernetes/ca-key.pem \\
  --service-account-private-key-file=/opt/certs/kubernetes/sa.key \\
  --kubeconfig=/opt/certs/kubernetes/kube-controller-manager.kubeconfig \\
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \\
  --leader-elect=true \\
  --use-service-account-credentials=true \\
  --node-monitor-grace-period=40s \\
  --node-monitor-period=5s \\
  --pod-eviction-timeout=2m0s \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=172.16.0.0/16 \\
  --requestheader-client-ca-file=/opt/certs/kubernetes/front-proxy-ca.pem \\
  --node-cidr-mask-size=24
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

<br>

启动服务：

```bash
# 创建启动文件软连接
ln -s /opt/service/kubernetes/server/conf/kube-controller-manager.service /etc/systemd/system/kube-controller-manager.service

# 启动服务
systemctl daemon-reload
systemctl enable --now kube-controller-manager
systemctl status kube-controller-manager
```





## 配置 scheduler

配置 scheduler 启动文件。

执行服务器：`所有 Master 节点`

```bash
cat > /opt/service/kubernetes/server/conf/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/service/kubernetes/server/bin/kube-scheduler \\
  --leader-elect=true \\
  --authentication-kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig \\
  --authorization-kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig \\
  --kubeconfig=/opt/certs/kubernetes/kube-scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

<br>

启动服务：

```bash
# 创建启动文件软连接
ln -s /opt/service/kubernetes/server/conf/kube-scheduler.service /etc/systemd/system/kube-scheduler.service

# 启动服务
systemctl daemon-reload
systemctl enable --now kube-scheduler
systemctl status kube-scheduler
```





## 配置 kubectl

执行 kubectl 默认证书配置。

执行服务器：`所有 Master 节点`

```bash
cd /opt/certs/kubernetes
mkdir ~/.kube
cp admin.kubeconfig ~/.kube/config
```

<br>

配置 kubectl 命令补全，方便后续直接 tab 补全命令：

```bash
yum install -y bash-completion

# 加载配置
source /usr/share/bash-completion/bash_completion

# 临时生效
source <(kubectl completion bash)

# 永久生效
echo "source <(kubectl completion bash)" >> /etc/profile
```

<br>

查看 Master 集群状态：

```bash
kubectl get cs
```

如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222215436856-1053053483.png "bg-black")





## 配置 TLS Bootstrapping

在一个 Kubernetes 集群中，Worker 节点上的组件（kubelet 和 kube-proxy）需要与 Kubernetes 控制平面组件通信，尤其是 kube-apiserver。 为了通信的安全性， 需要使用到节点上的客户端 TLS 证书。

但是客户端很多，又很难有通用的 TSL 证书直接使用，如果每一次加节点都需要重新生成证书，那维护将变得非常麻烦。

为了简化这一过程，从 1.4 版本开始，Kubernetes 引入了一个证书请求和签名 API。

> https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/

<br>

采用 TLS bootstrapping 生成证书的大致简化流程如下：

- 管理员通过 apiserver 生成一个 bootstrap token 并将它写入到 kubeconfig 文件中。
- Kubelet 通过 --bootstrap-kubeconfig 启动参数指定 kubeconfig 文件，然后调用 apiserver 的 API 接口，生成自己所需的服务器和客户端证书。
- 证书生成后，Kubelet 采用生成的证书和 apiserver 进行通信，并删除本地的 kubeconfig 文件，避免 bootstrap token 泄漏。

想要启动该功能，只需要在 apiserver 中启动参数中添加 `--enable-bootstrap-token-auth`，并创建一个 Kubelet 访问的 bootstrap token secret 即可。

> https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/bootstrap-tokens/

<br>

添加配置生成的资源清单。

执行服务器：`master-01`

```bash
cat > /opt/certs/kubernetes/bootstrap.secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-c8ad9c
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token generated by 'kubelet '."
  token-id: c8ad9c
  token-secret: 2e4d610cf3e7426e
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups:  system:bootstrappers:default-node-token,system:bootstrappers:worker,system:bootstrappers:ingress
 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-bootstrapper
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-certificate-rotation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF
```

<br>

创建资源：

```bash
kubectl create -f bootstrap.secret.yaml
```

<br>

生成 kubelet-bootstrap.kubeconfig，可以在 master-01 节点生成，也可以在 ops 节点：

```bash
kubectl config set-cluster kubernetes --certificate-authority=/opt/certs/kubernetes/ca.pem --embed-certs=true --server=https://192.168.2.100:6443 --kubeconfig=/opt/certs/kubernetes/kubelet-bootstrap.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user --token=c8ad9c.2e4d610cf3e7426e --kubeconfig=/opt/certs/kubernetes/kubelet-bootstrap.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes --cluster=kubernetes --user=tls-bootstrap-token-user --kubeconfig=/opt/certs/kubernetes/kubelet-bootstrap.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes --kubeconfig=/opt/certs/kubernetes/kubelet-bootstrap.kubeconfig
```

<br>

将生成的配置分发到 `master-02` 节点和 `ops` 节点，因为 ops 节点之后需要分发它到其它 Worker 节点：

```bash
# 由于 master-01 节点没有做其它节点的免密，需要一台一台的执行发送
scp *bootstrap* root@192.168.2.22:/opt/certs/kubernetes/
scp *bootstrap* root@192.168.2.40:/opt/certs/kubernetes/
```





## 配置 kubelet

在 ops 节点生成相关配置文件，然后分发到其它节点去。

执行服务器：`ops`

```bash
cd /opt/certs/kubernetes

# 生成 kubelet 配置
cat > kubelet.yaml << EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/certs/kubernetes/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
# CoreDNS 使用 Service 网段的 IP，后面会单独配置
clusterDNS:
- 10.10.10.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /opt/service/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF
```

<br>

配置启动文件：

```bash
cat > kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/opt/certs/kubernetes/kubelet-bootstrap.kubeconfig --kubeconfig=/opt/certs/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--cert-dir=/opt/certs/kubernetes --feature-gates=RotateKubeletServerCertificate=true --rotate-certificates"
Environment="KUBELET_RINTIME=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
Environment="KUBELET_CONFIG_ARGS=--config=/opt/service/kubernetes/server/conf/kubelet.yaml"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node=''"
ExecStart=/opt/service/kubernetes/server/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_SYSTEM_ARGS \$KUBELET_EXTRA_ARGS \$KUBELET_RINTIME
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

kubeconfig 会自动在配置的地方生成，所以直写上不用管。

<br>

将所有配置分发到所有 Master 节点，这里因为测试环境节点太少，需要 Master 节点同时作为 Worker 节点使用，所以也要部署 Kubelet：

```bash
scp * root@192.168.2.21:/opt/certs/kubernetes/
scp * root@192.168.2.22:/opt/certs/kubernetes/
```

<br>

将需要的配置分发到 Worker 节点：

```bash
scp kubelet* ca.pem front-proxy-ca.pem root@192.168.2.31:/opt/certs/kubernetes/
scp kubelet* ca.pem front-proxy-ca.pem root@192.168.2.32:/opt/certs/kubernetes/
scp kubelet* ca.pem front-proxy-ca.pem root@192.168.2.33:/opt/certs/kubernetes/
```

<br>

调整所有节点的配置文件结构：

```bash
cd /opt/certs/kubernetes/
mv kubelet.service /opt/service/kubernetes/server/conf/
mv kubelet.yaml /opt/service/kubernetes/server/conf/
ln -s /opt/service/kubernetes/server/conf/kubelet.service /etc/systemd/system/kubelet.service
```

<br>

所有节点启动服务：

```bash
systemctl daemon-reload
systemctl enable --now kubelet
systemctl status kubelet
```

任意 Master 节点查看节点添加情况：

```bash
kubectl get nodes
```

如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222221859573-145126108.png "bg-black")

此时节点都是 NotReady 状态，原因在于缺少 CNI 网络插件。

通过查看证书可以发现，Kubelet 已经是通过接口自动签发证书了：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230222222146402-1714818776.png "bg-black")

<br>

由于本文配置了自动允许加入集群，如果没配置还需要手动确认节点加入集群：

```bash
# 查看申请
kubectl get csr

# 同意申请
kubectl certificate approve 申请名称
```





## 配置 kube-proxy

任意 Master 创建 Service Account 和角色绑定。

执行服务器：`master-01`

```bash
kubectl -n kube-system create serviceaccount kube-proxy
kubectl create clusterrolebinding system:kube-proxy --clusterrole system:node-proxier --serviceaccount kube-system:kube-proxy
```

<br>

生成 kube-proxy.kubeconfig：

```bash
# 获取 Token
SECRET=$(kubectl -n kube-system get sa/kube-proxy --output=jsonpath='{.secrets[0].name}')
JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET --output=jsonpath='{.data.token}' | base64 -d)

# 生成 kubeconfig
kubectl config set-cluster kubernetes --certificate-authority=/opt/certs/kubernetes/ca.pem --embed-certs=true --server=https://192.168.2.100:6443 --kubeconfig=/opt/certs/kubernetes/kube-proxy.kubeconfig

kubectl config set-credentials kubernetes --token=${JWT_TOKEN} --kubeconfig=/opt/certs/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kubernetes --cluster=kubernetes --user=kubernetes --kubeconfig=/opt/certs/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kubernetes --kubeconfig=/opt/certs/kubernetes/kube-proxy.kubeconfig
```

将生成的配置分发到 `ops` 节点，因为 ops 节点之后需要分发它到其它 Worker 节点：

```bash
scp kube-proxy* root@192.168.2.40:/opt/certs/kubernetes/
```

<br>

创建主配置文件。

执行服务器：`ops`

```bash
cd /opt/certs/kubernetes

# 创建文件
cat > kube-proxy.yaml << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /opt/certs/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12 
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
EOF
```

<br>

创建启动文件：

```bash
cat > kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/service/kubernetes/server/bin/kube-proxy \\
  --config=/opt/service/kubernetes/server/conf/kube-proxy.yaml
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

<br>

发送配置文件到所有其它节点：

```bash
scp kube-proxy* root@192.168.2.21:/opt/certs/kubernetes/
scp kube-proxy* root@192.168.2.22:/opt/certs/kubernetes/
scp kube-proxy* root@192.168.2.31:/opt/certs/kubernetes/
scp kube-proxy* root@192.168.2.32:/opt/certs/kubernetes/
scp kube-proxy* root@192.168.2.33:/opt/certs/kubernetes/
```

<br>

调整所有节点的配置文件结构。

执行服务器：`所有 Master 和 Worker 节点`

```bash
cd /opt/certs/kubernetes/
mv kube-proxy.service /opt/service/kubernetes/server/conf/
mv kube-proxy.yaml /opt/service/kubernetes/server/conf/
ln -s /opt/service/kubernetes/server/conf/kube-proxy.service /etc/systemd/system/kube-proxy.service
```

<br>

启动服务：

```bash
systemctl daemon-reload
systemctl enable --now kube-proxy
systemctl status kube-proxy
```





## 安装 Calico

执行服务器：`master-01`

```bash
cd /opt/service/kubernetes/addons
wget https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml
kubectl apply -f calico.yaml
```

拉取镜像可能会非常的慢，半个小时都有可能。如果一直没有成功，可以手动拉取镜像看看：

```bash
nerdctl -n k8s.io image pull docker.io/calico/node:v3.25.0
nerdctl -n k8s.io image pull docker.io/calico/cni:v3.25.0
nerdctl -n k8s.io image pull docker.io/calico/kube-controllers:v3.25.0
```

也可以查看 Pod 输出的信息，看是什么问题：

```bash
kubectl -n kube-system describe pod calico-node-xxx
```

安装完成以后，所有节点都会变成 Ready 状态：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223100021243-1316879401.png "bg-black")





## 安装 CoreDNS

执行服务器：`master-01`

```bash
cd /opt/service/kubernetes/addons
git clone https://github.com/coredns/deployment.git
mv deployment coredns
cd coredns/kubernetes
./deploy.sh -s -i 10.10.10.10 | kubectl apply -f -
```

注意 Service IP 是之前设置的 `10.10.10.10`，结果如图：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223100237729-1779648553.png "bg-black")





## 验证集群解析功能

创建一个用于测试的资源清单。

执行服务器：`master-01`

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```

<br>

测试域名解析功能：

```bash
kubectl exec busybox -n default -- nslookup kubernetes
```

如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223100339401-1232284636.png "bg-black")





## 安装 Dashboard

在 master-01 节点下载资源清单进行安装。

```bash
cd /opt/service/kubernetes/addons/

# 由于 raw.githubusercontent.com 访问不到，所以加了个 github proxy 的代理
wget https://ghproxy.com/https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
mv recommended.yaml dashboard.yaml
```

修改配置文件的 Service 配置：

```yaml
...
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  # 加上 type=NodePort 变成 NodePort 类型的服务
  type: NodePort
```

应用配置：

```bash
kubectl apply -f dashboard.yaml
```

Dashboard 集成了一个 metrics-scraper 的组件，可以通过 Kubernetes 的 Metrics API 收集一些基础资源的监控信息，并在 web 页面上展示，所以要想在页面上展示监控信息就需要提供 Metrics API，比如安装 Metrics Server。

<br>

查看安装结果：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223183439502-122848868.png "bg-black")

可以看到 Dashboard 映射到了 34936 端口，通过任意节点 IP 访问，需要使用 HTTPS：

> https://192.168.2.21:34936/#/login

如图所示：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223184854407-643635810.png "border")

<br>

创建一个全局访问用户：

```bash
cd /opt/service/kubernetes/addons

# 资源清单
cat > dashboard-admin.yaml << EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
EOF

# 应用
kubectl apply -f dashboard-admin.yaml

# 获取 token
kubectl get secret -n kubernetes-dashboard | grep admin-token

# 生成 base64 字符串
kubectl get secret <获取到的admin-token> -o jsonpath={.data.token} -n kubernetes-dashboard | base64 -d
```

然后用上面的 base64 解码后的字符串作为 token 登录 Dashboard 即可，注意要复制全：

![](https://img2023.cnblogs.com/blog/979767/202302/979767-20230223184819010-1451037204.png "border")

到此，二进制基础集群环境搭建完成！

