# k8s架构

## 节点

ubernetes 通过将容器放入在节点（Node）上运行的 Pod 中来执行你的工作负载。 节点可以是一个虚拟机或者物理机器，取决于所在的集群配置。 每个节点包含运行 Pods 所需的服务； 这些节点由 控制面 负责管理。节点上的组件包括 kubelet、 容器运行时以及 kube-proxy。

## 管理

向 API 服务器添加节点的方式主要有两种：

1. 节点上的 kubelet 向控制面执行自注册；
2. 你，或者别的什么人，手动添加一个 Node 对象。

在你创建了 Node 对象或者节点上的 kubelet 执行了自注册操作之后， 控制面会检查新的 Node 对象是否合法。例如，如果你使用下面的 JSON 对象来创建 Node 对象：

```yaml
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

Kubernetes 会在内部创建一个 Node 对象作为节点的表示。Kubernetes 检查 kubelet 向 API 服务器注册节点时使用的 metadata.name 字段是否匹配。 如果节点是健康的（即所有必要的服务都在运行中），则该节点可以用来运行 Pod。 否则，直到该节点变为健康之前，所有的集群活动都会忽略该节点。

`说明： Kubernetes 会一直保存着非法节点对应的对象，并持续检查该节点是否已经 变得健康。 你，或者某个控制器必需显式地 删除该 Node 对象以停止健康检查操作。`

### 节点自注册

当 kubelet 标志 --register-node 为 true（默认）时，它会尝试向 API 服务注册自己。 这是首选模式，被绝大多数发行版选用。

对于自注册模式，kubelet 使用下列参数启动：

* `--kubeconfig` - 用于向 API 服务器表明身份的凭据路径。
* `--cloud-provider` - 与某云驱动 进行通信以读取与自身相关的元数据的方式。
* `--register-node` - 自动向 API 服务注册。
* `--register-with-taints` - 使用所给的污点列表注册节点。 当 register-node 为 false 时无效。
* `--node-ip` - 节点 IP 地址。
* `--node-labels` - 在集群中注册节点时要添加的 标签。 （参见 NodeRestriction 准入控制插件所实施的标签限制）。
* `--node-status-update-frequency` - 指定 kubelet 向控制面发送状态的频率。

启用节点授权模式和 NodeRestriction 准入插件 时，仅授权 kubelet 创建或修改其自己的节点资源。

### 手动节点管理

手动节点管理
你可以使用 kubectl 来创建和修改 Node 对象。

如果你希望手动创建节点对象时，请设置 kubelet 标志 --register-node=false。

你可以修改 Node 对象（忽略 --register-node 设置）。 例如，修改节点上的标签或标记其为不可调度。

你可以结合使用节点上的标签和 Pod 上的选择算符来控制调度。 例如，你可以限制某 Pod 只能在符合要求的节点子集上运行。

如果标记节点为不可调度（unschedulable），将阻止新 Pod 调度到该节点之上，但不会 影响任何已经在其上的 Pod。 这是重启节点或者执行其他维护操作之前的一个有用的准备步骤。

要标记一个节点为不可调度，执行以下命令：

```shell
kubectl cordon $NODENAME
```

## 控制面到节点通信

本文列举控制面节点（确切说是 API 服务器）和 Kubernetes 集群之间的通信路径。 目的是为了让用户能够自定义他们的安装，以实现对网络配置的加固，使得集群能够在不可信的网络上 （或者在一个云服务商完全公开的 IP 上）运行。

### 节点到控制面

Kubernetes 采用的是中心辐射型（Hub-and-Spoke）API 模式。 所有从集群（或所运行的 Pods）发出的 API 调用都终止于 apiserver。 其它控制面组件都没有被设计为可暴露远程服务。 apiserver 被配置为在一个安全的 HTTPS 端口（通常为 443）上监听远程连接请求， 并启用一种或多种形式的客户端身份认证机制。 一种或多种客户端鉴权机制应该被启用， 特别是在允许使用匿名请求 或服务账号令牌的时候。

应该使用集群的公共根证书开通节点，这样它们就能够基于有效的客户端凭据安全地连接 apiserver。 一种好的方法是以客户端证书的形式将客户端凭据提供给 kubelet。 请查看 kubelet TLS 启动引导 以了解如何自动提供 kubelet 客户端证书。

想要连接到 apiserver 的 Pod 可以使用服务账号安全地进行连接。 当 Pod 被实例化时，Kubernetes 自动把公共根证书和一个有效的持有者令牌注入到 Pod 里。 kubernetes 服务（位于 default 名字空间中）配置了一个虚拟 IP 地址，用于（通过 kube-proxy）转发 请求到 apiserver 的 HTTPS 末端。

控制面组件也通过安全端口与集群的 apiserver 通信。

这样，从集群节点和节点上运行的 Pod 到控制面的连接的缺省操作模式即是安全的， 能够在不可信的网络或公网上运行。

### 控制面到节点

从控制面（apiserver）到节点有两种主要的通信路径。 第一种是从 apiserver 到集群中每个节点上运行的 kubelet 进程。 第二种是从 apiserver 通过它的代理功能连接到任何节点、Pod 或者服务。

#### API 服务器到 kubelet

从 apiserver 到 kubelet 的连接用于：

* 获取 Pod 日志
* 挂接（通过 kubectl）到运行中的 Pod
* 提供 kubelet 的端口转发功能。
  
这些连接终止于 kubelet 的 HTTPS 末端。 默认情况下，apiserver 不检查 kubelet 的服务证书。这使得此类连接容易受到中间人攻击， 在非受信网络或公开网络上运行也是 不安全的。

为了对这个连接进行认证，使用 --kubelet-certificate-authority 标志给 apiserver 提供一个根证书包，用于 kubelet 的服务证书。

如果无法实现这点，又要求避免在非受信网络或公共网络上进行连接，可在 apiserver 和 kubelet 之间使用 SSH 隧道。

最后，应该启用 kubelet 用户认证和/或鉴权 来保护 kubelet API。

#### apiserver 到节点、Pod 和服务

从 apiserver 到节点、Pod 或服务的连接默认为纯 HTTP 方式，因此既没有认证，也没有加密。 这些连接可通过给 API URL 中的节点、Pod 或服务名称添加前缀 https: 来运行在安全的 HTTPS 连接上。 不过这些连接既不会验证 HTTPS 末端提供的证书，也不会提供客户端证书。 因此，虽然连接是加密的，仍无法提供任何完整性保证。 这些连接 目前还不能安全地 在非受信网络或公共网络上运行。

##### SSH 隧道

Kubernetes 支持使用 SSH 隧道来保护从控制面到节点的通信路径。在这种配置下，apiserver 建立一个到集群中各节点的 SSH 隧道（连接到在 22 端口监听的 SSH 服务） 并通过这个隧道传输所有到 kubelet、节点、Pod 或服务的请求。 这一隧道保证通信不会被暴露到集群节点所运行的网络之外。

SSH 隧道目前已被废弃。除非你了解个中细节，否则不应使用。 Konnectivity 服务是对此通信通道的替代品。

##### Konnectivity 服务

作为 SSH 隧道的替代方案，Konnectivity 服务提供 TCP 层的代理，以便支持从控制面到集群的通信。 Konnectivity 服务包含两个部分：Konnectivity 服务器和 Konnectivity 代理，分别运行在 控制面网络和节点网络中。Konnectivity 代理建立并维持到 Konnectivity 服务器的网络连接。 启用 Konnectivity 服务之后，所有控制面到节点的通信都通过这些连接传输。


