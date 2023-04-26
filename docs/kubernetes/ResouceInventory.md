## 资源清单

集群搭建成功后就可以在集群里运行应用。但是在之前，需要了解几个概念：

容器运行的基础就是镜像，在镜像准备好后，如何在镜像以容器的方式在集群中运行呢？

在使用 Docker 和 Containerd 的时候，可以直接通过命令 `docker run` 来运行应用，在 Kubernetes 环境下也同样也可以用类似 `kubectl run` 这样的命令来运行应用。

但是同样的，在是一个 Docker 的时候也发现了痛点，单纯的 run 命令在某些时候会很复杂，而且不便于保存下次继续运行，于是就有了 docker-compose 资源清单。

Kubernetes 也一样，不推荐使用命令行的方式运行，而是推荐使用 `资源清单` 来描述应用，它可以用 YAML 或者 JSON 文件来编写，一般来说 YAML 文件更方便阅读和理解，所以都会选择 YAML 文件来进行描述，虽然系统最终也会将它转换成 JSON。

资源清单文件来定义好一个应用后，就可以通过 kubectl 工具来直接运行它：

```bash
# 通过资源清单创建
kubectl create -f xxxx.yaml
```

也可以使用：

```bash
# 应用资源清单
kubectl apply -f xxxx.yaml
```

两者的区别在于：

* `create`：创建新资源。如果资源存在，则会抛出错误。
* `apply`：将配置应用于资源。 如果资源不存在，则创建。它可以运行多次，而且支持运行多个文件。

<br>

整个工作流程大致如下：

* Kubectl 将清单提交给了 API Server。
* API Server 获取清单描述的应用信息，存入到 ETCD 数据库中。
* Scheduler 发现该 Pod 还没绑定到节点上，就会把它调度到一个最合适的节点上，然后将信息传回 API Server，保存到 ETCD。
* 节点上的 Kubelet 组件 watch 到有一个 Pod 被分配过来，就去获取 Pod 的信息，然后根据描述通过容器运行时把容器创建出来。
* 创建完成后，把 Pod 状态再通过 API Server 写回到 ETCD 中去。





## 第一个资源清单

示例资源清单：

```yaml
# API 版本
apiVersion: v1
# API 对象
kind: Pod
# 元数据
metadata:
  name: nginx-demo
  namespace: default
  labels:
    name: demo
# 期望
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
      - containerPort: 80
```

所有的资源清单都应该包含 5 个配置：

* `apiVersion`：对应资源对象目前 Kubernetes 集群支持的版本，不同版本的 Kubernetes 相同资源对象提供的 API 版本不同。
* `kind`：资源对象名称。
* `metadata`：元数据，主要用于配置运行对象的名称，名称空间，标签等。
* `spec`：用户期望，包含需要运行哪些容器，怎样的方式运行等定义。
* `status`：不需要用户配置，这是应用资源清单后自动生成的目前系统运行状况。

具体资源清单支持哪些配置可以通过 kubectl 命令查看：

```bash
# 查看 Pod 支持的
kubectl explain pod

# 可以继续一级一级的查看配置
kubectl explain pod.metadata
```





## YAML

`YAML` 是专门用来写配置文件的语言，非常简洁和强大，远比 JSON 格式方便。

它的基本语法规则如下：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进时不允许使用 `Tab` 键，只允许使用空格
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
- `#` 表示注释，从这个字符一直到行尾，都会被解析器忽略
- `---` 内容分隔符，意味着上面的内容和下面的内容都是单独存在的，可以看成两个文件了。

在 Kubernetes 中，只需要了解两种结构类型就行了：

- `Lists`（列表）
- `Maps`（字典）



### Map

如果有其它编程语言基础，就知道字典，也就是 Key/Value 键值对。比如资源清单中最基础的格式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    name: myapp
```

Map 是可以嵌套 Map 的，当然也可以和 Lists 之间互相你嵌套我，我嵌套你。

注意：

> 在 YAML 文件中绝对不要使用 tab 键来进行缩进，在编辑器中可以将 tab 设置为替换成空格，一般两个空格。



### List

`List` 就是列表，说白了就是数组，在 YAML 文件中可以这样定义：

```yaml
lanuage:
  - java
  - golang
  - python
```

当然，如果数量不多，也可以一行写。

```yaml
language: ["java", "golang", "python"]
```

在日常使用中，一般都是 Map 和 List 嵌套着使用。



### 互相嵌套示例

Map 和 List 互相嵌套的示例：

```yaml
# List
users:
  # 每一项又是 Map
  - name: hello
    age: 18
    # Map 再次嵌套 List
    language:
      - chinese
      - englist
  # List 的第二项
  - name: world
    age: 20
    language:
      - chinese
```





## 编写资源清单

可以使用官方提供的文档，查看对应的资源对象支持哪些参数：

> https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#create-pod-v1-core

其实就是查看对应的接口支持哪些参数，这和开发中使用 POST，GET 请求接口的方式一样，这里请求的是 API Server 罢了。

为了方便用户使用，将请求的 JSON 换成了更容易编写的 YAML。

<br>

同时，通过前面已经了解到，可以通过 `kubectl explain 资源对象` 查看支持的配置，这也是日常使用中用的最多的。

特别注意，在 explain 的时候，如果某个字段显示的是 `required`，说明这个字段是必须的。







