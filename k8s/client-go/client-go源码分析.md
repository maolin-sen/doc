# client-go 源码分析

## client-go的SharedInformerFactory

```go
    // client-go/informers/factory.go
    // SharedInformerFactory是个interfaces，所以肯定有具体的实现类 

    type SharedInformerFactory interface {
        // 在informers这个包中又定义了一个SharedInformerFactory，这个主要是包内抽象，所以此处继承了这个接口
        internalinterfaces.SharedInformerFactory
        // 对指定类型的通用访问
        ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
        // 等待所有的Informer都已经同步完成，这里同步其实就是遍历调用SharedInformer.HasSynced()
        WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool
        // 返回SharedIndexInformer
        InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer

        //返回相关SharedInformer
        Admissionregistration() admissionregistration.Interface
        Internal() apiserverinternal.Interface
        Apps() apps.Interface
        Autoscaling() autoscaling.Interface
        Batch() batch.Interface
        Certificates() certificates.Interface
        Coordination() coordination.Interface
        Core() core.Interface
        Discovery() discovery.Interface
        Events() events.Interface
        Extensions() extensions.Interface
        Flowcontrol() flowcontrol.Interface
        Networking() networking.Interface
        Node() node.Interface
        Policy() policy.Interface
        Rbac() rbac.Interface
        Scheduling() scheduling.Interface
        Storage() storage.Interface
    }
```

### internalinterfaces.SharedInformerFactory介绍

SharedInformerFactory继承了internalinterfaces.SharedInformerFactory，我们要了解一下这个内部的SharedInformerFactory定义。
```go
    // 代码源自client-go/informers/internalinterfaces/factory_interfaces.go
    type SharedInformerFactory interface {
        // 核心逻辑函数，类似于很多类的Run()函数
        Start(stopCh <-chan struct{})
        // 这个很关键，通过对象类型，返回SharedIndexInformer，这个SharedIndexInformer管理的就是指定的对象
        // NewInformerFunc用于当SharedInformerFactory没有这个类型的Informer的时候创建使用
        InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
    }

    // 创建Informer的函数定义，这个函数需要apiserver的客户端以及同步周期，这个同步周期在SharedInformers反复提到
    type NewInformerFunc func(kubernetes.Interface, time.Duration) cache.SharedIndexInformer
```
internalinterfaces.SharedInformerFactory其实只提供了一个能力，就是通过对象类型构造Informer。因为SharedInformerFactory管理的就是SharedIndexInformer对象，SharedIndexInformer存储的对象类型决定了他是什么Informer，致使SharedInformerFactory无需知道具体的Informer如何构造，所以需要外部传入构造函数，这样可以减低耦合性。

### SharedInformerFactory实现类

如下代码就是SharedInformerFactory接口的一种实现：

```go
    // 代码源自client-go/informers/factory.go
    type sharedInformerFactory struct {
        // apiserver的客户端，暂时不用关心怎么实现的，只要知道他能列举和监听资源就可以了
        client  kubernetes.Interface
        // 每个namesapce需要一个SharedInformerFactory，那cache用namespace建索引还有啥用呢？
        // 并不是所有的使用者都需要指定namesapce，比如kubectl，他就可以列举所有namespace的资源，所以他没有指定namesapce
        namespace   string
        // 这是个函数指针，用来调整列举选项的，这个选项用来client列举对象使用
        tweakListOptions internalinterfaces.TweakListOptionsFunc
        // 互斥锁
        lock  sync.Mutex
        // 默认的同步周期，这个在SharedInformer需要用
        defaultResync    time.Duration
        // 每个类型的Informer有自己自定义的同步周期
        customResync     map[reflect.Type]time.Duration
        // 每类对象一个Informer，但凡使用SharedInformerFactory构建的Informer同一个类型其实都是同一个Informer
        informers map[reflect.Type]cache.SharedIndexInformer
        // 各种Informer启动的标记
        startedInformers map[reflect.Type]bool
    }
```
有了具体的实现类，我们就可以看看这个类是怎么实现 SharedInformerFactory所有的功能点的。在开始分析前，我们来看看client-go里面提供构造SharedInformerFactory的几个函数：
```go
    // 代码源自client-go/tools/cache/shared_informer.go

    // 这是一个通用的构造SharedInformerFactory的接口函数
    func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration)
    SharedInformerFactory {
        // 最终是调用NewSharedInformerFactoryWithOptions()实现的
        return NewSharedInformerFactoryWithOptions(client, defaultResync)
    }

    // 这个构造函数增加了namesapce过滤和调整列举选项
    func NewFilteredSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) 
    SharedInformerFactory {
        // 最终是调用NewSharedInformerFactoryWithOptions()实现的，无非选项是2个
        // WithNamespace()和WithTweakListOptions()会在后文讲解
        return NewSharedInformerFactoryWithOptions(client, defaultResync, WithNamespace(namespace), WithTweakListOptions(tweakListOptions))
    }

    // 到了构造SharedInformerFactory核心函数了，其实SharedInformerOption是个有意思的东西
    func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) 
    SharedInformerFactory {
        // 默认只有apiserver的client以及同步周期是需要外部提供的其他的都是可以有默认值的
        factory := &sharedInformerFactory{
            client:           client,
            namespace:        v1.NamespaceAll,
            defaultResync:    defaultResync,
            informers:        make(map[reflect.Type]cache.SharedIndexInformer),
            startedInformers: make(map[reflect.Type]bool),
            customResync:     make(map[reflect.Type]time.Duration),
        }

        //逐一遍历各个选项函数，opt是选项函数，下面面有详细介绍
        for _, opt := range options {
            factory = opt(factory)
        }

        return factory
    }
```
工厂对象已经创建出来了，我们就要看他如何实现internalinterfaces里面定义的接口了，第一个就是Start()接口：
```go
    // 代码源自client-go/informers/factory.go

    // 其实sharedInformerFactory的Start()函数就是启动所有具体类型的Informer的过程
    // 因为每个类型的Informer都是SharedIndexInformer，需要把每个SharedIndexInformer都要启动起来
    func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
        // 加锁操作
        f.lock.Lock()
        defer f.lock.Unlock()
        // 遍历informers这个map
        for informerType, informer := range f.informers {
            // 看看这个Informer是否已经启动过
            if !f.startedInformers[informerType] {
                // 如果没启动过，那就启动一个协程执行SharedIndexInformer的Run()函数
                go informer.Run(stopCh)
                // 设置Informer已经启动的标记
                f.startedInformers[informerType] = true
            }
        }
    }
```
这个函数比较简单的原因在于SharedIndexInformer已经帮它做了很多事情， 那这些SharedIndexInformer是如何添加到sharedInformerFactory的呢？请看下面的代码：
```go
    // 代码源自client-go/informers/factory.go

    // InformerFor()相当于每个类型Informer的构造函数了，即便具体实现构造的地方是使用者提供的
    // 这个函数需要使用者传入对象类型，因为在sharedInformerFactory里面是按照对象类型组织的Informer
    // 更有趣的是这些Informer不是sharedInformerFactory创建的，需要使用者传入构造函数
    // 这样做既保证了每个类型的Informer只构造一次，同时又保证了具体Informer构造函数的私有化能力
    func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
        // 加锁操作
        f.lock.Lock()
        defer f.lock.Unlock()
        // 通过反射获取obj的类型
        informerType := reflect.TypeOf(obj)
        // 看看这个类型的Informer是否已经创建了？如果Informer已经创建，那么就复用这个Informer
        informer, exists := f.informers[informerType]
        if exists {
            return informer
        }

        // 获取这个类型定制的同步周期，如果定制的同步周期那就用统一的默认周期
        resyncPeriod, exists := f.customResync[informerType]
        if !exists {
            resyncPeriod = f.defaultResync
        }

        // 调用使用者提供构造函数，然后把创建的Informer保存起来
        informer = newFunc(f.client, resyncPeriod)
        f.informers[informerType] = informer

        return informer
    }

    // 代码源自client-go/informers/internalinterfaces/factory_interfaces.go

    // 这个函数定义就是具体类型Informer的构造函数，后面会有地方说明如何使用
    type NewInformerFunc func(kubernetes.Interface, time.Duration) cache.SharedIndexInformer
```
我们就以PodInformer为例子，通过他的部分代码说明上面的总结的内容。
```go
    // 代码源自client-go/informers/core/v1/pod.go

    // PodInformer是抽象类，Informer()就是获取SharedIndexInformer的接口函数
    type PodInformer interface {
        Informer() cache.SharedIndexInformer
        Lister() v1.PodLister
    }

    // 这个是PodInformer的实现类，看到了没，他需要工厂对象的指针，貌似明细了很多
    type podInformer struct {
        factory          internalinterfaces.SharedInformerFactory
        tweakListOptions internalinterfaces.TweakListOptionsFunc
        namespace        string
    }

    // 这个就是要传入工厂的构造函数了
    func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
        return NewFilteredPodInformer(
            client, 
            f.namespace, 
            resyncPeriod, 
            cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, 
            f.tweakListOptions
            )
    }
    // 这个是实现Informer()的地方，看到了把，这里面调用了工厂的InformerFor把自己注册进去
    func (f *podInformer) Informer() cache.SharedIndexInformer {
        return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
    }
```
也就是说SharedInformerFactory的使用者使用Core().Pod()这个接口构造了PodInformer，但是需要调用PodInformer.Informer()才能获取到的SharedIndexInformer。既然已经涉及到了具体的Informer，我们就开始看看每个都是干啥的吧？

client-go为了方便管理，把Informer分类管理。那我们拿core这个组来讲解吧。通过SharedInformerFactory().Core()获取内核Informer的分组就是构造了一个对象，如下代码所示：
```go
    // 代码源自client-go/informers/factory.go
    func (f *sharedInformerFactory) Core() core.Interface {
        // 调用了内核包里面的New()函数，详情见下文分析
        return core.New(f, f.namespace, f.tweakListOptions)
    }

    // 代码源自client-go/informers/core/interface.go
    // Interface又是一个被玩坏的名字，如果没有报名，根本不知道干啥的
    type Interface interface {
        V1() v1.Interface // 只有V1一个版本
    }
    // 这个是Interface的实现类，从名字上没任何关联吧？其实开发者命名也是挺有意思的，Interface定义的是接口
    // 供外部使用，group也有意义，因为Core确实是内核Informer的分组
    type group struct {
        // 需要工厂对象的指针
        factory          internalinterfaces.SharedInformerFactory
        // 这两个变量决定了Core这个分组对于SharedInformerFactory来说只有以下两个选项
        namespace        string
        tweakListOptions internalinterfaces.TweakListOptionsFunc
    }
    // 构造Interface的接口
    func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
        return &group{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
    }
    // 实现V1()这个接口的函数
    func (g *group) V1() v1.Interface {
        // 通过调用v1包的New()函数实现的，下面会有相应代码的分析
        return v1.New(g.factory, g.namespace, g.tweakListOptions)
    }
```

Core分组中的V1版本如下代码所示
```go
    // 代码源自client-go/informers/core/v1/interface.go

    type Interface interface {
        // 获取ComponentStatusInformer
        ComponentStatuses() ComponentStatusInformer
        // 获取ConfigMapInformer
        ConfigMaps() ConfigMapInformer
        // 获取EndpointsInformer
        Endpoints() EndpointsInformer
        // 获取EventInformer
        Events() EventInformer
        // 获取LimitRangeInformer
        LimitRanges() LimitRangeInformer
        // 获取NamespaceInformer
        Namespaces() NamespaceInformer
        // 获取NodeInformer
        Nodes() NodeInformer
        // 获取PersistentVolumeInformer
        PersistentVolumes() PersistentVolumeInformer
        // 获取PersistentVolumeClaimInformer
        PersistentVolumeClaims() PersistentVolumeClaimInformer
        // 获取PodInformer
        Pods() PodInformer
        // 获取PodTemplateInformer
        PodTemplates() PodTemplateInformer
        // 获取ReplicationControllerInformer
        ReplicationControllers() ReplicationControllerInformer
        // 获取ResourceQuotaInformer
        ResourceQuotas() ResourceQuotaInformer
        // 获取SecretInformer
        Secrets() SecretInformer
        // 获取ServiceInformer
        Services() ServiceInformer
        // 获取ServiceAccountInformer
        ServiceAccounts() ServiceAccountInformer
    }
    // 这个就是上面抽象类的实现了，这个和Core分组的命名都是挺有意思，分组用group作为实现类名
    // 这个用version作为实现类名，确实这个是V1版本
    type version struct {
        // 工厂的对象指针
        factory          internalinterfaces.SharedInformerFactory
        namespace        string
        tweakListOptions internalinterfaces.TweakListOptionsFunc
    }
    // 这个就是Core分组V1版本的构造函数啦
    func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
        // 应该好理解吧？
        return &version{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
    }
```
Core分组有管理了很多Informer，每一个Informer负责获取相应类型的对象。每个对象干啥用的不是本文讲解重点，所以下面的章节我以PodInformer为例说一说具体类型的Informer的实现。PodInformer是通过Core分组Pods()创建的，我们来看看代码实现：
```go
    // 代码源自client-go/informers/core/v1/interface.go
    // 上面我们已经说过了version是v1.Interface的实现
    func (v *version) Pods() PodInformer {
        // 返回了podInformer的对象，说明podInformer是PodInformer 实现类
        return &podInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}
    }

    // 代码源自client-go/informers/core/v1/pod.go
    // PodInformer定义了两个接口，分别为Informer()和Lister()，
    // Informer()用来获取SharedIndexInformer对象
    // Lister()用来获取PodLister对象
    type PodInformer interface {
        Informer() cache.SharedIndexInformer
        Lister() v1.PodLister
    }
    // PodInformer的实现类，参数都是上面层层传递下来的，这里不说了
    type podInformer struct {
        factory          internalinterfaces.SharedInformerFactory
        tweakListOptions internalinterfaces.TweakListOptionsFunc
        namespace        string
    }
    // 这个就是需要传递给SharedInformerFactory的构造函数啦
    func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
        // NewFilteredPodInformer下面有代码注释
        return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
    }
    // 实现了PodInformer.Informer()接口函数
    func (f *podInformer) Informer() cache.SharedIndexInformer {
        // 此处调用了工厂实现了Informer的创建
        return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
    }
    // 实现了PodInformer.Lister()接口函数
    func (f *podInformer) Lister() v1.PodLister {
        return v1.NewPodLister(f.Informer().GetIndexer())
    }
    // 真正创建PodInformer的函数
    func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {

        return cache.NewSharedIndexInformer(
            // 需要ListWatch两个函数，就是用apiserver的client实现的
            &cache.ListWatch{
                ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
                        if tweakListOptions != nil {
                            tweakListOptions(&options)
                        }
                        return client.CoreV1().Pods(namespace).List(options)
                    },
                WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
                        if tweakListOptions != nil {
                            tweakListOptions(&options)
                        }
                        return client.CoreV1().Pods(namespace).Watch(options)
                    },
            },
            // 这个是要传入对象的类型，肯定是Pod了
            &corev1.Pod{},
            // 同步周期
            resyncPeriod,
            // 对象键的计算函数
            indexers,
        )

    }
```

## client-go的SharedInformer

Informer(就是SharedInformer)是client-go的重要组成部分，引用了官方的图，可以看到Informer在client-go中的位置。   
![](../img/general-pattern-k8s-controller.png)
SharedInformer名字上直译就是信息提供者，至于Shared是什么意思，我们来看看官方注释：
```
Shared指的是多个listeners共享同一个cache，而且资源的变化会通过cache通知到listeners。listerners指的就是OnAdd、OnUpdate、OnDelete这些回调函数背后的对象，而cache就是Local Stroe（Indexer）。      
```
我们先对上面的图做一些初步的认识：
1. List/Watch：List是列举apiserver中对象的接口，Watch是监控apiserver资源变化的接口；
2. Reflector：反射器，实现对apiserver指定类型对象的监控，其中反射实现的就是把监控的结果实例化成具体的对象；
3. DeltaIFIFO：将Reflector监控的变化的对象形成一个FIFO队列，此处的Delta就是变化；
4. LocalStore：指的就是Indexer的实现cache，这里面缓存的就是apiserver中的对象(其中有一部分可能还在DeltaFIFO中)，此时使用者再查询对象的时候就直接从cache中查找，减少了apiserver的压力；
5. Callbacks：通知回调函数，Infomer感知的所有对象变化都是通过回调函数通知使用者(Listener)；    

### ListerWatcher

ListerWatcher是一个interface类型，定义如下：
```go
// 代码源自client-go/tools/cache/listwatch.go

// 其中metav1.ListOptions，runtime.Object，watch.Interface都定义在apimachinery这个包中

type ListerWatcher interface {
  // 根据选项列举对象
  List(options metav1.ListOptions) (runtime.Object, error)

  // 根据选项监控对象变化
  Watch(options metav1.ListOptions) (watch.Interface, error)
}

type Interface interface {
	// Stop stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// ResultChan returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, the implementation will close this channel and
	// release any resources used by the watch.
	ResultChan() <-chan Event
}
```
知道ListerWatcher是通过apiserver的API来列举和监控的就行了，具体是如何实现的并不重要。
```
ListerWatcher是针对某一类对象的，比如Pod，不是所有对象的，这个在构造ListerWatcher对象的时候由apiserver的client类型决定了。
```

### Reflector实现

```go
    // 代码源自client-go/tools/cache/reflector.go

    type Reflector struct {

        name string              // 名字
        metrics *reflectorMetrics// 但凡遇到metrics多半是用于做监控的，可以忽略
        expectedType reflect.Type // 反射的类型，也就是要监控的对象类型，比如Pod
        store Store // 存储，就是DeltaFIFO
        listerWatcher ListerWatcher // 这个是用来从apiserver获取资源用的
        period       time.Duration // 反射器在List和Watch的时候理论上是死循环，只有出现错误才会退出
        // 这个变量用在出错后多长时间再执行List和Watch，默认值是1秒钟
        resyncPeriod time.Duration  // 重新同步的周期，很多人肯定认为这个同步周期指的是从apiserver的同步周期
        // 其实这里面同步指的是shared_informer使用者需要定期同步全量对象
        ShouldResync func() bool// 如果需要同步，调用这个函数问一下，当然前提是该函数指针不为空
        clock clock.Clock  // 时钟
        lastSyncResourceVersion string// 最后一次同步的资源版本
        lastSyncResourceVersionMutex sync.RWMutex   // 还专门为最后一次同步的资源版本弄了个锁
    }
```
根据上面定义的成员变量，我们可以推导出：
1. listerWatcher用于获取和监控资源，lister可以获取对象的全量，watcher可以获取对象的增量(变化)；
2. 系统会周期性的执行list-watch的流程，一旦过程中失败就要重新执行流程，这个重新执行的周期就是period指定的；
3. expectedType规定了监控对象的类型，非此类型的对象将会被忽略；
4. 实例化后的expectedType类型的对象会被添加到store中；
5. kubernetes资源在apiserver中都是有版本的，对象的任何修改(添加、删除、更新)都会造成资源版本更新，所以lastSyncResourceVersion就是指的这个版本；
6. 如果使用者需要定期同步全量对象，那么Reflector就会定期产生全量对象的同步事件给DeltaFIFO;
按照上面的推导，基本每个成员变量都涉及到了，仿佛我们已经知道Reflector的工作原理了，下面我们就要通过源码逐一验证上面的推导。Reflector有一个Run()函数，这个是Reflector的核心功能流程，我们可以沿着这个流程分析：
```go
    // 代码源自client-go/tools/cache/reflector.go

    func (r *Reflector) Run(stopCh <-chan struct{}) {

        // func Until(f func(), period time.Duration, stopCh <-chan struct{})
        // 是下面函数的声明
        // 这里面我们不用关心wait.Until是如何实现的，只要知道他调用函数f
        // 会被每period周期执行一次
        // 意思就是f()函数执行完毕再等period时间后在执行一次，
        // 也就是r.ListAndWatch()会被周期性的调用

        wait.Until(func() {
            if err := r.ListAndWatch(stopCh); err != nil {
            utilruntime.HandleError(err)
            }
        }, r.period, stopCh)

    }

    // 代码源自client-go/tools/cache/reflector.go

    func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {

        var resourceVersion string

        // 很多存储类的系统都是这样设计的，数据采用版本的方式记录，
        // 数据每变化(添加、删除、更新)都会触发版本更新，
        // 这样的做法可以避免全量数据访问。以apiserver资源监控为例，
        // 只要监控比缓存中资源版本大的对象就可以了，
        // 把变化的部分更新到缓存中就可以达到与apiserver一致的效果，
        // 一般资源的初始版本为0，从0版本开始列举就是全量的对象了
        options := metav1.ListOptions{ResourceVersion: "0"}

        // 与监控相关的内容不多解释
        r.metrics.numberOfLists.Inc()
        start := r.clock.Now()

        // 列举资源，这部分是apimachery相关的内容，读者感兴趣可以自己了解
        list, err := r.listerWatcher.List(options)
        if err != nil {
            return fmt.Errorf("%s: Failed to list %v: %v", r.name, r.expectedType, err)
        }

        // 还是监控相关的
        r.metrics.listDuration.Observe(time.Since(start).Seconds())
        // 下面的代码主要是利用apimachinery相关的函数实现，
        // 就是把列举返回的结果转换为对象数组
        // 下面的代码大部分来自apimachinery，此处不做过多说明，
        // 读者只要知道实现什么功能就行了
        listMetaInterface, err := meta.ListAccessor(list)
        if err != nil {
            return fmt.Errorf("%s: Unable to understand list result %#v: %v", r.name, list, err)
        }

        resourceVersion = listMetaInterface.GetResourceVersion()
        
        items, err := meta.ExtractList(list)
        if err != nil {
            return fmt.Errorf("%s: Unable to understand list result %#v (%v)", r.name, list, err)
        }

        // 和监控相关的内容
        r.metrics.numberOfItemsInList.Observe(float64(len(items)))

        // 以上部分都是对象实例化的过程，可以称之为反射，
        // 也是Reflector这个名字的主要来源，本文不是讲解反射原理的，
        // 而是作为SharedInformer的前端，所以我们重点介绍的是对象
        // 在SharedInformer中流转过程，所以反射原理部分不做为重点讲解
        // 这可是真正从apiserver同步过来的全量对象，所以要同步到DeltaFIFO中
        if err := r.syncWith(items, resourceVersion); err != nil {
            return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
        }

        // 设置最新的同步的对象版本
        r.setLastSyncResourceVersion(resourceVersion)

        // 下面要启动一个后台协程实现定期的同步操作，
        // 这个同步就是将SharedInformer里面的对象全量以同步事件的方式通知使用者
        // 我们暂且称之为“后台同步协程”，Run()函数退出需要后台同步协程退出，
        // 所以下面的cancelCh就是干这个用的

        // 利用defer close(cancelCh)实现的，而resyncerrc是后台同步协程反向
        // 通知Run()函数的报错通道
        // 当后台同步协程出错，Run()函数接收到信号就可以退出了
        resyncerrc := make(chan error, 1)
        cancelCh := make(chan struct{})
        defer close(cancelCh)

        // 下面这个匿名函数就是后台同步协程的函数了
        go func() {

            // resyncCh返回的就是一个定时器，如果resyncPeriod这个为0那么就会返回
            // 一个永久定时器，cleanup函数是用来清理定时器的
            resyncCh, cleanup := r.resyncChan()
            defer func() {
                cleanup()
            }()

            // 死循环等待各种信号
            for {
                // 只有定时器有信号才继续处理，其他的都会退出
                select {
                    case <-resyncCh:
                    case <-stopCh:
                    return
                    case <-cancelCh:
                    return
                }

                // ShouldResync是个函数地址，创建反射器对象的时候传入，即便时间到了，
                //也要通过函数问问是否需要同步
                if r.ShouldResync == nil || r.ShouldResync() {

                // 我们知道这个store是DeltaFIFO，DeltaFIFO.Resync()做了什么，
                // 读者自行温习相关的文章~
                // 就在这里实现了我们前面提到的同步，从这里看所谓的同步就是以
                // 全量对象同步事件的方式通知使用者
                if err := r.store.Resync(); err != nil {
                    resyncerrc <- err
                    return
                }

                }

                // 清理掉当前的计时器，获取下一个同步时间定时器

                cleanup()
                resyncCh, cleanup = r.resyncChan()
            }
        }()

        // 前面已经列举了全量对象，接下来就是watch的逻辑了

        for {

            // 如果有退出信号就立刻返回，否则就会往下走，因为有default.

            select {
                case <-stopCh:
                    return nil
                default:
            }

            // 计算watch的超时时间
            timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))

            // 设置watch的选项，因为前期列举了全量对象，从这里只要监听最新版本以后的
            // 资源就可以了
            // 如果没有资源变化总不能一直挂着吧？也不知道是卡死了还是怎么了，
            // 所以有一个超时会好一点
            options = metav1.ListOptions{
                ResourceVersion: resourceVersion,
                TimeoutSeconds: &timeoutSeconds,
            }

            // 监控相关
            r.metrics.numberOfWatches.Inc()

            // 开始监控对象
            w, err := r.listerWatcher.Watch(options)

            // watch产生错误了，大部分错误就要退出函数然后再重新来一遍流程
            if err != nil {

                switch err {
                    case io.EOF:
                    case io.ErrUnexpectedEOF:
                    default:
                        utilruntime.HandleError(fmt.Errorf("%s: Failed to watch %v: %v", r.name, r.expectedType, err))
                }

                // 类似于网络拒绝连接的错误要等一会儿再试，因为可能网络繁忙
                if urlError, ok := err.(*url.Error); ok {
                    if opError, ok := urlError.Err.(*net.OpError); ok {
                        if errno, ok := opError.Err.(syscall.Errno); ok && errno == syscall.ECONNREFUSED {
                            time.Sleep(time.Second)
                            continue
                        }
                    }
                }

                return nil
            }

            // watch返回是流，apiserver会将变化的资源通过这个流发送出来，client-go最终通过chan实现的
            // 所以watchHandler()是一个需要持续从chan读取数据的流程，所以需要传入resyncerrc和stopCh
            // 用于异步通知退出或者后台同步协程错误
            if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
                if err != errorStopRequested {
                    glog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
                }
                return nil
            }
        }

    }
```

### Controller实现

### SharedInformer分析

#### SharedInformer的定义

#### CacheMutationDetector

#### sharedProcessor分析

##### processorListener分析

##### sharedProcessor管理processorListener

#### SharedInformer实现