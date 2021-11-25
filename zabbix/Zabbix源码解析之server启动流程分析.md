# Zabbix源码解析之server启动流程分析

## main函数

操作序列如下图所示：
[main函数](./img/main()序列图.puml)

``` c
typedef struct{
    zbx_task_t  task;
    int     flags;
    int     data;
}ZBX_TASK_EX;
```

该结构体用于zabbix启动时记录zabbix任务的一些信息。

1. zbx_task_t：是一个枚举，
    - 表示任务类型，在启动时指定为：ZBX_TASK_START。
    - 如果启动时指定--runtime-control（或-R）选项(执行管理能力)，则设置t.task = ZBX_TASK_RUNTIME_CONTROL。
2. flags：代表启动时的标志。
    - 如果启动时指定--foreground（或-f）选项（在前台运行zabbix守护进程），则t.flags会多添加一个ZBX_TASK_FLAG_FOREGROUND标志。
    - 当zabbix_server作为daemon启动时，flags会作为daemon的参数传入。
3. data：记录--runtime-control的选项信息，比如改变日志的级别或cache reload的值。比如：-R log_level_increase,-R config_cache_reload。

``` c
struct cfg_line{
    const char      *parameter;
    void            *variable;
    int             type;
    int             mandatory;
    zbx_uint64_t    min;
    zbx_uint64_t    max;
};
```

该结构体用于表示一个配置选项

1. parameter: zabbix_server.conf中的参数名
2. variable: 程序中参数的值
3. type: 参数类型，有TYPE_INT，TYPE_STRING等类型
4. mandatory: 是否是必须设定的参数
5. min: 限定参数范围，最小值
6. max: 限定参数范围，最大值

## daemon_start函数

操作序列如下图所示：

[](./img/daemon_start()序列图.puml)

## zbx_set_common_signal_handlers函数

该函数用来设置通用的信号的handlers。

``` c
struct sigaction    phan;
​
// 处理结束信号
// #define  SIGINT      2   /* Interrupt (ANSI).  */
// #define  SIGQUIT     3   /* Quit (POSIX).  */
// #define  SIGTERM     15  /* Termination (ANSI).  */
phan.sa_sigaction = terminate_signal_handler;
sigaction(SIGINT, &phan, NULL);
sigaction(SIGQUIT, &phan, NULL);
sigaction(SIGTERM, &phan, NULL);
​
// 处理致命信号
// #define  SIGILL      4   /* Illegal instruction (ANSI).  */
// #define  SIGFPE      8   /* Floating-point exception (ANSI).  */
// #define  SIGSEGV     11  /* Segmentation violation (ANSI).  */
// #define  SIGBUS      7   /* BUS error (4.2 BSD).  */
phan.sa_sigaction = fatal_signal_handler;
sigaction(SIGILL, &phan, NULL);
sigaction(SIGFPE, &phan, NULL);
sigaction(SIGSEGV, &phan, NULL);
sigaction(SIGBUS, &phan, NULL);
​
// 处理警告信号
// #define  SIGALRM     14  /* Alarm clock (POSIX).  */
phan.sa_sigaction = alarm_signal_handler;
sigaction(SIGALRM, &phan, NULL);
```

### sigaction函数

POSIX标准定义的信号处理接口是sigaction函数，其接口头文件及原型如下：

``` c
#include <signal.h>

struct sigaction
{
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t  sa_mask;
    int       sa_flags;
    void     (*sa_restorer)(void);
};

该结构体用来描述对信号的处理。

sa_handler: 函数指针，其含义是一个信号处理函数。只不过它只有一个int参数。

sa_sigaction：函数指针，另一个信号处理函数，它有三个参数，可以获得关于信号的更详细的信息。

sa_mask： 用来指定在信号处理函数执行期间需要被屏蔽的信号，特别是当某个信号被处理时，它自身会被自动放入进程的信号掩码，因此在信号处理函数执行期间这个信号不会再度发生。

sa_flags: 用于指定信号处理的行为，它可以是以下值的“按位或”组合。

    SA_RESTART：使被信号打断的系统调用自动重新发起。

    SA_NOCLDSTOP：使父进程在它的子进程暂停或继续运行时不会收到 SIGCHLD 信号。

    SA_NOCLDWAIT：使父进程在它的子进程退出时不会收到 SIGCHLD 信号，这时子进程如果退出也不会成为僵尸进程。

    SA_NODEFER：使对信号的屏蔽无效，即在信号处理函数执行期间仍能发出这个信号。

    SA_RESETHAND：信号处理之后重新设置为默认的处理方式。

    SA_SIGINFO：使用 sa_sigaction 成员而不是 sa_handler 作为信号处理函数。包含了SA_SIGINFO标志时，系统将使用sa_sigaction函数作为信号处理函数，否则使用sa_handler作为信号处理函数。在某些系统中，成员sa_handler与sa_sigaction被放在联合体中，因此使用时不要同时设置。

int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

signum：要操作的信号。

act：要设置的对信号的新处理方式。

oldact：原来对信号的处理方式
```

## set_daemon_signal_handlers函数

该函数用来设置daemon使用的信号的handlers。

``` c
struct sigaction    phan;
​
// 处理user signal：SIGUSR1
// #define  SIGUSR1     10  /* User-defined signal 1 (POSIX).  */
phan.sa_sigaction = user1_signal_handler;
sigaction(SIGUSR1, &phan, NULL);
​
// 处理 pipe signal：SIGPIPE
// #define  SIGPIPE     13  /* Broken pipe (POSIX).  */
phan.sa_sigaction = pipe_signal_handler;
sigaction(SIGPIPE, &phan, NULL);
```

## zbx_set_child_signal_handler函数

该函数用来设置子进程的信号的handlers。

``` c
struct sigaction    phan;
​
// 处理子进程的信号
// #define  SIGCHLD     17  /* Child status has changed (POSIX).  */
phan.sa_sigaction = child_signal_handler;
sigaction(SIGCHLD, &phan, NULL);
```

## MAIN_ZABBIX_ENTRY函数

操作序列如下图所示：

[](./img/MAIN_ZABBIX_ENTRY函数.puml)

### zabbix_open_log函数

该函数用于打开日志文件

```c
int zabbix_open_log(int type, int level, const char *filename, char **error)
{
    log_type = type;
    zbx_log_level = level;
​
    if (LOG_TYPE_SYSTEM == type)
    {
        // 打开到system logger的连接
        openlog(syslog_app_name, LOG_PID, LOG_DAEMON);
    }
    else if (LOG_TYPE_FILE == type)
    {
        FILE    *log_file = NULL;
        // 打开文件
        if (NULL == (log_file = fopen(filename, "a+")))
        {
            *error = zbx_dsprintf(*error, "unable to open log file [%s]: %s", filename, zbx_strerror(errno));
            return FAIL;
        }
​
        strscpy(log_filename, filename);
        zbx_fclose(log_file);
    }
    else if (LOG_TYPE_CONSOLE == type)
    {
        // 使用控制台，将stderr重定向到 stdout
        fflush(stderr);
        if (-1 == dup2(STDOUT_FILENO, STDERR_FILENO))
            zbx_error("cannot redirect stderr to stdout: %s", zbx_strerror(errno));
    }
    else if (LOG_TYPE_UNDEFINED != type)
    {
        *error = zbx_strdup(*error, "unknown log type");
        return FAIL;
    }
​
    return SUCCEED;
}
```

### zbx_load_modules函数

加载可加载模块（动态链接库的形式，example中的dummy的例子）。

### DBconnect函数

连接到数据库。

### DCsync_configuration函数

从DB中同步配置数据

### zbx_check_postinit_tasks函数

处理初始化以后的任务

```c
int zbx_check_postinit_tasks(char **error)
{
    // select taskid from task where type=%d and status=%d
    result = DBselect("select taskid from task where type=%d and status=%d", ZBX_TM_TASK_UPDATE_EVENTNAMES,ZBX_TM_STATUS_NEW);
​
    // DBfetch：取结果
    if (NULL != (row = DBfetch(result)))
    {
        // 开始执行事务（transaction）
        DBbegin();
​
        // update event names in events and problem tables
        /* "select triggerid,description,expression,priority,comments,url,recovery_expression,"
                "recovery_mode,value"
            " from triggers"
            " order by triggerid"
         */
        if (SUCCEED == (ret = update_event_names()))
        {
            // 执行一个非select操作
            // delete from task where taskid=%s
            DBexecute("delete from task where taskid=%s", row[0]);
            // 提交事务
            DBcommit();
        }
        else
            // 执行失败，回滚事务
            DBrollback();
    }
​
    // 释放内存
    DBfree_result(result);
​
    if (SUCCEED != ret)
        *error = zbx_strdup(*error, "cannot update event names");
​
    return ret;
}
```

### DBclose函数

关闭数据库连接。

### 创建子进程

下面分析创建子进程的过程。

```c
typedef struct
{
    int     server_num;
    int     process_num;
    unsigned char   process_type;
    void        *args;
}zbx_thread_args_t;
该结构体保存进程参数:
server_num：进程编号
process_num：此种类型的进程的序号
process_type：创建的子进程类型，如：ZBX_PROCESS_TYPE_HOUSEKEEPER，ZBX_PROCESS_TYPE_POLLER
args：进程的附加参数信息
```

ZBX_THREAD_ENTRY宏
定义宏的目的，是为了跨平台，统一使用一个名称。

```c
// ZBX_THREAD_ENTRY_POINTER是一个函数指针，它所指的函数类型与ZBX_THREAD_ENTRY所代表的函数的类型一致
// ZBX_THREAD_ENTRY是一个宏，指代一个函数，该函数的返回值为unsigned，参数类型为void *
#if defined(_WINDOWS)
    #define ZBX_THREAD_ENTRY_POINTER(pointer_name) \
        unsigned (__stdcall *pointer_name)(void *)
    #define ZBX_THREAD_ENTRY(entry_name, arg_name)  \
        unsigned __stdcall entry_name(void *arg_name)
#else   /* not _WINDOWS */
    #define ZBX_THREAD_ENTRY_POINTER(pointer_name) \
        unsigned (* pointer_name)(void *)
    #define ZBX_THREAD_ENTRY(entry_name, arg_name)  \
        unsigned entry_name(void *arg_name)
```

ZBX_THREAD_ENTRY定义了函数，在创建新进程时，会调用这些函数

```c
ZBX_THREAD_ENTRY(dbconfig_thread, args)
...
ZBX_THREAD_ENTRY(poller_thread, args)
...
ZBX_THREAD_ENTRY(trapper_thread, args)
...
```

创建各个子进程

```c
int MAIN_ZABBIX_ENTRY(int flags)
{
    ...
    // 总进程数。这里使用"thread"，实际上下面的操作是fork进程
    threads_num = CONFIG_CONFSYNCER_FORKS + CONFIG_POLLER_FORKS
            + CONFIG_UNREACHABLE_POLLER_FORKS + CONFIG_TRAPPER_FORKS + CONFIG_PINGER_FORKS
            + CONFIG_ALERTER_FORKS + CONFIG_HOUSEKEEPER_FORKS + CONFIG_TIMER_FORKS
            + CONFIG_HTTPPOLLER_FORKS + CONFIG_DISCOVERER_FORKS + CONFIG_HISTSYNCER_FORKS
            + CONFIG_ESCALATOR_FORKS + CONFIG_IPMIPOLLER_FORKS + CONFIG_JAVAPOLLER_FORKS
            + CONFIG_SNMPTRAPPER_FORKS + CONFIG_PROXYPOLLER_FORKS + CONFIG_SELFMON_FORKS
            + CONFIG_VMWARE_FORKS + CONFIG_TASKMANAGER_FORKS + CONFIG_IPMIMANAGER_FORKS
            + CONFIG_ALERTMANAGER_FORKS + CONFIG_PREPROCMAN_FORKS + CONFIG_PREPROCESSOR_FORKS;
    // 分配内存
    threads = (pid_t *)zbx_calloc(threads, threads_num, sizeof(pid_t));
​
    // 到此为止，主进程[main process]已启动完成了
    zabbix_log(LOG_LEVEL_INFORMATION, "server #0 started [main process]");
​
    // 下面是启动每一个子进程
    for (i = 0; i < threads_num; i++)
    {
        zbx_thread_args_t   thread_args;
        unsigned char       poller_type;
​
        // 根据threads_num来循环，获取本次循环中进程的类型和进程编号
        if (FAIL == get_process_info_by_thread(i + 1, &thread_args.process_type, &thread_args.process_num))
        {
            THIS_SHOULD_NEVER_HAPPEN;
            exit(EXIT_FAILURE);
        }
​
        thread_args.server_num = i + 1;
        thread_args.args = NULL;
​
        switch (thread_args.process_type)
        {
            case ZBX_PROCESS_TYPE_CONFSYNCER:
                zbx_thread_start(dbconfig_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_POLLER:
                poller_type = ZBX_POLLER_TYPE_NORMAL;
                thread_args.args = &poller_type;
                // thread_args作为进程参数传入poller_thread函数，并被调用
                zbx_thread_start(poller_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_UNREACHABLE:
                poller_type = ZBX_POLLER_TYPE_UNREACHABLE;
                thread_args.args = &poller_type;
                zbx_thread_start(poller_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_TRAPPER:
                thread_args.args = &listen_sock;
                zbx_thread_start(trapper_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_PINGER:
                zbx_thread_start(pinger_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_ALERTER:
                zbx_thread_start(alerter_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_HOUSEKEEPER:
                zbx_thread_start(housekeeper_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_TIMER:
                zbx_thread_start(timer_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_HTTPPOLLER:
                zbx_thread_start(httppoller_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_DISCOVERER:
                zbx_thread_start(discoverer_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_HISTSYNCER:
                zbx_thread_start(dbsyncer_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_ESCALATOR:
                zbx_thread_start(escalator_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_JAVAPOLLER:
                poller_type = ZBX_POLLER_TYPE_JAVA;
                thread_args.args = &poller_type;
                zbx_thread_start(poller_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_SNMPTRAPPER:
                zbx_thread_start(snmptrapper_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_PROXYPOLLER:
                zbx_thread_start(proxypoller_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_SELFMON:
                zbx_thread_start(selfmon_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_VMWARE:
                zbx_thread_start(vmware_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_TASKMANAGER:
                zbx_thread_start(taskmanager_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_PREPROCMAN:
                zbx_thread_start(preprocessing_manager_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_PREPROCESSOR:
                zbx_thread_start(preprocessing_worker_thread, &thread_args, &threads[i]);
                break;
#ifdef HAVE_OPENIPMI
            case ZBX_PROCESS_TYPE_IPMIMANAGER:
                zbx_thread_start(ipmi_manager_thread, &thread_args, &threads[i]);
                break;
            case ZBX_PROCESS_TYPE_IPMIPOLLER:
                zbx_thread_start(ipmi_poller_thread, &thread_args, &threads[i]);
                break;
#endif
            case ZBX_PROCESS_TYPE_ALERTMANAGER:
                zbx_thread_start(alert_manager_thread, &thread_args, &threads[i]);
                break;
        }
    }
​
    if (SUCCEED == zbx_is_export_enabled())
    {
        zbx_history_export_init("main-process", 0);
        zbx_problems_export_init("main-process", 0);
    }
​
    // 等待子进程结束，当子进程正常结束时，返回子进程的pid，这里会继续执行循环
    // 当发生错误时，会返回(pid_t) -1.这时会结束循环，从而主进程也就结束了。
    while (-1 == wait(&i))  /* wait for any child to exit */
    {
        if (EINTR != errno)
        {
            zabbix_log(LOG_LEVEL_ERR, "failed to wait on child processes: %s", zbx_strerror(errno));
            break;
        }
    }
​
    /* all exiting child processes should be caught by signal handlers */
    THIS_SHOULD_NEVER_HAPPEN;
​
    // 进程退出时的操作
    zbx_on_exit();
​
    return SUCCEED;
}

```

zbx_thread_start函数

zbx_thread_start是一个封装函数，在其中会调用fork()函数，创建子进程。

```c
/*
 * Parameters: handler     - [IN] new thread starts execution from this       *
 *                                handler function                            *
 *             thread_args - [IN] arguments for thread function               *
 *             thread      - [OUT] handle to a newly created thread           *
 */
void    zbx_thread_start(ZBX_THREAD_ENTRY_POINTER(handler), zbx_thread_args_t *thread_args, ZBX_THREAD_HANDLE *thread)
{
    // call fork()
    zbx_child_fork(thread);
​
    // 返回0，是子进程
    if (0 == *thread)   /* child process */
    {
        // 调用进程函数
        (*handler)(thread_args);
    }
    // 返回-1，fork失败
    else if (-1 == *thread) 
    {
        zbx_error("failed to fork: %s", zbx_strerror(errno));
        *thread = (ZBX_THREAD_HANDLE)ZBX_THREAD_ERROR;
    }
}
​
void    zbx_child_fork(pid_t *pid)
{
    *pid = zbx_fork();
}
​
int zbx_fork(void)
{
    return fork();
}
```

zbx_on_exit函数

当子进程发生错误，导致主进程退出时，执行zbx_on_exit函数。

```c
void    zbx_on_exit(void)
{
    // 如果还有数据库事务在执行，则回滚该事务
    if (SUCCEED == DBtxn_ongoing())
        DBrollback();
​
    if (NULL != threads)
    {
        // 会主动执行kill SIGTERM，并等待子进程结束
        zbx_threads_wait(threads, threads_num); /* wait for all child processes to exit */
        // 释放内存
        zbx_free(threads);
    }
​
    //释放各指标参数的内存
    free_metrics();
    zbx_ipc_service_free_env();
​
    // 释放DB cache内存
    DBconnect(ZBX_DB_CONNECT_EXIT);
​
    free_database_cache();
​
    DBclose();
​
    // 释放配置cache内存
    free_configuration_cache();
​
    /* free history value cache */
    zbx_vc_destroy();
​
    zbx_destroy_itservices_lock();
​
    /* free vmware support */
    if (0 != CONFIG_VMWARE_FORKS)
        zbx_vmware_destroy();
​
    free_selfmon_collector();
​
    zbx_uninitialize_events();
​
    // 卸载可加载模块
    zbx_unload_modules();
​
    // 关闭log文件
    zabbix_close_log();
​
    exit(EXIT_SUCCESS);
}
```

## 记录日志

### 配置文件中的配置

zabbix_server的日志的位置，级别等配置，在zabbix_server.conf中进行配置。

```shell
### Option: LogType
#   Specifies where log messages are written to:
#       system  - syslog
#       file    - file specified with LogFile parameter
#       console - standard output
#
# Mandatory: no
# Default:
# LogType=file
​
### Option: LogFile
#   Log file name for LogType 'file' parameter.
#
# Mandatory: yes, if LogType is set to file, otherwise no
# Default:
# LogFile=
​
LogFile=/tmp/zabbix_server.log
​
### Option: DebugLevel
#   Specifies debug level:
#   0 - basic information about starting and stopping of Zabbix processes
#   1 - critical information
#   2 - error information
#   3 - warnings
#   4 - for debugging (produces lots of information)
#   5 - extended debugging (produces even more information)
#
# Mandatory: no
# Range: 0-5
# Default:
# DebugLevel=3
```

### 实现

1. zabbix日志的实现，头文件在：include/log.h，实现文件在src/libs/zbxlog/log.c中。
2. 配置文件中的LogType指定日志文件的类型，一般记录到文本文件中，因此指定为file。对应的源码中的设置为：

```c
#define LOG_TYPE_UNDEFINED  0
#define LOG_TYPE_SYSTEM     1
#define LOG_TYPE_FILE       2
#define LOG_TYPE_CONSOLE    3
​
#define ZBX_OPTION_LOGTYPE_SYSTEM   "system"
#define ZBX_OPTION_LOGTYPE_FILE     "file"
#define ZBX_OPTION_LOGTYPE_CONSOLE  "console"
```

3. 当LogType指定为file时，需要LogFile指定记录的日志文件的名称和位置。
4. DebugLevel指定日志级别。默认为warnings。对应源码中的设置为：

```c
#define LOG_LEVEL_EMPTY     0   /* printing nothing (if not LOG_LEVEL_INFORMATION set) */
#define LOG_LEVEL_CRIT      1
#define LOG_LEVEL_ERR       2
#define LOG_LEVEL_WARNING   3
#define LOG_LEVEL_DEBUG     4
#define LOG_LEVEL_TRACE     5
#define LOG_LEVEL_INFORMATION   127 /* printing in any case no matter what level set */
```

5. 无论日志级别设置为什么，使用LOG_LEVEL_INFORMATION级别来设置日志，总是能打印出来。所以我们在调查源码时，可以以LOG_LEVEL_INFORMATION级别来记录调试日志。
