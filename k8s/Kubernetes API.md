# Kubernetes API

Kubernetes 控制面 的核心是 API 服务器。 API 服务器负责提供 HTTP API，以供`用户`、`集群中的不同部分`和`集群外部组件`相互通信。

Kubernetes API 使你可以查询和操纵 Kubernetes API 中对象（例如：Pod、Namespace、ConfigMap 和 Event）的状态。

大部分操作都可以通过 kubectl 命令行接口或 类似 kubeadm 这类命令行工具来执行， 这些工具在背后也是调用 API。不过，你也可以使用 REST 调用来访问这些 API。

如果你正在编写程序来访问 Kubernetes API，可以考虑使用 [客户端库](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)之一。

## OpenAPI 规范

完整的 API 细节是用 [OpenAPI](https://www.openapis.org/) 来表述的。

Kubernetes API 服务器通过 /openapi/v2 末端提供 OpenAPI 规范。 你可以按照下表所给的请求头部，指定响应的格式：
|头部 |可选值 |说明
|---|---|----|
|Accept-Encoding |gzip |不指定此头部也是可以的
|Accept |application/com.github.proto-openapi.spec.v2@v1.0+protobuf |主要用于集群内部
||application/json |默认值
||* |提供application/json

## API 组和版本

为了简化删除字段或者重构资源表示等工作，Kubernetes 支持多个 API 版本， 每一个版本都在不同 API 路径下，例如 /api/v1 或 /apis/rbac.authorization.k8s.io/v1alpha1。

版本化是在 API 级别而不是在资源或字段级别进行的，目的是为了确保 API 为系统资源和行为提供清晰、一致的视图，并能够控制对已废止的和/或实验性 API 的访问。

API 资源之间靠 `API 组`、`资源类型`、`名字空间`（对于名字空间作用域的资源而言）和 `名字`来相互区分。API 服务器可能通过多个 API 版本来向外提供相同的下层数据， 并透明地完成不同 API 版本之间的转换。所有这些不同的版本实际上都是同一资源 的（不同）表现形式。例如，假定同一资源有 v1 和 v1beta1 版本， 使用 v1beta1 创建的对象则可以使用 v1beta1 或者 v1 版本来读取、更改 或者删除。

## API 扩展

有两种途径来扩展 Kubernetes API：

* 你可以使用[自定义资源](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 来以声明式方式定义 API 服务器如何提供你所选择的资源 API。
* 你也可以选择实现自己的 [聚合层](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) 来扩展 Kubernetes API。


