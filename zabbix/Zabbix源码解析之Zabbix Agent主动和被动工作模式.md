# Zabbix源码解析之Zabbix Agent主动和被动工作模式 

## 被动模式（Passive Check）

1. 主动和被动都是相对于参照物的，这里我们所说的主动和被动，都是相对于“agent”来说的。
2. agent工作在被动模式下，那么Zabbix服务器会定时对agent进行轮询，以监控项关键字为请求参数，agent收到请求后查询这个监控项关键字对应的监控数据，最后返回给服务器。
3. 被动模式监控项的类型，在前端设置界面中需要选择“Zabbix agent”。

### 被动模式相关配置参数

使用被动式agent，需要在zabbix_agentd.conf配置文件中指定如下几个配置参数：

1. Server: 用于指定Zabbix服务器的主机名或IP地址。这个参数是一个以逗号分隔的IP地址（或主机名）列表，Zabbix agent只会接受列表中指定主机的连接。

2. ListenPort: 用于指定Zabbix agent监听的端口号，默认值为10050，取值范围为1024-32767。

3. ListenIP: 用于指定Zabbix agent监听的IP地址，这个参数适用于受监控主机具有多块网卡或多个IP地址的情况。这个参数是一个以逗号分隔的IP地址列表。

4. StartAgents: 用于指定为了处理被动式监控项而预先建立的zabbix_agentd的进程实例的数量，默认值为3。如果将这个参数设置为0，那么便会禁用agent的被动模式，agent也就不会监听任何TCP端口了。

### zabbix_get工具

zabbix_get工具可用于模拟被动式agent和Zabbix服务器之间的网络通信流程。zabbix_get应用于server端，向agent请求数据。

```shell
zabbix_get [-s <host name or IP> ] [-p <port number> ] [-I <IP address> ] [-k <item key> ]

-s/-host <host name or IP>: 指定受监控服务器agent的主机名或IP地址。

-p/--port <port number>: 指定agent的端口号。默认值为10050。

-I/--source-address <IP address>: 指定源IP地址。

-k/--key <item key>: 指定监控项的关键字，以获取相应的监控数据。
```

[](./img/被动模式监控流程图.puml)

listener_thread

```c

ZBX_THREAD_ENTRY(listener_thread, args)
{
 int  ret;
 zbx_socket_t s;
 
 process_type = ((zbx_thread_args_t *)args)->process_type;
 server_num = ((zbx_thread_args_t *)args)->server_num;
 process_num = ((zbx_thread_args_t *)args)->process_num;
 
 zabbix_log(LOG_LEVEL_INFORMATION, "%s #%d started [%s #%d]", get_program_type_string(program_type),
   server_num, get_process_type_string(process_type), process_num);
 
 memcpy(&s, (zbx_socket_t *)((zbx_thread_args_t *)args)->args, sizeof(zbx_socket_t));
 
 zbx_free(args);
 
 // 无限循环
 while (ZBX_IS_RUNNING())
 {
  zbx_setproctitle("listener #%d [waiting for connection]", process_num);
  // zbx_tcp_accept: select()，accept()，recv()
  ret = zbx_tcp_accept(&s, configured_tls_accept_modes);
  zbx_update_env(zbx_time());
 
  if (SUCCEED == ret)
  {
   zbx_setproctitle("listener #%d [processing request]", process_num);
 
   if ('\0' != *CONFIG_HOSTS_ALLOWED &&
     SUCCEED == (ret = zbx_tcp_check_allowed_peers(&s, CONFIG_HOSTS_ALLOWED)))
   {
    {
     process_listener(&s);  // 被动模式主要处理逻辑
    }
   }
 
   // 断开连接
   zbx_tcp_unaccept(&s);
  }
 
  if (SUCCEED == ret || EINTR == zbx_socket_last_error())
   continue;
 
  // 休眠
  if (ZBX_IS_RUNNING())
   zbx_sleep(1);
 }
}
 
 
static void process_listener(zbx_socket_t *s)
{
 AGENT_RESULT result;
 char  **value = NULL;
 int  ret;
 
 // zbx_tcp_recv_to： read()，读取server端发送的监控项
 if (SUCCEED == (ret = zbx_tcp_recv_to(s, CONFIG_TIMEOUT)))
 {
  zbx_rtrim(s->buffer, "\r\n");
 
  // 日志示例： Requested [system.cpu.load[percpu,avg15]]
  zabbix_log(LOG_LEVEL_DEBUG, "Requested [%s]", s->buffer);
 
  init_result(&result);
 
  // process：调用监控项函数（command->function(&request, result)），查询监控数据
  if (SUCCEED == process(s->buffer, PROCESS_WITH_ALIAS, &result))
  { // 成功
   if (NULL != (value = GET_TEXT_RESULT(&result)))
   {
    // 日志示例： Sending back [0.640000]
    zabbix_log(LOG_LEVEL_DEBUG, "Sending back [%s]", *value);
    // zbx_tcp_send_to: write()，向server端发送监控项返回结果
    ret = zbx_tcp_send_to(s, *value, CONFIG_TIMEOUT);
   }
  }
  else
  { // 失败
   value = GET_MSG_RESULT(&result);
 
   if (NULL != value)
   {
    static char *buffer = NULL;
    static size_t buffer_alloc = 256;
    size_t  buffer_offset = 0;
 
    zabbix_log(LOG_LEVEL_DEBUG, "Sending back [" ZBX_NOTSUPPORTED ": %s]", *value);
 
    if (NULL == buffer)
     buffer = (char *)zbx_malloc(buffer, buffer_alloc);
 
    zbx_strncpy_alloc(&buffer, &buffer_alloc, &buffer_offset,
      ZBX_NOTSUPPORTED, ZBX_CONST_STRLEN(ZBX_NOTSUPPORTED));
    buffer_offset++;
    zbx_strcpy_alloc(&buffer, &buffer_alloc, &buffer_offset, *value);
 
    ret = zbx_tcp_send_bytes_to(s, buffer, buffer_offset, CONFIG_TIMEOUT);
   }
   else
   {
    zabbix_log(LOG_LEVEL_DEBUG, "Sending back [" ZBX_NOTSUPPORTED "]");
 
    ret = zbx_tcp_send_to(s, ZBX_NOTSUPPORTED, CONFIG_TIMEOUT);
   }
  }
 
  free_result(&result);
 }
 
 if (FAIL == ret)
  zabbix_log(LOG_LEVEL_DEBUG, "Process listener error: %s", zbx_socket_strerror());
}

```

## 主动模式（Active Check）

1. 如果agent工作在主动模式下，那么agent会定时向Zabbix服务器请求刷新主动式监控项列表，然后会主动地定时向Zabbix服务器发送主动式监控项列表对应的监控数据。
2. 主动模式监控项的类型，在前端设置界面中需要选择“Zabbix agent (active)”。
3. 在前端设置界面中的Update interval：监控数据的更新时间间隔，以秒为单位。当agent向Zabbix服务器请求主动式监控项列表时，也会取得这个参数。agent会根据这个时间间隔，主动向Zabbix服务器发送监控数据。我们会在源码中看到这个参数的应用。

### 主动模式相关配置参数

1. ServerActive: 用于指定能够接收监控数据的Zabbix服务器。这个参数是一个以逗号分隔的的“IP:端口号”（或“主机名:端口号”）列表。如果不指定端口号，则使用默认端口号10051。如果不指定这个参数，则禁用agent的主动模式。
2. Hostname: 用于指定受监控主机的主机名，它是唯一的、区分大小写的。这个主机名必须和Zabbix前端页面上注册的受监控主机的主机名完全相同，否则agent将不能成功刷新自己的主动式监控项列表。
3. RefreshActiveChecks: 用于指定agent的主动式监控项列表的刷新时间间隔，以秒为单位。
4. BufferSend: 用于指定agent的缓冲发送时间，以秒为单位。监控数据在缓冲区中的保存时间不会超过这个参数指定的时间。
5. BufferSize: 用于指定缓冲区的大小，也就是监控数据的最大数量。如果缓冲区满了，那么agent便会向Zabbix服务器或代理发送所有已经收集的监控数据。

### zabbix_sender工具

zabbix_sender的实用工具可用于模拟主动式agent和Zabbix服务器之间的网络通信流程。zabbix_sender应用于agent端，向server发送数据。

```sh
zabbix_sender [-hpzvIV] {-kso | [-T] -i <inputfile> } [-c <config-file> ]

-c/--config <config-file>: 使用config-file指定的配置文件。zabbix_sender会从agent的配置文件中读取服务端的详细信息。应当指定配置文件的绝对路径。在默认情况下，zabbix_sender不会读取任何配置文件。zabbix_sender只会用到配置文件中的Hostname、ServerActive(只会使用其第一个条目)和SourceIP参数。

-z/--zabbix-server <server>: Zabbix服务器的主机名或IP地址。如果某台服务器是由Proxy监控的，那么就应当指定Proxy的主机名或IP地址。

-p/--port <port>: 指定在Zabbix服务器上运行的服务端捕捉器（trapper）的端口号。默认值为10051。

-s/--host <host>: 指定在Zabbix前端页面上注册的主机名。主机的IP地址和DNS名称将不会起作用。

-I/--source-address <IP>: 指定来源IP地址。

-k/--key <key>: 指定监控项的关键字。

-o/--value <value>: 指定监控数据。

-i/--input-file <inputfile>: 从inputfile指定的文件中加载监控数据。

-T/--with-timestamps: 这个选项可以和--input-file选项配合使用。

-r/--real-time: 当接收到监控数据时，便一个接一个地发送至Zabbix服务器。当从标准输入中读取监控数据时，可以使用这个选项。
```

[](img/主动模式监控流程图.puml)

active_checks_thread

``` c
ZBX_THREAD_ENTRY(active_checks_thread, args)
{
 ZBX_THREAD_ACTIVECHK_ARGS activechk_args;
 
 time_t nextcheck = 0, nextrefresh = 0, nextsend = 0, now, delta, lastcheck = 0;
 double time_now;
 
 assert(args);
 assert(((zbx_thread_args_t *)args)->args);
 
 process_type = ((zbx_thread_args_t *)args)->process_type;
 server_num = ((zbx_thread_args_t *)args)->server_num;
 process_num = ((zbx_thread_args_t *)args)->process_num;
 
 zabbix_log(LOG_LEVEL_INFORMATION, "%s #%d started [%s #%d]", get_program_type_string(program_type),
   server_num, get_process_type_string(process_type), process_num);
 
 activechk_args.host = zbx_strdup(NULL, ((ZBX_THREAD_ACTIVECHK_ARGS *)((zbx_thread_args_t *)args)->args)->host);
 activechk_args.port = ((ZBX_THREAD_ACTIVECHK_ARGS *)((zbx_thread_args_t *)args)->args)->port;
 
 zbx_free(args);
 
 session_token = zbx_create_token(0);
 
 // 初始化主动监控项
 init_active_metrics();
 
 // 无限循环
 while (ZBX_IS_RUNNING())
 {
  time_now = zbx_time();
  zbx_update_env(time_now);
  now = (int)time_now;
 
  // 发送缓冲区中的监控数据，调用send_buffer函数
  if (now >= nextsend)
  {
   send_buffer(activechk_args.host, activechk_args.port);
   nextsend = time(NULL) + 1;
  }
 
  // 刷新主动监控项列表，默认120s刷一次。
  if (now >= nextrefresh)
  {
   zbx_setproctitle("active checks #%d [getting list of active checks]", process_num);
   // 如果刷新失败，下次在60s后再刷新一次
   if (FAIL == refresh_active_checks(activechk_args.host, activechk_args.port))
   {
    nextrefresh = time(NULL) + 60;
   }
   else
   {
    nextrefresh = time(NULL) + CONFIG_REFRESH_ACTIVE_CHECKS;
   }
  }
 
  // 主动发送监控数据
  if (now >= nextcheck && CONFIG_BUFFER_SIZE / 2 > buffer.pcount)
  {
   zbx_setproctitle("active checks #%d [processing active checks]", process_num);
 
   process_active_checks(activechk_args.host, activechk_args.port);
 
   if (CONFIG_BUFFER_SIZE / 2 <= buffer.pcount) /* failed to complete processing active checks */
    continue;
 
   // 找到所有主动监控指标中那个所设置的时间间隔最小值，更新nextcheck时间，如果得不到，则默认设为60s
   nextcheck = get_min_nextcheck();
   if (FAIL == nextcheck)
    nextcheck = time(NULL) + 60;
  }
  else
  {
   // 当前时间 - lastcheck < 0，这会在系统时间被向前调整的情况下发生，此时zabbix要相应更新nextcheck值
   if (0 > (delta = now - lastcheck))
   {
    zabbix_log(LOG_LEVEL_WARNING, "the system time has been pushed back,"
      " adjusting active check schedule");
    // 系统时间被调整，更新主动监控任务的时间安排，每个指标的nextcheck + delta
    update_schedule((int)delta);
    nextcheck += delta;
    nextsend += delta;
    nextrefresh += delta;
   }
 
   zbx_setproctitle("active checks #%d [idle 1 sec]", process_num);
   // 在不发送主动监控指标的时候，休眠1s
   zbx_sleep(1);
  }
 
  lastcheck = now;
 }
 
 zbx_free(session_token);
}
 
static void process_active_checks(char *server, unsigned short port)
{
 const char *__function_name = "process_active_checks";
 char  *error = NULL;
 int  i, now, ret;
 
 zabbix_log(LOG_LEVEL_DEBUG, "In %s() server:'%s' port:%hu", __function_name, server, port);
 
 now = (int)time(NULL);
 
 // 遍历active_metrics，依次发送
 for (i = 0; i < active_metrics.values_num; i++)
 {
  zbx_uint64_t  lastlogsize_last, lastlogsize_sent;
  int   mtime_last, mtime_sent;
  ZBX_ACTIVE_METRIC *metric;
 
  metric = (ZBX_ACTIVE_METRIC *)active_metrics.values[i];
 
  // 如果指标设置的nextcheck时间大于当前时间，则该指标这次不发送，继续下面的指标
  if (metric->nextcheck > now)
   continue;
 
  ...
  // 发送指标，调用send_buffer函数
  send_buffer(server, port);
  // 更新下次发送时间
  metric->nextcheck = (int)time(NULL) + metric->refresh;
 }
}
```

1. 发送指标的函数是send_buffer，将buffer中的数据发送到server端，buffer中的数据会序列化到json中，然后使用zbx_tcp_send函数发送出去。

2. active_checks_thread中涉及到3个时间参数，他们分别为：

    - RefreshActiveChecks: 用于指定agent的主动式监控项列表的刷新时间间隔，以秒为单位。

        - 当“当前时间 - 上次刷新时间 >= 刷新时间间隔(RefreshActiveChecks)”时，刷新主动监控项。

    - Update interval: 监控数据的更新时间间隔，以秒为单位。agent会根据这个时间间隔，主动向Zabbix服务器发送监控数据。

        - 当“当前时间 - 上次发送时间 >= 发送时间间隔(Update interval)”时，发送主动监控指标。

    - BufferSend: 用于指定agent的缓冲发送时间，以秒为单位。监控数据在缓冲区中的保存时间不会超过这个参数指定的时间。
