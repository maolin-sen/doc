# hyperledge工具-configtxgen

configtxgen是Hyperledger Fabric提供的用于通道配置的实用程序，主要生成以下3种文件：

* 排序服务节点使用的创世区块；
* 创建通道使用的通道配置交易；
* 更新通道用的锚节点交易。
  
目前，该工具主要侧重于生成排序服务节点的创世区块，但是将来预计增加生成新通道的配置以及重新配置已有的通道。

## configtxgen使用

```shell
`configtxgen --help`
Usage of configtxgen:
  -asOrg string
      作为特定的组织(按名称string)执行配置生成，只包括org(可能)有权设置的写集中的值。如用来指明生成的锚节点所在的组织
  -channelCreateTxBaseProfile string
      指定一个概要文件作为orderer系统通道当前状态，以允许在通道创建tx生成期间修改非应用程序参数。仅在与“outputCreateChannelTx”组合时有效。
  -channelID string
      在configtx中使用的通道ID，即通道名称，默认是"testchainid"
  -configPath string
      包含要使用的配置的路径(如果设置的话)
  -inspectBlock string
      按指定路径打印块中包含的配置，用于检查和输出通道中创世区块的内容，锚节点在configtx.yaml中的AnchorPeers中指定
  -inspectChannelCreateTx string
      按指定路径打印交易中包含的配置，用来检查通道的配置交易信息
  -outputAnchorPeersUpdate string
      创建一个配置更新来更新锚节点(仅在默认通道创建时工作，并且仅在第一次更新时工作)
  -outputBlock string
      将genesis块写入(如果设置)的路径。configtx.yaml文件中的Profiles要指定Consortiums，否则启动排序服务节点会失败
  -outputCreateChannelTx string
      将通道配置交易文件写入(如果设置)的路径。configtx.yaml文件中的Profiles必须包含Application，否则创建通道会失败
  -printOrg string
      将组织的定义打印为JSON。(对于手动向通道添加组织非常有用)
  -profile string
      指定使用的是configtx.yaml中某个用于生成的Profiles配置项。(默认为“SampleInsecureSolo”)
  -version
      显示版本信息
```
