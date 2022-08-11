# Sectors状态

扇区是Filecoin网络中的数据存储单元，目前主网的扇区大小有32GiB和64GiB。

```shell
*   Empty
  |   |
  |   v
  *<- Packing <- incoming
  |   |
  |   v
  *<- Unsealed <--> SealFailed
  |   |
  |   v
  *   PreCommitting <--> PreCommitFailed
  |   |                  ^
  |   v                  |
  *<- WaitSeed ----------/
  |   |||
  |   vvv      v--> SealCommitFailed
  *<- Committing
  |   |        ^--> CommitFailed
  |   v             ^
  *<- CommitWait ---/
  |   |
  |   v
  *<-FinalizeSector <--> FinalizeFailed
  |
  |
  *<- Proving --> Faulty
  |
  |
  v
  FailedUnrecoverable

  UndefinedSectorState <- ¯\_(ツ)_/¯
      |                     ^
      *---------------------/
-----------------------------------
```


