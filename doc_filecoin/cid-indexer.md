


## 索引器

### 概述

索引器是一个网络节点，它存储内容多重哈希到提供者数据记录的映射。想要知道一条信息存储在哪里，客户端可以使用内容的 CID 或多重哈希查询索引器，并接收提供者记录，该记录告诉客户端可以在哪里检索内容以及如何检索它。

![](image/indexer_ecosys.png)

### 术语

* Advertisement：从发布者处获得的记录，其中包含指向多哈希块链的链接、先前Advertisement CID 以及由链接的多哈希块中的所有多哈希引用的提供商特定的内容元数据。提供者数据由称为上下文 ID 的键标识。
* Announce Message：通知索引器有关广告可用性的消息。这通常通过 gossip pubsub 发送，但也可以通过 HTTP 发送。通告消息包含它正在通告的广告 CID，如果索引器已经对广告编制索引，则可以忽略该通告。发布者的地址包含在公告中，以告诉索引器从哪里检索广告。
上下文 ID：对于提供者来说，唯一标识内容元数据的键。这允许在索引器上更新或删除内容元数据，而无需使用映射到它的多重哈希来引用它。
* Gossip Pubsub：通过 libp2p gossip 网格发布/订阅通信。发布者使用它来向订阅了发送公告消息的主题的所有索引器广播公告消息。对于生产发布者和索引器，此主题是"/indexer/ingest/mainnet".
* Indexer ：一个网络节点，它保存多重哈希到提供者记录的映射。
* Metadata：检索客户端从索引器查询中获取并在检索内容时传递给提供者的特定于提供者的数据。提供者使用此元数据来识别和查找特定内容，并通过元数据中指定的协议（例如图形同步）交付该内容。
* Provider：也称为存储提供者，这是检索客户端可以从中检索内容的实体。在索引器上查找多重哈希时，响应包含提供多重哈希引用的内容的提供程序。提供者由 libp2p 对等 ID 标识。
* Publisher：这是一个将广告和索引数据发布到索引器的实体。它通常（但不总是）与数据提供者相同。发布者由 libp2p 对等 ID 标识。
* Retrieval Client：查询索引器以查找内容可用位置并从提供者检索该内容的客户端。
* Sync（索引器与发布者）：将索引器索引的内容与发布者发布的内容同步的操作。当索引器收到并宣布消息时启动同步，通过管理命令与发布者同步，或者在一段时间内（默认为 24​​ 小时）没有提供者更新时由索引器启动。


### config


```yml
{
  "Version": 2,
  "Identity": {
    "PeerID": "12D3KooWJZSPZN7cudwBJ6UdTm8V5FmgWhAd2oVa8qFagU7bZSm1",
    "PrivKey": "CAESQKP70L69Q7An97xPH6g3PgSypws6mHYcAZ77t9pC2a0/geY+Kh3gaSzfPqHpSyjy7Fd3hQPweU+BNbhhKHSHrJw="
  },
  "Addresses": {
    "Admin": "/ip4/127.0.0.1/tcp/3002",
    "Finder": "/ip4/0.0.0.0/tcp/3000",
    "Ingest": "/ip4/0.0.0.0/tcp/3001",
    "P2PAddr": "/ip4/0.0.0.0/tcp/3003",
    "NoResourceManager": false
  },
  "Bootstrap": {
    "Peers": [
      "/dns4/bootstrap-1.mainnet.filops.net/tcp/1347/p2p/12D3KooWCwevHg1yLCvktf2nvLu7L9894mcrJR4MsBCcm4syShVc",
      "/dns4/bootstrap-3.mainnet.filops.net/tcp/1347/p2p/12D3KooWKhgq8c7NQ9iGjbyK7v7phXvG6492HQfiDaGHLHLQjk7R",
      "/dns4/bootstrap-8.mainnet.filops.net/tcp/1347/p2p/12D3KooWScFR7385LTyR4zU1bYdzSiiAb5rnNABfVahPvVSzyTkR",
      "/dns4/bootstrap-0.ipfsmain.cn/tcp/34721/p2p/12D3KooWQnwEGNqcM2nAcPtRR9rAX8Hrg4k9kJLCHoTR5chJfz6d",
      "/dns4/bootstrap-0.mainnet.filops.net/tcp/1347/p2p/12D3KooWCVe8MmsEMes2FzgTpt9fXtmCY7wrq91GRiaC8PHSCCBj",
      "/dns4/bootstrap-2.mainnet.filops.net/tcp/1347/p2p/12D3KooWEWVwHGn2yR36gKLozmb4YjDJGerotAPGxmdWZx2nxMC4",
      "/dns4/bootstrap-5.mainnet.filops.net/tcp/1347/p2p/12D3KooWLFynvDQiUpXoHroV1YxKHhPJgysQGH2k3ZGwtWzR4dFH",
      "/dns4/bootstrap-6.mainnet.filops.net/tcp/1347/p2p/12D3KooWP5MwCiqdMETF9ub1P3MbCvQCcfconnYHbWg6sUJcDRQQ",
      "/dns4/bootstrap-1.starpool.in/tcp/12757/p2p/12D3KooWQZrGH1PxSNZPum99M1zNvjNFM33d1AAu5DcvdHptuU7u",
      "/dns4/lotus-bootstrap.ipfsforce.com/tcp/41778/p2p/12D3KooWGhufNmZHF3sv48aQeS13ng5XVJZ9E6qy2Ms4VzqeUsHk",
      "/dns4/bootstrap-0.starpool.in/tcp/12757/p2p/12D3KooWGHpBMeZbestVEWkfdnC9u7p6uFHXL1n7m1ZBqsEmiUzz",
      "/dns4/bootstrap-4.mainnet.filops.net/tcp/1347/p2p/12D3KooWL6PsFNPhYftrJzGgF5U18hFoaVhfGk7xwzD8yVrHJ3Uc",
      "/dns4/bootstrap-7.mainnet.filops.net/tcp/1347/p2p/12D3KooWRs3aY1p3juFjPy8gPN95PEQChm2QKGUCAdcDCC4EBMKf",
      "/dns4/node.glif.io/tcp/1235/p2p/12D3KooWBF8cpp65hp2u9LK5mh19x67ftAam84z9LsfaquTDSBpt",
      "/dns4/bootstrap-1.ipfsmain.cn/tcp/34723/p2p/12D3KooWMKxMkD5DMpSWsW7dBddKxKT7L2GgbNuckz9otxvkvByP"
    ],
    "MinimumPeers": 4
  },
  "Datastore": {
    "Dir": "datastore",
    "Type": "levelds"
  },
  "Discovery": {
    "LotusGateway": "https://api.chain.love",
    "Policy": {
      "Allow": true,
      "Except": ["12D3KooWEbhQxDZpDwvqBVPbxUXz8AquMziyUv2HT77YNKQYPiDx"],
      "Publish": true,
      "PublishExcept": null
    },
    "PollInterval": "24h0m0s",
    "PollRetryAfter": "5h0m0s",
    "PollStopAfter": "168h0m0s",
    "PollOverrides": [
      {
        "ProviderID": "12D3KooWRYLtcVBtDpBZDt5zkAVFceEHyozoQxr4giccF7fquHR2",
        "Interval": "12h0m0s",
        "RetryAfter": "45m0s",
        "StopAfter": "3h0m0s"
      }
    ],
    "RediscoverWait": "5m0s",
    "Timeout": "2m0s"
  },
  "Indexer": {
    "CacheSize": 300000,
    "ConfigCheckInterval": "30s",
    "GCInterval": "30m0s",
    "ShutdownTimeout": "10s",
    "ValueStoreDir": "valuestore",
    "ValueStoreType": "sth"
  },
  "Ingest": {
    "AdvertisementDepthLimit": 33554432,
    "EntriesDepthLimit": 65536,
    "HttpSyncRetryMax": 4,
    "HttpSyncRetryWaitMax": "30s",
    "HttpSyncRetryWaitMin": "1s",
    "HttpSyncTimeout": "10s",
    "IngestWorkerCount": 10,
    "PubSubTopic": "/indexer/ingest/mainnet",
    "RateLimit": {
      "Apply": false,
      "Except": null,
      "BlocksPerSecond": 100,
      "BurstSize": 500
    },
    "ResendDirectAnnounce": true,
    "StoreBatchSize": 4096,
    "SyncSegmentDepthLimit": 2000,
    "SyncTimeout": "2h0m0s"
  },
  "Logging": {
    "Level": "info",
    "Loggers": {
      "basichost": "warn",
      "bootstrap": "warn",
      "dt-impl": "warn",
      "dt_graphsync": "warn",
      "graphsync": "warn"
    }
  }
  "Peering": {
    "Peers": [
      "/ip4/10.11.12.13/3003/p2p/12D3KooWH1cT2UxrKYikmrksmCsdekvb6yuhxvNMup68DLpFEKZ3"
    ]
  }
}
```

## 创建索引提供者

### 索引器系统概述

这里有几个玩家。我将使用具体示例使其更清楚，但请注意，其中一些可以概括。

1. Filecoin Storage Provider – 为人们托管数据并通过 Filecoin 网络链证明它。又名存储提供者。
2. 索引器（又名storetheindex）——一种可以回答以下问题的服务：“给定这个 CID，谁有副本？”。这只不过是一个查找表。
3. Index Provider – 与 Storage Provider 一起运行的服务，它告诉 Indexer 这个存储提供者有什么内容。

索引提供者充当存储提供者和索引器之间的接口。它可以从内部使用，Lotus以便自动发布新数据。但它也可以单独发生。
索引提供者通过一系列广告消息向索引器发送更新。每条消息都引用了一个先前的广告，因此它作为一个整体形成了一个广告链。索引器的状态基本上是使用从初始广告到最新广告的这条链的函数。

### HTTP 索引提供程序

HTTP 索引提供程序需要提供两个端点：

1. GET /head
  这将返回索引提供者知道的最新广告。
2. GET /<cid>
  这将返回由给定 cid 标识的块的内容。

就是这样。使用这两个端点，索引器应该能够与提供者同步。

#### 同步时

如果索引器注册了索引提供者，它会偶尔轮询提供者以检查是否有新的内容要索引。如果您想让索引器知道您有新的更改，您可以调用Announce索引器上的端点。

### libp2p 索引提供程序

虽然这应该可以用任何语言实现，但 Go 在这里得到了最好的支持。如果您尝试以另一种语言为目标，我建议您构建一个 HTTP 索引提供程序。

在 Go 中，使用go-legs是最简单的。Legs 为 go-data-transfer 和 graphsync 提供了一个更简单的接口。

对于索引提供者，您需要设置一个 go-legs Publisher：

```go
pub, err := dtsync.NewPublisher(pubLibp2pHost, datastore, linksys, topicName)
```

当你有一个新的广告时，你会打电话给：

```go
err = pub.UpdateRoot(ctx, newAdCid)
```

这将自动在 gossipsub 频道上发送一条消息，让索引器知道此提供程序有新的更新。

go-legs 发布者还将自动处理来自索引器的数据传输请求。



