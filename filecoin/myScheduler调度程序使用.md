# myScheduler调度程序使用

## myScheduler 调度程序简介

目前官方窗口调度Scheduler工作效果不理想，有的worker工作繁忙，有的 worker 又非常轻闲。个人认为filecoin运维人员最为熟悉自己的设备，所以基于运维人员的工作任务指令，重新编写了一套调度程序，摈弃官方窗口调度和纯基于设备资源排序的规则，重新自定义排序规则，总体调度目标是让相关任务均匀分配到各个可以执行的 worker 上面，调度程序支持以下业务功能：

1. ① 支持90% 以上的设备利用率；
2. ② 由你的运维人员，设置每个 Worker 最大的 AP/P1/P2/C2 工作任务数量；
3. ③ 支持单 Worker 任意的 AP/P1/P2/C2 数量配比和工作并行；
4. ④ 支持单 Worker AP/P1 绑定（免传输32GB），P1/P2绑定（免传输453GB），P2/C2 绑定调度，如果三项全部绑定，则该 Worker 进入独立工作模式；
5. ⑤ 支持 Worker 待离线维护工作模式，不接新任务，完成手头任务后进行安全下线维护；
6. ⑥ 实时监控 Worker 本地 cache/sealed/unsealed 磁盘空间使用，当空间不足时，自动进入空间紧缩工作模式，支持 Worker 在 out of space 不接收任务工作分配情形下进行特别工作处理；
7. ⑦ 以上 Worker 参数，均可以在 Worker 运行过程中动态调整、实时生效；
8. ⑧ lotus-miner sectors mypledge 一次性填充所有 lotus-worker 可以工作AP的数量总和，可用自动化任务定时调用 mypledge

## lotus-worker 动态参数设置

如果首次启动 lotus-worker，调度程序会自动将 run 相关的参数写进 $LOTUS_WORKER_PATH 路径下面的参数文件 myscheduler.json， miner 本地worker，则是在 $LOTUS_MINER_PATH 目录创建此配置文件，并且设置为默认初始状态，相关参数均可以在不重启 lotus-worker 的状态下，动态修改并实时生效。以下是默认参数：

```shell
{
  "WorkerName": "",
  "AddPieceMax": 0,
  "PreCommit1Max": 0,
  "PreCommit2Max": 0,
  "Commit2Max": 0,
  "DiskHoldMax": 0,
  "APDiskHoldMax": 0,
  "ForceP1FromLocalAP": true,
  "ForceP2FromLocalP1": true,
  "ForceC2FromLocalP2": false,
  "IsPlanOffline": false,
  "AllowP2C2Parallel": false,
  "IgnoreOutOfSpace": false,
  "AutoPledgeDiff": 0
}
```

## 调度外部 lotus-worker 参数设置

myScheduler 社区版加入以下外部 lotus-worker 功能支持：

① 支持调度各类开源软件编译的 lotus-worker 优化版本；
② 支持调度外部 C2 的 lotus-worker 版本；
③ 支持调度外部阿里 FPGA卡的 lotus-worker P2优化版本；
④ 支持调度官方标准的 lotus-worker 等其它通用版本。 通过动态修改 $LOTUS_MINER_PATH 路径下面的 testOpenSource.json，可以控制被调度的 Worker 进行任意 AP/P1/P2/C2 数量配比和工作并行，以及 AP/P1/P2本地文件绑定不传输：

```shell
{
  "AddPieceMax": 1,
  "PreCommit1Max": 1,
  "PreCommit2Max": 1,
  "Commit2Max": 1,
  "ForceP1FromLocalAP": true,
  "ForceP2FromLocalP1": true,
  "ForceC2FromLocalP2": false,
  "AllowP2C2Parallel": true
}
```

## 自义定 pledge 任务实用工具

① `lotus-miner sectors mypledge mypledge` 一次性填充所有可以工作AP的 lotus-worker 的数量 AddPieceMax 总和，自动化任务可以定时调用 mypledge

② `lotus-miner sectors mypledgeonce mypledgeonce` 一次性为每个可以工作AP　的 lotus-worker 发送一个 AP，适合初始实施时，第一次使用 AP 模板复制功能。

## 调度过程日志分析与问题排查

由于调度的频繁度，在运行过程中，有大量的日志用于记录任务分配细节，可以通过下面的方式轻松查询相关成功分配和未分配的调度情况列表：

① more miner.log |grep -a trySchedMine > trySchedMine.txt， 这个里面记录了，所有成功调度的 has sucessfully scheduled 相关信息。
② more miner.log |grep -a "not scheduling" > not_scheduling.txt， 则是拒绝接收任务分配的worker 的日志。
③ 对于个别扇区的调度分配过程，则可以用以下方式查询和它相关的全部日志过程， more miner.log |grep -a "s-t0XXXX-YYYY"> s-t0XXXX-YYYY.txt ，格式是 "s-t0你的MinerID-某个扇区编号“。对于长时间不分配任务工作的 worker，一般是在 miner日志中可以看到 out of space 或者 didn't receive heartbeats for 的官方标准错误提醒。
④ 如果在 sealing workers 看到大量的 worker 闲置情况，但是挺多已经预分配的工作任务在 Preparing 里面，而长时间无法到在 Running。Preparing 通过了第一步的预备分配，在实际分配响应任务的时候不满足条件，就无法到达 Running 执行状态。尽量减少在动态参数配置文件中不必要的限制，限制条件越多，则越容易无法达到真正可运行状态。有时可以偿试进行一次 lotus-miner sectors mypledgeonce 触发新一轮的 AP 分配和调度。
下面的三种 miner 日志，存在任何一种，都表示这个 worker 不会接收任何工作任务：

```shell
cat miner.log |grep -a "didn't receive heartbeats for"|awk '{print $8}'|sort -rn|uniq -c

cat miner.log |grep -a "out of space "|awk '{print $8}'|sort -rn|uniq -c

cat miner.log |grep -a "trySchedMine skipping disabled worker "|awk '{print $9}'|sort -rn|uniq -c
```

对于这种 disabled worker 状态又不是断连，从调度程序的角度来看，disabled worker 和 out of space 一样的，不接收工作，更像是假死，而且 disabled 和 enabled 是互相动态变化的。


