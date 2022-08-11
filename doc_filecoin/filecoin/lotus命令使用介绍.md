# louts命令使用

```shell
NAME:
   lotus - Filecoin decentralized storage network client

USAGE:
   lotus [global options] command [command options] [arguments...]

VERSION:
   1.13.3-dev+debug+git.9ccd4ee24

COMMANDS:
   daemon         Start a lotus daemon process
   backup         Create node metadata backup
   config         Manage node config
   advance-block  
   version        Print version
   help, h        Shows a list of commands or help for one command
   BASIC:
     send     Send funds between accounts
     wallet   Manage wallet
     client   Make deals, store data, retrieve data
     msig     Interact with a multisig wallet
     filplus  Interact with the verified registry actor used by Filplus
     paych    Manage payment channels
   DEVELOPER:
     auth          Manage RPC permissions
     mpool         Manage message pool
     state         Interact with and query filecoin chain state
     chain         Interact with filecoin blockchain
     log           Manage logging
     wait-api      Wait for lotus api to come online
     fetch-params  Fetch proving parameters
   NETWORK:
     net   Manage P2P Network
     sync  Inspect or interact with the chain syncer
   STATUS:
     status  Check node status

GLOBAL OPTIONS:
   --interactive  setting to false will disable interactive functionality of commands (default: true)
   --force-send   if true, will ignore pre-send checks (default: false)
   --vv           enables very verbose mode, useful for debugging the CLI (default: false)
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

## lotus-seed

```shell
NAME:
   lotus-seed - Seal sectors for genesis miner

USAGE:
   lotus-seed [global options] command [command options] [arguments...]

COMMANDS:
   genesis              
   pre-seal             
   aggregate-manifests  aggregate a set of preseal manifests into a single file
   help, h              Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --sector-dir value  (default: "~/.genesis-sectors")
   --help, -h          show help (default: false)
   --version, -v       print the version (default: false)
```

### lotus-seed pre-seal

```shell
NAME:
   lotus-seed pre-seal 

USAGE:
   lotus-seed pre-seal [command options] [arguments...]

OPTIONS:
   --miner-addr value       specify the future address of your miner (default: "t01000")
   --sector-size value      specify size of sectors to pre-seal (default: "2KiB")
   --ticket-preimage value  set the ticket preimage for sealing randomness (default: "lotus is fire")
   --num-sectors value      select number of sectors to pre-seal (default: 1)
   --sector-offset value    how many sector ids to skip when starting to seal (default: 0)
   --key value              (optional) Key to use for signing / owner/worker addresses
   --fake-sectors           (default: false)
   --network-version value  specify network version (default: 14)
   --help, -h               show help (default: false)
```

### lotus-seed genesis

```shell
NAME:
   lotus-seed genesis - A new cli application

USAGE:
   lotus-seed genesis command [command options] [arguments...]

DESCRIPTION:
   manipulate lotus genesis template

COMMANDS:
   new                  
   add-miner            
   add-msigs            
   set-vrk              Set the verified registry\`s root key
   set-remainder        Set the remainder actor
   set-network-version  Set the version that this network will start from
   car                  
   help, h              Shows a list of commands or help for one command
OPTIONS:
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

## lotus

```shell
NAME:
   lotus - Filecoin decentralized storage network client

USAGE:
   lotus [global options] command [command options] [arguments...]

COMMANDS:
   daemon         Start a lotus daemon process
   backup         Create node metadata backup
   config         Manage node config
   advance-block  
   version        Print version
   help, h        Shows a list of commands or help for one command
   BASIC:
     send     Send funds between accounts
     wallet   Manage wallet
     client   Make deals, store data, retrieve data
     msig     Interact with a multisig wallet
     filplus  Interact with the verified registry actor used by Filplus
     paych    Manage payment channels
   DEVELOPER:
     auth          Manage RPC permissions
     mpool         Manage message pool
     state         Interact with and query filecoin chain state
     chain         Interact with filecoin blockchain
     log           Manage logging
     wait-api      Wait for lotus api to come online
     fetch-params  Fetch proving parameters
   NETWORK:
     net   Manage P2P Network
     sync  Inspect or interact with the chain syncer
   STATUS:
     status  Check node status

GLOBAL OPTIONS:
   --interactive  setting to false will disable interactive functionality of commands (default: true)
   --force-send   if true, will ignore pre-send checks (default: false)
   --vv           enables very verbose mode, useful for debugging the CLI (default: false)
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

## lotus-miner

```shell
NAME:
   lotus-miner - Filecoin decentralized storage network miner

USAGE:
   lotus-miner [global options] command [command options] [arguments...]

VERSION:
   1.13.3-dev+debug+git.9ccd4ee24

COMMANDS:
   init     Initialize a lotus miner repo
   run      Start a lotus miner process
   stop     Stop a running lotus miner
   config   Manage node config
   backup   Create node metadata backup
   version  Print version
   help, h  Shows a list of commands or help for one command
   CHAIN:
     actor  manipulate the miner actor
     info   Print miner info
   DEVELOPER:
     auth          Manage RPC permissions
     log           Manage logging
     wait-api      Wait for lotus api to come online
     fetch-params  Fetch proving parameters
   MARKET:
     storage-deals    Manage storage deals and related configuration
     retrieval-deals  Manage retrieval deals and related configuration
     data-transfers   Manage data transfers
     dagstore         Manage the dagstore on the markets subsystem
   NETWORK:
     net  Manage P2P Network
   RETRIEVAL:
     pieces  interact with the piecestore
   STORAGE:
     sectors  interact with sector store
     proving  View proving information
     storage  manage sector storage
     sealing  interact with sealing pipeline

GLOBAL OPTIONS:
   --actor value, -a value                  specify other actor to query / manipulate
   --color                                  use color in display output (default: depends on output being a TTY)
   --miner-repo value, --storagerepo value  Specify miner repo path. flag(storagerepo) and env(LOTUS_STORAGE_PATH) are DEPRECATION, will REMOVE SOON (default: "~/.lotusminer") [$LOTUS_MINER_PATH, $LOTUS_STORAGE_PATH]
   --markets-repo value                     Markets repo path [$LOTUS_MARKETS_PATH]
   --call-on-markets                        (experimental; may be removed) call this command against a markets node; use only with common commands like net, auth, pprof, etc. whose target may be ambiguous (default: false)
   --vv                                     enables very verbose mode, useful for debugging the CLI (default: false)
   --help, -h                               show help (default: false)
   --version, -v                            print the version (default: false)
```

## lotus-worker

```shell
NAME:
   lotus-worker - Remote miner worker

USAGE:
   lotus-worker [global options] command [command options] [arguments...]

VERSION:
   1.13.3-dev+debug+git.9ccd4ee24

COMMANDS:
   run         Start lotus worker
   info        Print worker info
   storage     manage sector storage
   set         Manage worker settings
   wait-quiet  Block until all running tasks exit
   resources   Manage resource table overrides
   tasks       Manage task processing
   help, h     Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --worker-repo value, --workerrepo value  Specify worker repo path. flag workerrepo and env WORKER_PATH are DEPRECATION, will REMOVE SOON (default: "~/.lotusworker") [$LOTUS_WORKER_PATH, $WORKER_PATH]
   --miner-repo value, --storagerepo value  Specify miner repo path. flag storagerepo and env LOTUS_STORAGE_PATH are DEPRECATION, will REMOVE SOON (default: "~/.lotusminer") [$LOTUS_MINER_PATH, $LOTUS_STORAGE_PATH]
   --enable-gpu-proving                     enable use of GPU for mining operations (default: true)
   --help, -h                               show help (default: false)
   --version, -v                            print the version (default: false)
```

