# Filecoin 存储仓库

filecoin 的存储仓库也就是 repo 表现为文件系统上的一个目录， 用来存放的一些程序相关的数据。 在 filecoin 中有4中类型的仓库.

```go
const (
	_                 = iota   // Default is invalid
	FullNode RepoType = iota   // 全节点， 用来存储daemon 相关的数据
	StorageMiner               // 用来存储miner 相关数据
	Worker                     // 用来存储 Worker 相关的数据
	Wallet                     // 用来存储 Wallet 相关的数据
)
```

每个仓库都规定一些有特定意义文件的名字.

```go
const (
	fsAPI           = "api"  
	fsAPIToken      = "token"
	fsConfig        = "config.toml"
	fsStorageConfig = "storage.json"
	fsDatastore     = "datastore"
	fsLock          = "repo.lock"
	fsKeystore      = "keystore"
)
```

## 存储仓库的数据结构

存储仓库结构体表示：

```go
// FsRepo is struct for repo, use NewFS to create
type FsRepo struct {
	path string        //仓库路径
	configPath string  //配置文件路径
}
```
你可以采用下面的方式创建一个存储仓库：

```go
func NewFS(path string) (*FsRepo, error)
```

在创建仓库后， 你可以传递一个仓库类型， 初始化仓库：

```go
func (fsr *FsRepo) Init(t RepoType) error 
```

LockedRepo 是 Repo 仓库的近一步实现，其带有文件锁以便进程之间的互斥使用， 并且实现其他的更多功能：

```go
type fsLockedRepo struct {
	path       string
	configPath string
	repoType   RepoType
	closer     io.Closer
	readonly   bool

	ds     map[string]datastore.Batching
	dsErr  error
	dsOnce sync.Once

	bs     blockstore.Blockstore
	bsErr  error
	bsOnce sync.Once
	ssPath string
	ssErr  error
	ssOnce sync.Once

	storageLk sync.Mutex
	configLk  sync.Mutex
}
```

LockedRepo 只能由 Repo 创建， 你可以采用下面的方式创建 LockedRepo

```go
func (fsr *FsRepo) Lock(repoType RepoType) (LockedRepo, error)
```