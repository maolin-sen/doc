# 名词解释

1. `抵押扇区`是一种用随机数据密封扇区以增加矿工在网络中的权力的技术。通过用随机数据密封扇区，矿工可以向网络证明，当有足够的需求或决定这样做时，它有可能为真实交易提供那么多存储。这就是所谓的质押。

2. lotus主要应用：
   1. lotus：在每个 Lotus 节点上运行 Lotus 守护进程。此应用程序可以
      1. 验证交易
      2. 管理 FIL 地址
      3. 存储和检索交易
   2. lotus-miner：
   3. lotus-worker:协助矿工执行采矿相关任务的工人

3. 常用环境变量说明：
   1. `FIL_PROOFS_PARAMETER_CACHE` <=> proof 证明参数路径，默认在/var/tmp/filecoin-proof-parameters下

   2. `FFI_BUILD_FROM_SOURCE` <=> 从源码编译底层库

   3. `IPFS_GATEWAY` <=> 配置证明参数下载的代理地址,默认<https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/>
  
   4. `TMPDIR` <=> 临时文件夹路径，用于存放显卡锁定文件

   5. `RUST_LOG` <=> 配置Rust日志级别
  
   6. `GOPROXY` <=> 配置Golang代理

4. Lotus Deamon环境变量:
   1. `LOTUS_PATH` <=> lotus daemon 路径

5. Lotus Miner环境变量:
   1. `LOTUS_MINER_PATH` <=> lotus miner 路径

   2. `FULLNODE_API_INFO` <=> lotus daemon API 环境变量

   3. `BELLMAN_CUSTOM_GPU` <=> 指定GPU型号

6. Lotus Worker环境变量:
   1. `LOTUS_WORKER_PATH` <=> Lotus worker 路径

   2. `FIL_PROOFS_PARENT_CACHE` <=> Parent cache 参数

   3. `FIL_PROOFS_MAXIMIZE_CACHING` <=> 最大化内存参数

   4. `FIL_PROOFS_USE_MULTICORE_SDR` <=> CPU多核心绑定

   5. `FIL_PROOFS_USE_GPU_TREE_BUILDER` <=> 使用GPU计算Precommit2 TREE hash

   6. `FIL_PROOFS_USE_GPU_COLUMN_BUILDER` <=> 使用GUP计算Precommit2 COLUMN hash

   7. `BELLMAN_NO_GPU` <=> 不使用GPU计算Commit2

   8. `MINER_API_INFO` <=> Lotus miner的API信息

   9. `BELLMAN_CUSTOM_GPU` <=> 指定Commit2的GPU型号
