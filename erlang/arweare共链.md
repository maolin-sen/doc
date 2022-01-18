# arweare公链

## 命令

```sh
Usage: arweave-server [options]
Compatible with network: arweave.N.1
Options:
```

| Option | reference |
| --- | --- |
|config_file (path)                      |Load configuration from specified file.|
|peer (ip:port)                          |Join a network on a peer (or set of peers).|
|start_from_block_index                  |Start the node from the latest stored block index.|
|mine                                    |Automatically start mining once the netwok has been joined.|
|port                                    |The local port to use for mining. This port must be accessible by remote peers.|
|data_dir                                |The directory for storing the weave and the wallets (when generated).|
|metrics_dir                             |The directory for persisted metrics.|
|polling (num)                           |Poll peers for new blocks every N seconds. Default is 60. Useful in environments where port forwarding is not possible.|
|no_auto_join                            |Do not automatically join the network of your peers.|
|mining_addr (addr)                      |The address that mining rewards should be credited to. Set 'unclaimed' to send all the rewards to the endowment pool.|
|stage_one_hashing_threads (num)         |The number of mining processes searching for the SPoRA chunks to read.Default: 8. If the total number of stage one and stage two processes exceeds the number of available CPU cores, the excess processes will be hashing chunks when anything gets queued, and search for chunks otherwise.|
|io_threads (num)                        |The number of processes reading SPoRA chunks during mining. Default: 10.|
|stage_two_hashing_threads (num)         |The number of mining processes hashing SPoRA chunks.Default: 12. If the total number of stage one and stage two processes exceeds the number of available CPU cores, the excess processes will be hashing chunks when anything gets queued, and search for chunks otherwise.|
|max_emitters (num)                      |The maximum number of transaction propagation processes (default 2).|
|tx_propagation_parallelization (num)    |The maximum number of best peers to propagate transactions to at a time (default 4).|
|max_propagation_peers                   |max_propagation_peers (num)The maximum number of best peers to propagate transactions to. Default is 40.|
|max_block_propagation_peers             |max_block_propagation_peers (num)The maximum number of best peers to propagate blocks to. Default is 50.|
|sync_jobs (num)                         |The number of data syncing jobs to run. Default: 20. Each job periodically picks a range and downloads it from peers.|
|header_sync_jobs (num)                  |The number of header syncing jobs to run. Default: 1. Each job periodically picks the latest not synced block header and downloads it from peers.
|load_mining_key (file)                  |Load the address that mining rewards should be credited to from file.|
|ipfs_pin                                |Pin incoming IPFS tagged transactions on your local IPFS node.|
|transaction_blacklist (file)           |A file containing blacklisted transactions. One Base64 encoded transaction ID per line.|
|transaction_blacklist_url               |An HTTP endpoint serving a transaction blacklist.|
|transaction_whitelist (file)            |A file containing whitelisted transactions. One Base64 encoded transaction ID per line. If a transaction is in both lists, it is considered whitelisted.|
|transaction_whitelist_url               |An HTTP endpoint serving a transaction whitelist.|
|disk_space (num)                        |Max size (in GB) for the disk partition containing the Arweave data directory (blocks, txs, etc) when the miner stops writing files to disk.|
|disk_space_check_frequency (num)        |The frequency in seconds of requesting the information about the available disk space from the operating system, used to decide on whether to continue syncing the historical data or clean up some space. Default is 30.|
|init                                    |Start a new weave.|
|internal_api_secret (secret)            |Enables the internal API endpoints, only accessible with this secret. Min. 16 chars.|
|enable (feature)                        |Enable a specific (normally disabled) feature. For example, subfield_queries.|
|disable (feature)                       |Disable a specific (normally enabled) feature.|
|gateway (domain)                        |Run a gateway on the specified domain|
|custom_domain (domain)                  |Add a domain to the list of supported custom domains.|
|requests_per_minute_limit (number)      |Limit the maximum allowed number of HTTP requests per IP address per minute. Default is 900.|
|max_connections                         |The number of connections to be handled concurrently. Its purpose is to prevent your system from being overloaded and ensuring all the connections are |handled optimally. Default is 1024.|
|max_gateway_connections                 |The number of gateway connections to be handled concurrently. Default is 128.|
|max_poa_option_depth                    |The number of PoA alternatives to try until the recall data is found. Has to be an integer > 1. The mining difficulty grows linearly as a function of |the alternative as (0.75 + 0.25 * number) * diff, up to (0.75 + 0.25 * max_poa_option_depth) *diff. Default is 500.|
|disk_pool_data_root_expiration_time     |The time in seconds of how long a pending or orphaned data root is kept in the disk pool. The default is 2* 60 * 60 (2 hours).|
|max_disk_pool_buffer_mb                 |The max total size in mebibytes of the pending chunks in the disk pool.The default is 2000 (2 GiB).|
|max_disk_pool_data_root_buffer_mb       |The max size in mebibytes per data root of the pending chunks in the disk pool. The default is 50.|
|randomx_bulk_hashing_iterations         |The number of hashes RandomX generates before reporting the result back to the Arweave miner. The faster CPU hashes, the higher this value should be.|
|disk_cache_size_mb                      |The maximum size in mebibytes of the disk space allocated for storing recent block and transaction headers. Default is 5120.|
|packing_rate                            |The maximum number of chunks per second to pack or unpack. Default: 15.|
|debug                                   |Enable extended logging.|

启动之后

```shell
2022-01-13T22:56:56.042087+08:00 [info] /opt/arweave/apps/arweave/src/ar_meta_db.erl:97 event: `ar_meta_db_start`
2022-01-13T22:56:56.042290+08:00 [info] /opt/arweave/apps/arweave/src/ar_arql_db.erl:173 `ar_arql_db: init, data_dir: /ar/ar`
2022-01-13T22:56:59.525790+08:00 [info] /opt/arweave/apps/arweave/src/ar_header_sync.erl:47 event: `ar_header_sync_start`
2022-01-13T22:56:59.542534+08:00 [info] /opt/arweave/apps/arweave/src/ar_data_sync.erl:208 event: `ar_data_sync_start`
2022-01-13T22:57:00.730693+08:00 [info] /opt/arweave/apps/arweave/src/ar_blacklist_middleware.erl:29 event: `ar_blacklist_middleware_start`
2022-01-13T22:57:00.733231+08:00 [info] /opt/arweave/apps/arweave/src/ar_poller.erl:34 event: `ar_poller_start`
```

## 共享区块存储部署方案

不共享文件：

`data_sync_state文件`
`chunk_storage_index文件`
`rocksdb文件夹`

其他共享。
