# FISCO-BCOS 常见问题

标签：``FISCO BCOS`` ``问题排查``

----
## 1. 节点返回错误 [ImportResult::NodeIsSyncing"]

**问题分析** 
SDK或WeBase或其他客户端交易发送工具连接了正在同步区块的节点，且该节点的区块高度与其他正常共识的节点区块高度相差超过10，此时正在同步的节点会拒绝所有交易

**解决方法**
- 方法1：将正在同步节点的信息从SDK的节点连接列表中去掉，仅连接正常共识的节点
- 方法2：若因为机构的限制，SDK仅可连接这个正在同步的节点，可等到该节点区块同步完成后，再向该节点发送交易




