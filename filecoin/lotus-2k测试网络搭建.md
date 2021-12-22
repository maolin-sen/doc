# lotus-2k测试网络搭建


## lotus-daemon配置文件详解

```shell


```

## lotus-miner配置文件详解

```shell
[API]
  # 矿工API绑定地址
  ListenAddress = "/ip4/127.0.0.1/tcp/2345/http"
  # 这应该设置为从外部可以看到的矿工 API 地址
  RemoteListenAddress = "127.0.0.1:2345"
  # 一般网络超时值
  Timeout = "30s"
[Sealing ]
#上限是指在任何给定时间开始密封之前，为了更多的交易被打包，最多有多少扇区可以等待。
#如果矿商同时接受多个交易，则最多可以创建MaxWaitDeassectors个扇区。
#如果同时接受多于MaxWaitDealAssectors个交易，则仅处理MaxWaitDealAssectors个交易
#请注意，将此数字相对于交易摄取率设置得太高可能会导致扇区打包效率低下
MaxWaitDeassectors=2
#创建新CC扇区时可以同时密封多少扇区的上限（0=无限制）
MaxSealingSectors=0
#创建有交易的新部门时，可以同时密封多少部门的上限（0=无限）
MaxSealingSectorsOrderals=0
#一段时间，一个新成立的部门将等待更多的交易被打包，然后才开始密封。
#完全填满的扇区将立即开始密封
WaitDealsDelay=“6h0m0s”
#无论客户是否要求保留未密封的交易数据副本。这让矿工
#避免了以后以占用更多存储空间为代价而启封数据的相对较高成本
AlwaysKeyUnsealedCopy=true
#在向链提交扇区证明之前运行扇区终结
finalizeeEarly=false
#启用/禁用预提交批处理（在nv13之后生效）
BatchPreCommits=true
#最大预提交批量大小达256个扇区-批量将立即发送到此大小之上
MaxPreCommitBatch=256
#超过最小批量大小后，提交批需要等待多长时间
PreCommitBatchWait=“24h0m0s”
#在批量中的扇区/交易开始过期之前，强制批量提交的时间缓冲区
PreCommitBatchSlack=“3h0m0s”
#启用/禁用提交聚合（在nv13之后生效）
AggregateCommits=true
#最小批量提交大小，不小于4
MinCommitBatch=4
#最大批量提交大小高达819个扇区-批量将立即发送到此大小之上
最大提交批次=819
#超过最小批量大小后，提交批需要等待多长时间
CommitBatchWait=“24h0m0s”
#在批量中的部门/交易开始过期之前，强制批量提交的时间缓冲区
CommitBatchSlack=“1h0m0s”
#网络基本费用，低于此费用停止执行提交聚合，而不是单独向链提交证明
AggregateAboveBaseFee=0.00000000015#0.15nanoFIL
TerminateBatchMax =100
TerminateBatchMin  =1
TerminateBatchWait=“5mos”

#存储扇区控制矿工是否可以执行某些密封操作。
[Storage]
  # 存储系统一次可以并行获取多少扇区的上限
  ParallelFetchLimit = 10
  AllowAddPiece = true
  AllowPreCommit1 = true
  AllowPreCommit2 = true
  AllowCommit = true
  AllowUnseal = true

#费用部分允许为矿工提交给链的不同消息设置gas消耗限制
[Fees]
  # Maximum fees to pay
  MaxPreCommitGasFee = "0.025 FIL"
  MaxCommitGasFee = "0.05 FIL"
  MaxTerminateGasFee = "0.5 FIL"
  MaxWindowPoStGasFee = "5 FIL"
  MaxPublishDealsFee = "0.05 FIL"
  MaxMarketBalanceAddFee = "0.007 FIL"
  [Fees.MaxPreCommitBatchGasFee]
      Base = "0 FIL"
      PerSector = "0.02 FIL"
  [Fees.MaxCommitBatchGasFee]
      Base = "0 FIL"
      PerSector = "0.03 FIL" 
```

## 创建创世块

```shell
1. lotus-seed pre-seal --sector-size=2KiB --num-sectors=2
2. lotus-seed genesis new localnet.json
3. lotus-seed genesis add-miner localnet.json ~/.genesis-sectors/pre-seal-t01000.json
```

上述命令主要是创建miner结构以及写入文件信息。

## 启动创世节点（第一个节点）

```shell
lotus daemon --lotus-make-genesis=devgen.car --genesis-template=localnet.json --bootstrap=false
```

创建全节点或者轻节点，目的运行rpcserver。（不同的节点类型，不同的rpc接口）

## 导入创世节点的钱包

```shell
lotus wallet import --as-default ~/.genesis-sectors/pre-seal-t01000.key
```

导入钱包信息之后，就可以看到你的钱包，里面会有大量的初始余额

## 初始化创世旷工

```shell
lotus-miner init --genesis-miner --actor=t01000 --sector-size=2KiB --pre-sealed-sectors=~/.genesis-sectors --pre-sealed-metadata=~/.genesis-sectors/pre-seal-t01000.json --nosync
```

创建创世矿工

## 启动创世旷工

```shell
lotus-miner run --nosync
```

创建一个节点，并启动一个矿工。

