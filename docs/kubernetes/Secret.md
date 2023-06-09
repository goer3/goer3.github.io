## Secret

ConfigMap 一般用于存储非安全的配置信息，原因在于 ConfigMap 使用的是明文的方式存储，通过 describe 就能直接看到。

这用来保存密码等对象显然是不合理的。此时就需要另外一个对象帮忙完成，那就是 `Secret`。

Secret 主要包含了以下几种类型：

* `Opaque`：base64 编码的 Secret，主要用来存储密码，密钥等。但数据可以通过 `base64 -d` 解码得到，加密性很弱。
* `kubernetes.io/dockercfg`：`~/dockercfg` 文件的序列化形式。
* `kubernetes.io/dockerconfigjson`：用来存储私有 docker registry 的认证信息，`~/.docker/config.json` 文件的序列号形式。
* `kubernetes.io/service-account-token`：ServiceAccount 在创建时，Kubernetes 会默认创建一个对应的 Secret 对象，Pod 如果使用 ServiceAccount，对应的 Secret 会自动的挂载到 Pod 的 `/run/secret/kubernetes.io/serviceaccount` 中。
* `kubernetes.io/ssh-auth`：用于 SSH 身份认证的凭据。
* `kubernetes.io/basic-auth`：用于基本身份认证的凭据。
* `bootstrap.kubernetes.io/token`：用于节点接入集群校验的 Secret。

上面是 Secret 对象内置的几种类型，通过 Secret 的 type 字段设置，也可以定义自己的 Secret 类型。如果 type 字段为空，则使用默认的 `Opaque` 类型。



### Opaque

Secret 资源对象包含 2 个键值对：

* `data`：用于存储 base64 编码的任意数。

* `stringData`：为了方便 secret 使用未编码的字符串。

<br>

使用示例：

```bash
# 通过 base64 加密两个数据
echo -n admin | base64
echo -n admin123 | base64
```

如图所示：

![image-20230427171941669](images/Secret/image-20230427171941669.png "bg-black")

使用 data 的方式创建 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo1
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW4xMjM=
```

使用 stringData 的方式创建 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo2
type: Opaque
stringData:
  username: admin
  password: admin123
```

查看创建结果：

![image-20230427173046071](images/Secret/image-20230427173046071.png "bg-black")

查看配置信息：

![image-20230427172542077](images/Secret/image-20230427172542077.png "bg-black")

可以看到两种方式创建的 Serect 的  value 部分都不会直接显示。

如果想要获取内容，可以将其输出为 yaml 就能看到，不过都是 base64 之后的值，然后通过 base64 解密。

![image-20230427172804936](images/Secret/image-20230427172804936.png "bg-black")

base64 解密方法：

```bash
echo -n YWRtaW4xMjM= | base64 -d
```

<br>

注意，如果 Secret 需要加密的 Key/Value 很多，那么可以直接将它看成一个整体，和 ConfigMap 定义文件一样。例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo3
type: Opaque
stringData:
  config.yaml: |
    username: admin
    password: admin123
```

这样所以配置会被当成一个整体加密。如图所示：

![image-20230427173721444](images/Secret/image-20230427173721444.png "bg-black")



### 使用 Secret（环境变量）

和 ConfigMap 一样，也可以在环境变量中直接使用 Secret 内容。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-demo
spec:
  containers:
  - name: busybox
    image: busybox
    env:
      - name: SRT_USERNAME
        valueFrom:
          secretKeyRef:
            name: secret-demo1
            key: username
      - name: SRT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: secret-demo1
            key: password
    args: ["/bin/sh", "-c", "env"]
```

创建后结果如图所示：

![image-20230427174301430](images/Secret/image-20230427174301430.png "bg-black")

可以看到 Pod 中拿到的值已经被解析成明文了。



### 使用 Secret（数据卷挂载）

同样的，Secret 也支持像 ConfigMap 一样挂载数据卷的方式将配置挂载到 Pod 中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-demo
spec:
  containers:
  - name: busybox
    image: busybox
    args: ["/bin/sh", "-c", "cd /data/config && cat config.yaml"]
    volumeMounts:
      - name: v-secret-demo
        mountPath: /data/config
  volumes:
    - name: v-secret-demo
      secret:
        secretName: secret-demo3
        items:
          - key: config.yaml
            path: config.yaml
```

查看如图所示：

![image-20230427175132031](images/Secret/image-20230427175132031.png "bg-black")



### kubernetes.io/dockerconfigjson

该类型的 Secret 主要用于存放 Docker registry 的认证信息，命令格式为：

```bash
kubectl create secret docker-registry NAME --docker-username=user --docker-password=password --docker-email=email [--docker-server=string] [--from-file=[key=]source]
```

多了一个关键字：`docker-registry`，标识这是用来创建特定类型的 Secret。使用示例：

```bash
# 命令行创建
kubectl create secret docker-registry secret-demo4 --docker-server="hub.docker.com" --docker-username="dylan" --docker-password="123456" --docker-email="1214966109@qq.com"

# 从文件创建
kubectl create secret docker-registry secret-demo5 --from-file=.dockerconfigjson=/root/.docker/config.json

# 或者另一种方式
kubectl create secret generic secret-demo6 --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
```

当从私有仓库中拉取镜像的时候就需要用到认证，可以在资源清单中配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  containers:
  - name: demo
    image: hub.docker.com/imageDemo:v1.0
  imagePullSecrets:
    - name: secret-demo4
```

特别注意：

> `ImagePullSecrets` 与 Secret 不同，因为 Secret 可以挂载到 Pod 中，但是 ImagePullSecrets 只能由 Kubelet 访问。

除了该方法设置 ImagePullSecrets 的方式访问私有仓库获取镜像以外，还可以通过 ServiceAccount 中设置 ImagePullSecrets 然后自动为使用了该 SA 的 Pod 注入配置信息。



### kubernetes.io/basic-auth

该类型用来存放用于基本身份认证所需的凭据信息，使用这种 Secret 类型时，Secret 的 data（或 stringData）字段中一般会包含以下两个键：

* username：用于身份认证的用户名。
* password：用于身份认证的密码或令牌。

可以和创建 Opaque 一样通过 data 字段或者 stringData 字段定义。 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo7
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: admin123
```

提供基本身份认证类型的 Secret 也仅仅是出于用户方便性和分类管理考虑，也可以使用 Opaque 类型来代替。



### kubernetes.io/ssh-auth

该类型用来存放 SSH 身份认证中所需要的凭据，使用这种 Secret 类型时，Secret 的 data（或 stringData）字段中一般会提供一个 `ssh-privatekey` 键值对，作为要使用的 SSH 凭据。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo8
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: |
          MIIEpQIBAAKCAQEAulqb...
```

提供 SSH 身份认证类型的 Secret 也仅仅是出于用户方便性和分类管理考虑，也可以使用 Opaque 类型来代替。



### kubernetes.io/tls

该类型用来存放证书及其相关密钥。常用于给 Ingress 资源校验 TLS 链接，使用此类型的 Secret 时，Secret 的 data （或 stringData）字段必须包含 `tls.key` 和 `tls.crt` 主键。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo9
type: kubernetes.io/tls
data:
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

提供 TLS 类型的 Secret 也仅仅是出于用户方便性和分类管理考虑，也可以使用 Opaque 类型来代替。

当使用 kubectl 来创建 TLS Secret 时，可以通过命令指定证书来直接创建更为方便：

```bash
kubectl create secret tls secret-demo9 --cert=/path/cert --key=/path/key
```

需要注意的是用于 `--cert` 的公钥证书必须是 `.PEM` 编码的 （Base64 编码的 DER 格式），且与 `--key` 所给定的私钥匹配，私钥必须是通常所说的 PEM 私钥格式，且未加密。

对这两个文件而言，PEM 格式数据的第一行和最后一行（`--------BEGIN CERTIFICATE-----` 和 `-------END CERTIFICATE----`）都不会包含在其中。



### kubernetes.io/service-account-token

`ServiceAccount` 是 Pod 和集群 API Server 通讯的访问凭证。

`kubernetes.io/service-account-token` 这种类型就是主要给 ServiceAccount 使用。ServiceAccount 在创建时会默认创建对应的 Secret。每一个名称空间下都会有一个默认的 ServiceAccount 和一个与之绑定的 Secret。

![image-20230428000852663](images/Secret/image-20230428000852663.png "bg-black")

当创建 Pod 的时候，如果没有指定 ServiceAccount，Pod 就会使用当前名称空间中名为 default 的 ServiceAccount。

<br>

运行一个 Pod 然后查看创建之后的资源清单：

```bash
kubectl run nginx --image=nginx
kubectl get pod nginx -o yaml
```

可以看到关于 ServiceAccount 的信息：

```yaml
...
spec:
  ...
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-82ffk
      ...
  serviceAccount: default
  serviceAccountName: default
  ...
  volumes:
  - name: kube-api-access-82ffk
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

可以看到这样几个关于 ServiceAccount 的信息：

* 因为没指定 Service Account，所以 `serviceAccountName` 字段使用了默认的 default。
* 默认自动定义了一个类型为 `projected` 的 Volume，该类型支持同时挂载多个来源的数据。这里有三个来源：
  * downwardAPI：获取 namespace。
  * configMap：获取 ca.crt 证书。
  * serviceAccountToken：也就是 ServiceAccount。

挂载后在容器 `/var/run/secrets/kubernetes.io/serviceaccount` 目录如图所示：

![image-20230427235641082](images/Secret/image-20230427235641082.png "bg-black")

<br>

当然，用户也可以通过在 ServiceAccount 或者 Pod 上配置不自动挂载 ServiceAccount API 凭据：

```yaml
# ServiceAccount 示例
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo
automountServiceAccountToken: false
...
---
# Pod 示例
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  serviceAccountName: demo
  automountServiceAccountToken: false
  ...
```



### ServiceAccount Token

ServiceAccount 是 Pod 和集群 API Server 通讯的访问凭证，传统方式下，在 Pod 中使用 ServiceAccount 可能存在以下风险：

- ServiceAccount 中的 JWT 没有绑定 audience 身份，因此所有 ServiceAccount 的使用者都可以彼此扮演，存在伪装攻击的可能。
- 传统方式下，每一个 ServiceAccount 都需要存储在一个对应的 Secret 中，并且会以文件形式存储在对应的应用节点上。而集群的系统组件在运行过程中也会使用到一些权限很高的 ServiceAccount，其增大了集群管控平面的攻击面，攻击者可以通过获取这些管控组件使用的 ServiceAccount 非法提权。
- ServiceAccount 中的 JWT token 没有设置过期时间，当上述 ServiceAccount 泄露情况发生时，只能通过轮转 ServiceAccount 的签发私钥来进行防范。
- 每一个 ServiceAccount 都需要创建一个与之对应的 Secret，在大规模的应用部署下存在弹性和容量风险。

为解决这个问题，Kubernetes 提供了 `ServiceAccount Token` 特性用于增强 ServiceAccount 的安全性。

该方案可使 Pod 支持以卷投影的形式将 ServiceAccount 挂载到容器中，从而避免了对 Secret 的依赖。

同时 ServiceAccount Token 还受时间限制（`expirationSeconds`），受 audience 约束，并且不与 Secret 对象关联。如果删除 Pod 或删除 ServiceAccount，则这些 Token 将无效，从而可以防止任何误用。

Kubelet 还会在 Token 即将到期时自动更新 Token。

此功能在 Kubernetes 1.12 中引入，v1.20 稳定，想要开启，需要修改 API Server 设置以下命令行参数，Kubeadm 集群默认开启。

```bash
# serviceaccount token 中的签发身份，即 token payload 中的 iss 字段
--service-account-issuer
# token 私钥文件路径
--service-account-key-file
# token 签名私钥文件路径
--service-account-signing-key-file
# 可选参数，合法的请求 token 身份，用于 API Server 服务端认证请求 token 是否合法
--api-audiences
```

如果是按照我这里的二进制安装方法，也已经开启该功能。

已经开启配置集群就可以指定 Token 所需属性，例如身份和有效时间。

使用示例：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-demo

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    # 挂在自定义的目录
    - mountPath: /var/run/secrets/tokens
      name: v-token
  serviceAccountName: sa-demo
  volumes:
  - name: v-token
    projected:
      sources:
      # 设置过期时间和身份
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200
          audience: vault
```

Kubelet 组件会替 Pod 请求 Token 并将其保存起来以供 Pod 内可用，并在 Token 快要到期的时候刷新它。 

一般刷新时间在 TTL 的 80% 的时候，或生命期超过 24 小时的时候主动轮换它。然后应用程序负责在 Token 被轮换时重新加载其内容。





## 不可变配置

如果某个 Pod 已经在通过环境变量使用某 Secret，对该 Secret 的更新不会被容器马上看见，除非容器被重启，当然也可以使用一些第三方的解决方案在 Secret 发生变化时触发容器重启。

在 Kubernetes v1.21 版本提供了不可变的 Secret 和 ConfigMap 的可选配置 `stable`，对于大量使用 Secret 或 ConfigMap 的集群时，禁止变更具有以下好处：

- 可以防止意外更新导致应用程序中断。
- 通过标记为不可变来关闭 API Server 对其的 watch 操作，从而显著降低 API Server 的负载，提升集群性能。

配置方法：

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
# 标记为不可变
immutable: true
```

一旦 Secret 或 ConfigMap 被标记为不可更改，撤销此操作或更改 data 字段的内容都是不允许的，只能删除并重新创建这个 Secret。

现有的 Pod 将继续使用已删除 Secret 的挂载点，所以 Pod 也需要重启。





## Secret vs ConfigMap

相同点：

* k/v 形式
* 属于特定的命名空间
* 可以导出到环境变量
* 可以通过目录/文件形式挂载
* 通过 volume 挂载的配置信息均可热更新

<br>

不同点：

* Secret 可以被 ServerAccount 关联。
* Secret 可以存储 docker register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像。
* Secret 支持 Base64 加密。
* Secret 有多种分类，而 ConfigMap 不区分类型。

<br>

> 同样 Secret 文件大小限制为 1MB（ETCD 的要求），Secret 虽然采用 Base64 编码，但还是可以很方便解码获取到原始信息，所以对于非常重要的数据还是需要慎重考虑，可以考虑使用 Vault 来进行加密管理。



