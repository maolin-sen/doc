# 使用 Kubernetes 对象

* 理解 Kubernetes 对象
* Kubernetes 对象管理
* 对象名称和 IDs
* 名字空间
* 标签和选择算符
* 注解
* Finalizers
* 字段选择器
* 属主与附属
* 推荐使用的标签

## 理解 Kubernetes 对象

本页说明了 Kubernetes 对象在 Kubernetes API 中是如何表示的，以及如何在 .yaml 格式的文件中表示。

### 详细理解 Kubernetes 对象

在 Kubernetes 系统中，Kubernetes 对象 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

* 哪些容器化应用在运行（以及在哪些节点上）
* 可以被应用使用的资源
* 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 期望状态（Desired State）。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 Kubernetes API。

#### 对象规约（Spec）与状态（Status）

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： `对象 spec（规约）` 和 `对象 status（状态）` 。 对于具有 spec 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： `期望状态（Desired State）` 。

status 描述了对象的 `当前状态（Current State）`，它是由 Kubernetes 系统和组件 设置并更新的。在任何时刻，Kubernetes 控制平面 都一直积极地管理着对象的实际状态，以使之与期望状态相匹配。

#### 描述 Kubernetes 对象

创建 Kubernetes 对象时，必须提供对象的规约，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于kubectl）， API 请求必须在请求体中包含 JSON 格式的信息。 大多数情况下，需要在 .yaml 文件中为 kubectl 提供这些信息。 `kubectl` 在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 `.yaml` 示例文件，展示了 Kubernetes Deployment 的必需字段和对象规约：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

#### 必需字段

在想要创建的 Kubernetes 对象对应的 .yaml 文件中，需要配置如下的字段：

`apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
`kind` - 想要创建的对象的类别
`metadata` - 帮助唯一性标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace
`spec` - 你所期望的该对象的状态

## Kubernetes 对象管理

## 对象名称和 IDs

集群中的每一个对象都有一个名称 来标识在同类资源中的唯一性。每个 Kubernetes 对象也有一个UID 来标识在整个集群中的唯一性。

比如，在同一个名字空间 中有一个名为 myapp-1234 的 Pod, 但是可以命名一个 Pod 和一个 Deployment 同为 myapp-1234.

对于用户提供的非唯一性的属性，Kubernetes 提供了 标签（Labels）和 注解（Annotation）机制。

### 名称

客户端提供的字符串，引用资源 url 中的对象，如/api/v1/pods/some name。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

以下是比较常见的四种资源命名约束。

* DNS 子域名
* RFC 1123 标签名
* RFC 1035 标签名
* 路径分段名称

### UIDs

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 uid，它旨在区分类似实体的历史事件。

Kubernetes UIDs 是全局唯一标识符（也叫 UUIDs）。 UUIDs 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667.

## 名字空间

Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。 在一些文档里名字空间也称为命名空间。

### 何时使用多个名字空间

名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名称空间提供的功能时，请开始使用它们。

名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。

名字空间是在多个用户之间划分集群资源的一种方法（通过资源配额）。

不必使用多个名字空间来分隔仅仅轻微不同的资源，例如同一软件的不同版本： 应该使用标签 来区分同一名字空间中的不同资源。

## 标签和选择算符

标签（Labels） 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。标签能够支持高效的查询和监听操作，对于用户界面和命令行是很理想的。 应使用注解 记录非识别信息。

```yaml
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

## 注解

你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。客户端程序（例如工具和库）能够获取这些元数据信息。

### 为对象附加元数据

你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。 相反，注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

注解和标签一样，是键/值对:

```yaml
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### Finalizers

Finalizer 是带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源。 Finalizer 提醒控制器清理被删除的对象拥有的资源。

当你告诉 Kubernetes 删除一个指定了 Finalizer 的对象时， Kubernetes API 会将该对象标记为删除，使其进入只读状态。 此时控制平面或其他组件会采取 Finalizer 所定义的行动， 而目标对象仍然处于终止中（Terminating）的状态。 这些行动完成后，控制器会删除目标对象相关的 Finalizer。 当 metadata.finalizers 字段为空时，Kubernetes 认为删除已完成。

你可以使用 Finalizer 控制资源的垃圾收集。 例如，你可以定义一个 Finalizer，在删除目标资源前清理相关资源或基础设施。

你可以通过使用 Finalizers 提醒控制器 在删除目标资源前执行特定的清理任务， 来控制资源的垃圾收集。

Finalizers 通常不指定要执行的代码。 相反，它们通常是特定资源上的键的列表，类似于注解。 Kubernetes 自动指定了一些 Finalizers，但你也可以指定你自己的。

### Finalizers 如何工作

当你使用清单文件创建资源时，你可以在 metadata.finalizers 字段指定 Finalizers。 当你试图删除该资源时，管理该资源的控制器会注意到 finalizers 字段中的值， 并进行以下操作：

* 修改对象，将你开始执行删除的时间添加到 metadata.deletionTimestamp 字段。
* 将该对象标记为只读，直到其 metadata.finalizers 字段为空。

然后，控制器试图满足资源的 Finalizers 的条件。 每当一个 Finalizer 的条件被满足时，控制器就会从资源的 finalizers 字段中删除该键。 当该字段为空时，垃圾收集继续进行。 你也可以使用 Finalizers 来阻止删除未被管理的资源。

一个常见的 Finalizer 的例子是 kubernetes.io/pv-protection， 它用来防止意外删除 PersistentVolume 对象。 当一个 PersistentVolume 对象被 Pod 使用时， Kubernetes 会添加 pv-protection Finalizer。 如果你试图删除 PersistentVolume，它将进入 Terminating 状态， 但是控制器因为该 Finalizer 存在而无法删除该资源。 当 Pod 停止使用 PersistentVolume 时， Kubernetes 清除 pv-protection Finalizer，控制器就会删除该卷。

### 属主引用、标签和 Finalizers

与标签类似， 属主引用 描述了 Kubernetes 中对象之间的关系，但它们作用不同。 当一个控制器 管理类似于 Pod 的对象时，它使用标签来跟踪相关对象组的变化。 例如，当 Job 创建一个或多个 Pod 时， Job 控制器会给这些 Pod 应用上标签，并跟踪集群中的具有相同标签的 Pod 的变化。

Job 控制器还为这些 Pod 添加了属主引用，指向创建 Pod 的 Job。 如果你在这些 Pod 运行的时候删除了 Job， Kubernetes 会使用属主引用（而不是标签）来确定集群中哪些 Pod 需要清理。

当 Kubernetes 识别到要删除的资源上的属主引用时，它也会处理 Finalizers。

在某些情况下，Finalizers 会阻止依赖对象的删除， 这可能导致目标属主对象，保持在只读状态的时间比预期的长，且没有被完全删除。 在这些情况下，你应该检查目标属主和附属对象上的 Finalizers 和属主引用，来排查原因。

### 字段选择器

字段选择器（Field selectors）允许你根据一个或多个资源字段的值 筛选 Kubernetes 资源。 下面是一些使用字段选择器查询的例子：

* metadata.name=my-service
* metadata.namespace!=default
* status.phase=Pending
下面这个 kubectl 命令将筛选出 status.phase 字段值为 Running 的所有 Pod：

```shell
kubectl get pods --field-selector status.phase=Running
```

### 支持的操作符

你可在字段选择器中使用 =、==和 != （= 和 == 的意义是相同的）操作符。

## 属主与附属

在 Kubernetes 中，一些对象是其他对象的属主（Owner）。 例如，ReplicaSet 是一组 Pod 的属主。 具有属主的对象是属主的附属（Dependent） 。

### 对象规约中的属主引用

附属对象有一个 metadata.ownerReferences 字段，用于引用其属主对象。
附属对象还有一个 ownerReferences.blockOwnerDeletion 字段，该字段使用布尔值， 用于控制特定的附属对象是否可以阻止垃圾收集删除其属主对象。 如果控制器（例如 Deployment 控制器） 设置了 metadata.ownerReferences 字段的值，Kubernetes 会自动设置 blockOwnerDeletion 的值为 true。 你也可以手动设置 blockOwnerDeletion 字段的值，以控制哪些附属对象会阻止垃圾收集。

### 属主关系与 Finalizer

当你告诉 Kubernetes 删除一个资源，API 服务器允许管理控制器处理该资源的任何 Finalizer 规则。 Finalizer 防止意外删除你的集群所依赖的、用于正常运作的资源。

## 推荐使用的标签

共享标签和注解都使用同一个前缀：app.kubernetes.io。没有前缀的标签是用户私有的。共享前缀可以确保共享标签不会干扰用户自定义的标签。

### 标签

为了充分利用这些标签，应该在每个资源对象上都使用它们。
|键 |描述 |示例 |类型
|---|---|---|---|
|app.kubernetes.io/name |应用程序的名称 |mysql |字符串
|app.kubernetes.io/instance |用于唯一确定应用实例的名称 |mysql-abcxzy |字符串
|app.kubernetes.io/version |应用程序的当前版本（例如，语义版本，修订版哈希等） |5.7.21 |字符串
|app.kubernetes.io/component |架构中的组件 |database |字符串
|app.kubernetes.io/part-of |此级别的更高级别应用程序的名称 |wordpress |字符串
|app.kubernetes.io/managed-by |用于管理应用程序的工具 |helm |字符串
|app.kubernetes.io/created-by |创建该资源的控制器或者用户 |controller-manager |字符串

