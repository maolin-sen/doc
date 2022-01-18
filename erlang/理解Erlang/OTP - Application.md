# 理解Erlang/OTP - Application

```erlang
  1>application:start(log4erl).
```

我们就从这一行命令开始说起吧,回车之后可以把log4erl应用程序启动起来.Erlang/OTP中的能完成特定功能集合的组件被称为application. ,application是Erlang代码和功能组织的形式之一([Erlang 0015]Erlang OTP设计原则).application的设计目的是通过运行一个或者多个进程来完成一定功能.为了能够管理这些进程的生命周期,需要通过supervisor进行管理.

application:start(log4erl).实际上是执行了application:start(log4erl,temporary).第二个参数表示应用程序的启动类型(Start_Type).启动类型还可以是permanent  transient,三种启动类型的区别是:

* 如果Start_Type==permanent 应用程序终止之后所有其它的应用程序和运行时系统都会死掉;
* 如果Start_Type==transient 应用程序终止的原因是normal,这个消息会报出来但是其它应用程序不会重启,如果应用程序终止的原因不是normal,其他应用程序和运行时也会跟着死掉;
* 如果Start_Type==temporary 应用程序死掉会报错误出来但是其它应用程序不受影响.注意实践过程中很少使用transient参数,因为进程树崩掉的时候,进程正常退出原因是shutdown不是normal,这个我之前的文章提到过:[Erlang 0017]Erlang/OTP基础模块 proc_lib 无论使用哪种类型启动,直接调用stop方法关闭application是不会影响到其它应用程序的.

application的配置文件(Application Resource File)基本上确定了application的格局.如果启动失败了就要检查一下配置文件了,配置文件命名要求必须与application命名一致,即必须要有一个log4erl.app文件:

```erlang
{application, log4erl,
[{description, "Logger for erlang in the spirit of Log4J"},
{vsn, "0.9.0"},
{modules, [console_appender,    
           dummy_appender,     
           email_msg,     
           error_logger_log4erl_h,     
           % ................省略部分模块     
           xml_appender]},
{registered,[log4erl]}, %%这个应用程序将使用的注册名
{applications, [kernel,stdlib]}, %%注意这个配置节是指定当前应用程序依赖哪些应用程序,类似Windows服务的依赖关系
{mod, {log4erl,[]}},
{env,[{key,value},{k2,v2}]}, %%env配置节,里面以key-value的形式组织配置数据.可以用application:get_env/2读取.
{start_phases, []}]}.
```

## application启动过程

Erlang运行时启动时,application controller进程会随着Kernel应用程序启动,应用程序相关的操作都由它来协调完成,它通过application模块暴露出来的接口来实现应用程序的加载,卸载,启动,停止等等.

应用程序启动之前首先会进行加载load,如果没有加载,application controller会首先执行load/1.加载完成,application controller会检查application配置文件中的applications配置节中所列出的应用程序都已经在运行.如果有依赖项还没有启动,就抛出{error,{not_started,App}}的错误.

完成依赖项检查之后application controller会为application创建application master.application master会成为该application中所有进程的Group Leader.通过配置文件的mod配置节,application master知道要调用回调函数log4erl:start/2.

application behavior有两个回调函数需要实现 start/2 stop/1分别来指定应用程序如何启动,如何停止.启动application的时候调用start方法通过启动顶层的supervisor来创建进程树.log4erl实现了application的behavior,它的start方法也是中规中矩的启动了顶层superior:

```erlang
start(_Type, []) ->
    log4erl_sup:start_link(?DEFAULT_LOGGER). %%这里?DEFAULT_LOGGER宏的值为default_logger
```

start对返回值是有规格要求的:start(StartType, StartArgs) -> {ok, Pid} | {ok, Pid, State}

当返回结果不一致的时候应用程序自然启动失败,初学的时候最常见的错误就是加一下输出看看是不是启动了,结果就悲剧了.比如下面的代码,start函数的返回值是最后一个表达式io:format的值,而这个结果是ok,不符合application对start回调函数结果的要求.

```erlang
start(Startype,Arg) ->
    demo_sup:start_link(),
    io:format("demo app start").
```

## application停止过程

要停止application,application master首先会调用Module:prep_stop/1(如果存在),然后告知顶层的supervisor关闭(shutdown),shutdown的过程:整个监控树的所有进程和包含的应用程序按照启动的逆序终止.shutdown完成之后,application master调用Module:stop/1.最后application master自己终止掉.application停止了,但是依然处于已加载的状态(loaded).
