# Zabbix源码解析之Zabbix Agent启动流程

## main函数

与zabbix_server中main函数的流程类似。

daemon_start函数
与zabbix_server中main函数的流程一致。只不过在agent中，daemon_start函数被START_MAIN_ZABBIX_ENTRY宏封装了一下。

MAIN_ZABBIX_ENTRY函数
与zabbix_server中MAIN_ZABBIX_ENTRY函数的流程类似，只是其中有些步骤采取的操作不同。操作序列如下图所示：

### 创建子进程

```c
int MAIN_ZABBIX_ENTRY(int flags)
{
 ...
 // 总进程数。
 /* allocate memory for a collector, all listeners and active checks */
 threads_num = CONFIG_COLLECTOR_FORKS + CONFIG_PASSIVE_FORKS + CONFIG_ACTIVE_FORKS;
 // 分配内存
 threads = (ZBX_THREAD_HANDLE *)zbx_calloc(threads, threads_num, sizeof(ZBX_THREAD_HANDLE));
 
 // 启动每一个子进程
 for (i = 0; i < threads_num; i++)
 {
  zbx_thread_args_t *thread_args;
 
  // 根据threads_num来循环，获取本次循环中进程的类型和进程编号
  thread_args = (zbx_thread_args_t *)zbx_malloc(NULL, sizeof(zbx_thread_args_t));
 
  if (FAIL == get_process_info_by_thread(i + 1, &thread_args->process_type, &thread_args->process_num))
  {
   THIS_SHOULD_NEVER_HAPPEN;
   exit(EXIT_FAILURE);
  }
 
  thread_args->server_num = i + 1;
  thread_args->args = NULL;
 
  switch (thread_args->process_type)
  {
   case ZBX_PROCESS_TYPE_COLLECTOR:
    zbx_thread_start(collector_thread, thread_args, &threads[i]);
    break;
   case ZBX_PROCESS_TYPE_LISTENER:
    thread_args->args = &listen_sock;
    zbx_thread_start(listener_thread, thread_args, &threads[i]);
    break;
   case ZBX_PROCESS_TYPE_ACTIVE_CHECKS:
    thread_args->args = &CONFIG_ACTIVE_ARGS[j++];
    zbx_thread_start(active_checks_thread, thread_args, &threads[i]);
    break;
  }
  zbx_free(thread_args);
 }
 
 while (-1 == wait(&i)) /* wait for any child to exit */
 {
  if (EINTR != errno)
  {
   zabbix_log(LOG_LEVEL_ERR, "failed to wait on child processes: %s", zbx_strerror(errno));
   break;
  }
 }
 
 /* all exiting child processes should be caught by signal handlers */
 THIS_SHOULD_NEVER_HAPPEN;
 
 // 进程退出时的操作
 zbx_on_exit();
 
 return SUCCEED;
}
```
zabbix agent启动的时候，默认会启动三个进程。

collector_thread: 用来收集本机的基本监控信息（收集cpu状态存入结构体ZBX_CPUS_STAT_DATA中，收集磁盘的状态存入结构体ZBX_DISKDEVICES_DATA中）。

listener_thread: 用来实现zabbix server连接过来的被动监控，它负责监听10050端口，然后等待server端的请求并进行响应。

active_checks_thread: 负责主动发送收集到的信息给zabbix_server。
