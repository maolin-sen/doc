# client-go 源码分析


## kubernetes client-go解析

下图为来自官方的Client-go架构图:
![]()

### Reflector

reflector使用listerWatcher获取资源，并将其保存在store中，此处的store就是DeltaFIFO，Reflector核心处理函数为ListAndWatch(client-go/tools/cache/reflector.go)。ListAndWatch在Reflector.Run函数中启动，并以Reflector.period周期性进行调度。ListAndWatch使用resourceVersion来获取资源的增量变化：在List时会获取资源的首个resourceVersion值，在Watch的时候会使用List获取的resourceVersion来获取资源的增量变化，然后将获取到的资源的resourceVersion保存起来，作为下一次Watch的基线。

```go
    // client-go/tools/cache/reflector.go
    type Reflector struct {
        // name identifies this reflector. By default it will be a 
        // file:line if possible.
        name string
        // metrics tracks basic metric information about the reflector
        metrics *reflectorMetrics
        // The type of object we expect to place in the store.
        expectedType reflect.Type
        // The destination to sync up with the watch source
        store Store
        // listerWatcher is used to perform lists and watches.
        listerWatcher ListerWatcher
        // period controls timing between one watch ending and
        // the beginning of the next one.
        period       time.Duration
        resyncPeriod time.Duration
        ShouldResync func() bool
        // clock allows tests to manipulate time
        clock clock.Clock
        // lastSyncResourceVersion is the resource version token last
        // observed when doing a sync with the underlying store
        // it is thread safe, but not synchronized with the underlying store
        lastSyncResourceVersion string
        // lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
        lastSyncResourceVersionMutex sync.RWMutex
        // WatchListPageSize is the requested chunk size of initial and resync watch lists.
        // Defaults to pager.PageSize.
        WatchListPageSize int64
    }

    // client-go/tools/cache/reflector.go
    func (r *Reflector) Run(stopCh <-chan struct{}) {
        wait.Until(func() {
            if err := r.ListAndWatch(stopCh); err != nil {
                utilruntime.HandleError(err)
            }
        }, r.period, stopCh)
    }
```

### DeltaFIFO

```go
    //client-go/tools/cache/delta_fifo.go
    type DeltaFIFO struct {
        // lock/cond protects access to 'items' and 'queue'.
        lock sync.RWMutex
        cond sync.Cond

        // We depend on the property that items in the set are in
        // the queue and vice versa, and that all Deltas in this
        // map have at least one Delta.
        items map[string]Deltas
        queue []string

        // populated is true if the first batch of items inserted by Replace() has been populated
        // or Delete/Add/Update was called first.
        populated bool
        // initialPopulationCount is the number of items inserted by the first call of Replace()
        initialPopulationCount int

        // keyFunc is used to make the key used for queued item
        // insertion and retrieval, and should be deterministic.
        keyFunc KeyFunc  //用于计算Delta的key

        // knownObjects list keys that are "known", for the
        // purpose of figuring out which items have been deleted
        // when Replace() or Delete() is called.
        knownObjects KeyListerGetter// Indication the queue is closed.
        // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
        // Currently, not used to gate any of CRED operations.
        closed     bool
        closedLock sync.Mutex
    }

    // A KeyListerGetter is anything that knows how to list its keys and look up by key.
    type KeyListerGetter interface {
        KeyLister
        KeyGetter
    }

    // A KeyLister is anything that knows how to list its keys.
    type KeyLister interface {
        ListKeys() []string
    }

    // A KeyGetter is anything that knows how to get the value stored under a given key.
    type KeyGetter interface {
        GetByKey(key string) (interface{}, bool, error)
    }
```

### Indexer

Indexer保存了来自apiServer的资源。使用listWatch方式来维护资源的增量变化。通过这种方式可以减小对apiServer的访问，减轻apiServer端的压力。
Indexer的接口定义如下，它继承了Store接口，Store中定义了对对象的增删改查等方法。

```go
    // client-go/tools/cache/index.go
    type Indexer interface {
        Store
        // Retrieve list of objects that match on the named indexing function
        Index(indexName string, obj interface{}) ([]interface{}, error)
        // IndexKeys returns the set of keys that match on the named indexing function.
        IndexKeys(indexName, indexKey string) ([]string, error)
        // ListIndexFuncValues returns the list of generated values of an Index func
        ListIndexFuncValues(indexName string) []string
        // ByIndex lists object that match on the named indexing function with the exact key
        ByIndex(indexName, indexKey string) ([]interface{}, error)
        // GetIndexer return the indexers
        GetIndexers() Indexers
        // AddIndexers adds more indexers to this store.
        // If you call this after you already have data
        // in the store, the results are undefined.
        AddIndexers(newIndexers Indexers) error
    }


    // client-go/tools/cache/store.go
    type Store interface {
        Add(obj interface{}) error
        Update(obj interface{}) error
        Delete(obj interface{}) error
        List() []interface{}
        ListKeys() []string
        Get(obj interface{}) (item interface{}, exists bool, err error)
        GetByKey(key string) (item interface{}, exists bool, err error)

        // Replace will delete the contents of the store,
        // using instead the given list. Store takes ownership
        // of the list, you should not reference
        // it after calling this function.
        Replace([]interface{}, string) error
        Resync() error
    }
```

### ListWatch

Lister用于获取某个资源(如Pod)的全量，Watcher用于获取某个资源的增量变化。实际使用中Lister和Watcher都从apiServer获取资源信息，Lister一般用于首次获取某资源(如Pod)的全量信息，而Watcher用于持续获取该资源的增量变化信息。Lister和Watcher的接口定义如下，使用NewListWatchFromClient函数来初始化ListerWatcher

```go
    // client-go/tools/cache/listwatch.go
    type Lister interface {
        // List should return a list type object; the Items field will be extracted, and the
        // ResourceVersion field will be used to start the watch in the right place.
        List(options metav1.ListOptions) (runtime.Object, error)
    }

    // Watcher is any object that knows how to start a watch on a resource.
    type Watcher interface {
        // Watch should begin a watch at the specified version.
        Watch(options metav1.ListOptions) (watch.Interface, error)
    }

    // ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
    type ListerWatcher interface {
        Lister
        Watcher
    }
```



### Controller

controller的结构如下，其包含一个配置变量config，在注释中可以看到Config.Queue就是DeltaFIFO。controller定义了如何调度Reflector。

```go
    // client-go/tools/cache/controller.go
    type controller struct {
        config         Config
        reflector      *Reflector
        reflectorMutex sync.RWMutex
        clock          clock.Clock
    }

    // client-go/tools/cache/controller.go
    type Config struct {
        // The queue for your objects - has to be a DeltaFIFO due to
        // assumptions in the implementation. Your Process() function
        // should accept the output of this Queue's Pop() method.
        Queue
        // Something that can list and watch your objects.
        ListerWatcher
        // Something that can process your objects.
        Process ProcessFunc
        // The type of your objects.
        ObjectType runtime.Object
        // Reprocess everything at least this often.
        // Note that if it takes longer for you to clear the queue than this
        // period, you will end up processing items in the order determined
        // by FIFO.Replace(). Currently, this is random. If this is a
        // problem, we can change that replacement policy to append new
        // things to the end of the queue instead of replacing the entire
        // queue.
        FullResyncPeriod time.Duration
        // ShouldResync, if specified, is invoked when the controller's reflector determines the next
        // periodic sync should occur. If this returns true, it means the reflector should proceed with
        // the resync.
        ShouldResync ShouldResyncFunc
        // If true, when Process() returns an error, re-enqueue the object.
        // TODO: add interface to let you inject a delay/backoff or drop
        //       the object completely if desired. Pass the object in
        //       question to this interface as a parameter.
        RetryOnError bool
    }

```

controller的框架比较简单。
    1. 它使用wg.StartWithChannel启动Reflector.Run，相当于启动了一个DeltaFIFO的生产者(wg.StartWithChannel(stopCh, r.Run)表示可以将r.Run放在独立的协程运行，并可以使用stopCh来停止r.Run)；
    2. 使用wait.Until来启动一个消费者(wait.Until(c.processLoop, time.Second, stopCh)表示每秒会触发一次c.processLoop，但如果c.processLoop在1秒之内没有结束，则运行c.processLoop继续运行，不会结束其运行状态)

```go
    // client-go/tools/cache/controller.go
    func (c *controller) Run(stopCh <-chan struct{}) {
        defer utilruntime.HandleCrash()
        go func() {
            <-stopCh
            c.config.Queue.Close()
        }()
        r := NewReflector(
            c.config.ListerWatcher,
            c.config.ObjectType,
            c.config.Queue,
            c.config.FullResyncPeriod,
        )
        r.ShouldResync = c.config.ShouldResync
        r.clock = c.clock

        c.reflectorMutex.Lock()
        c.reflector = r
        c.reflectorMutex.Unlock()

        var wg wait.Group
        defer wg.Wait()

        wg.StartWithChannel(stopCh, r.Run)

        wait.Until(c.processLoop, time.Second, stopCh)
    }
```
processLoop的框架也很简单，它运行了DeltaFIFO.Pop函数，用于消费DeltaFIFO中的对象，并在DeltaFIFO.Pop运行失败后可能重新处理该对象(AddIfNotPresent)

```go
    // client-go/tools/cache/controller.go
    func (c *controller) processLoop() {
        for {
            obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
            if err != nil {
                if err == FIFOClosedError {
                    return
                }
                if c.config.RetryOnError {
                    // This is the safe way to re-enqueue.
                    c.config.Queue.AddIfNotPresent(obj)
                }
            }
        }
    }

    //client-go/tools/cache/shared_informer.go
    func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
        defer utilruntime.HandleCrash()

        fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

        cfg := &Config{
            Queue:            fifo,
            ListerWatcher:    s.listerWatcher,
            ObjectType:       s.objectType,
            FullResyncPeriod: s.resyncCheckPeriod,
            RetryOnError:     false,
            ShouldResync:     s.processor.shouldResync,

            Process: s.HandleDeltas,
        }
    ...
```


### ShareInformer

下图为SharedInformer的运行图。可以看出SharedInformer启动了controller，reflector，并将其与Indexer结合起来。

### SharedInformerFactory

sharedInformerFactory接口的内容如下，它按照group和version对informer进行了分类。

```go
    // client-go/informers/factory.go
    type SharedInformerFactory interface {
        internalinterfaces.SharedInformerFactory
        ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
        WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

        Admissionregistration() admissionregistration.Interface
        Apps() apps.Interface
        Auditregistration() auditregistration.Interface
        Autoscaling() autoscaling.Interface
        Batch() batch.Interface
        Certificates() certificates.Interface
        Coordination() coordination.Interface
        Core() core.Interface
        Events() events.Interface
        Extensions() extensions.Interface
        Networking() networking.Interface
        Node() node.Interface
        Policy() policy.Interface
        Rbac() rbac.Interface
        Scheduling() scheduling.Interface
        Settings() settings.Interface
        Storage() storage.Interface
    }
```
sharedInformerFactory负责在不同的chan中启动不同的informer

```go
    // client-go/informers/factory.go
    func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
        f.lock.Lock()
        defer f.lock.Unlock()

        for informerType, informer := range f.informers {
            if !f.startedInformers[informerType] {
                go informer.Run(stopCh)
                f.startedInformers[informerType] = true
            }
        }
    }
```

那sharedInformerFactory启动的informer又是怎么注册到sharedInformerFactory.informers中的呢？informer的注册函数统一为InformerFor，代码如下，所有类型的informer都会调用该函数注册到sharedInformerFactory。

```go

    // client-go/informers/factory.go
    func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
        f.lock.Lock()
        defer f.lock.Unlock()

        informerType := reflect.TypeOf(obj)
        informer, exists := f.informers[informerType]
        if exists {
            return informer
        }

        resyncPeriod, exists := f.customResync[informerType]
        if !exists {
            resyncPeriod = f.defaultResync
        }

        informer = newFunc(f.client, resyncPeriod)
        f.informers[informerType] = informer

        return informer
    }

```

### workqueue

indexer用于保存apiserver的资源信息，而workqueue用于保存informer中的handler处理之后的数据。workqueue的接口定义如下： 

```go
    // client-go/util/workqueue/queue.go
    type Interface interface {
        Add(item interface{})
        Len() int
        Get() (item interface{}, shutdown bool)
        Done(item interface{})
        ShutDown()
        ShuttingDown() bool
    }

    // Type is a work queue.
    type Type struct {
        // queue defines the order in which we will work on items. Every
        // element of queue should be in the dirty set and not in the
        // processing set.
        queue []t
        // dirty defines all of the items that need to be processed.
        dirty set
        // Things that are currently being processed are in the processing set.
        // These things may be simultaneously in the dirty set. When we finish
        // processing something and remove it from this set, we'll check if
        // it's in the dirty set, and if so, add it to the queue.
        processing set
        cond *sync.Cond
        shuttingDown bool
        metrics queueMetrics
        unfinishedWorkUpdatePeriod time.Duration
        clock                      clock.Clock
    }
```




## Kubernetes的informer原理

Kubernetes的控制器模式是其非常重要的一个设计模式，整个Kubernetes定义的资源对象以及其状态都保存在etcd数据库中，通过apiserver对其进行增删查改，而各种各样的控制器需要从apiserver及时获取这些对象，然后将其应用到实际中，即将这些对象的实际状态调整为期望状态，让他们保持匹配。

不过这因为这样，各种控制器需要和apiserver进行频繁交互，需要能够及时获取对象状态的变化，而如果简单的通过暴力轮询的话，会给apiserver造成很大的压力，且效率很低，因此，Kubernetes设计了Informer这个机制，用来作为控制器跟apiserver交互的桥梁，它主要有两方面的作用：
1. 依赖Etcd的List&Watch机制，在本地维护了一份目标对象的缓存。
    ```
        Etcd的Watch机制能够使客户端及时获知这些对象的状态变化，然后通过List机制，更新本地缓存，这样就在客户端为这些API对象维护了一份和Etcd数据库中几乎一致的数据，然后控制器等客户端就可以直接访问缓存获取对象的信息，而不用去直接访问apiserver，这一方面显著提高了性能，另一方面则大大降低了对apiserver的访问压力；
    ``` 
2. 依赖Etcd的Watch机制，触发控制器等客户端注册到Informer中的事件方法。
    ```
        客户端可能会对某些对象的某些事件感兴趣，当这些事件发生时，希望能够执行某些操作，比如通过apiserver新建了一个pod，那么kube-scheduler中的控制器收到了这个事件，然后将这个pod加入到其队列中，等待进行调度。
    ```
Kubernetes的各个组件本身就内置了非常多的控制器，而自定义的控制器也需要通过Informer跟apiserver进行交互，因此，Informer在Kubernetes中应用非常广泛，下面重点分析下Informer的机制原理，以加深对其的理解。

### 使用方法

先来看看Informer是怎么用的，以Endpoint为例，来看下其使用Informer的相关代码：

* 创建Informer工厂
  
    ```go
        # client-go/informers/factory.go

        // SharedInformerFactory provides access to a shared informer and lister for dynamic client
        type SharedInformerFactory interface {
            Start(stopCh <-chan struct{})
            ForResource(gvr schema.GroupVersionResource) informers.GenericInformer
            WaitForCacheSync(stopCh <-chan struct{}) map[schema.GroupVersionResource]bool
        }
        sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())
    ```
    首先创建了一个SharedInformerFactory，这个结构主要有两个作用：
    1. 一个是用来作为创建Informer的工厂，典型的工厂模式，在Kubernetes中这种设计模式也很常用；
    2. 一个是共享Informer，所谓共享，就是多个Controller可以共用同一个Informer，因为不同的Controller可能对同一种API对象感兴趣，这样相同的API对象，缓存就只有一份，通知机制也只有一套，大大提高了效率，减少了资源浪费。
   
* 创建对象Informer结构体

    ```go
        # client-go/informers/core/v1/endpoints.go

        type EndpointsInformer interface {
            Informer() cache.SharedIndexInformer
            Lister() v1.EndpointsLister
        }
        endpointsInformer := kubeInformerFactory.Core().V1().Endpoints()
    ```
    使用InformerFactory创建出对应版本的对象的Informer结构体，如Endpoints对象对应的就是EndpointsInformer结构体，该结构体实现了两个方法：Informer()和Lister()
      1. 前者用来构建出最终的Informer，即我们本篇文章的重点：SharedIndexInformer，
      2. 后者用来获取创建出来的Informer的缓存接口：Indexer，该接口可以用来查询缓存的数据。

* 注册事件方法
  
    ```go
        # Client-go/tools/cache/shared_informer.go

        informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
            AddFunc:    onAdd,
            UpdateFunc: func(interface{}, interface{}) { fmt.Println("update not implemented") }, // 此处省略 workqueue 的使用
            DeleteFunc: func(interface{}) { fmt.Println("delete not implemented") },
        })

        func onAdd(obj interface{}) {
            node := obj.(*corev1.Endpoint)
            fmt.Println("add a endpoint:", endpoint.Name)
        }
    ```
    1. 调用Infomer()创建出来SharedIndexInformer，然后向其中注册事件方法，这样当有对应的事件发生时，就会触发这里注册的方法去做相应的事情。
    2. 调用Lister()获取到缓存接口，就可以通过它来查询Informer中缓存的数据了，而且Informer中缓存的数据，是可以有索引的，这样可以加快查询的速度。
   
* 启动Informer
  
    ```go
        # kubernetes/cmd/kube-controller-manager/app/controllermanager.go
        controllerContext.InformerFactory.Start(controllerContext.Stop)
    ```
    这里InformerFactory的启动，会遍历Factory中创建的所有Informer，依次将其启动。

### 机制解析

Informer的实现都是在client-go这个库中，通过上述的工厂方法，其实最终创建出来的是一个叫做SharedIndexInformer的结构体：

```go
    # k8s.io/client-go/tools/cache/shared_informer.go

    type sharedIndexInformer struct {
        indexer    Indexer
        controller Controller

        processor             *sharedProcessor
        cacheMutationDetector MutationDetector

        listerWatcher ListerWatcher
        ......
    }

    func NewSharedIndexInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
        realClock := &clock.RealClock{}
        sharedIndexInformer := &sharedIndexInformer{
            processor:                       &sharedProcessor{clock: realClock},
            indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
            listerWatcher:                   lw,
            objectType:                      exampleObject,
            resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
            defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
            cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", exampleObject)),
            clock:                           realClock,
        }
        return sharedIndexInformer
    }
```
可以看到，在创建SharedIndexInformer时，就创建出了processor, indexer等结构，而在Informer启动时，还创建出了controller, fifo queue, reflector等结构。

### Reflector

Reflector的作用，就是通过List&Watch的方式，从apiserver获取到感兴趣的对象以及其状态，然后将其放到一个称为”Delta”的先进先出队列中。所谓的Delta FIFO Queue，就是队列中的元素除了对象本身外，还有针对该对象的事件类型：

```go
    type Delta struct {
        Type   DeltaType
        Object interface{}
    }
```

目前有5种Type: Added, Updated, Deleted, Replaced, Resync，所以，针对同一个对象，可能有多个Delta元素在队列中，表示对该对象做了不同的操作，比如短时间内，多次对某一个对象进行了更新操作，那么就会有多个Updated类型的Delta放入到队列中。后续队列的消费者，可以根据这些Delta的类型，来回调注册到Informer中的事件方法。
而所谓的List&Watch，就是

  1. 先调用该API对象的List接口，获取到对象列表，将它们添加到队列中，Delta元素类型为Replaced，
  2. 然后再调用Watch接口，持续监听该API对象的状态变化事件，将这些事件按照不同的事件类型，组成对应的Delta类型，添加到队列中，Delta元素类型有Added, Updated, Deleted三种。
   
此外，Informer还会周期性的发送Resync类型的Delta元素到队列中，目的是为了周期性的触发注册到Informer中的事件方法UpdateFunc，保证对象的期望状态和实际状态一致，该周期是由一个叫做resyncPeriod的参数决定的，在向Informer中添加EventHandler时，可以指定该参数，若为0的话，则关闭该功能。需要注意的是，Resync类型的Delta元素中的对象，是通过Indexer从缓存中获取到的，而不是直接从apiserver中拿的，即这里resync的，其实是”缓存”的对象的期望状态和实际状态的一致性。

根据以上Reflector的机制，依赖Etcd的Watch机制，通过事件来获知对象变化状态，建立本地缓存。即使在Informer中，也没有周期性的调用对象的List接口，正常情况下，List&Watch只会执行一次，即先执行List把数据拉过来，放入队列中，后续就进入Watch阶段。

那什么时候才会再执行List呢？其实就是异常的时候，在List或者Watch的过程中，如果有异常，比如apiserver重启了，那么Reflector就开始周期性的执行List&Watch，直到再次正常进入Watch阶段。为了在异常时段，不给apiserver造成压力，这个周期是一个称为backoff的可变的时间间隔，默认是一个指数型的间隔，即越往后重试的间隔越长，到一定时间又会重置回一开始的频率。而且，为了让不同的apiserver能够均匀负载这些Watch请求，客户端会主动断开跟apiserver的连接，这个超时时间为60秒，然后重新发起Watch请求。此外，在控制器重启过程中，也会再次执行List，所以会观察到之前已经创建好的API对象，又重新触发了一遍AddFunc方法。

从以上这些点，可以看出来，Kubernetes在性能和稳定性的提升上，还是下了很多功夫的。

### Controller

这里Controller的作用是通过轮询不断从队列中取出Delta元素，根据元素的类型，一方面通过Indexer更新本地的缓存，一方面调用Processor来触发注册到Informer的事件方法：

```go
    # k8s.io/client-go/tools/cache/controller.go


    func (c *controller) processLoop() {
        for {
            obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
        }
    }
```

这里的c.config.Process是定义在shared_informer.go中的HandleDeltas()方法：

```go
    # k8s.io/client-go/tools/cache/shared_informer.go


    func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
        s.blockDeltas.Lock()
        defer s.blockDeltas.Unlock()


        // from oldest to newest
        for _, d := range obj.(Deltas) {
            switch d.Type {
            case Sync, Replaced, Added, Updated:
                s.cacheMutationDetector.AddObject(d.Object)
                if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
                    if err := s.indexer.Update(d.Object); err != nil {
                        return err
                    }


                    isSync := false
                    switch {
                    case d.Type == Sync:
                        // Sync events are only propagated to listeners that requested resync
                        isSync = true
                    case d.Type == Replaced:
                        if accessor, err := meta.Accessor(d.Object); err == nil {
                            if oldAccessor, err := meta.Accessor(old); err == nil {
                                // Replaced events that didn't change resourceVersion are treated as resync events
                                // and only propagated to listeners that requested resync
                                isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
                            }
                        }
                    }
                    s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
                } else {
                    if err := s.indexer.Add(d.Object); err != nil {
                        return err
                    }
                    s.processor.distribute(addNotification{newObj: d.Object}, false)
                }
            case Deleted:
                if err := s.indexer.Delete(d.Object); err != nil {
                    return err
                }
                s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
            }
        }
        return nil
    }
```

#### Processer & Listener

Processer和Listener则是触发事件方法的机制，在创建Informer时会创建一个Processer，而在向Informer中通过调用AddEventHandler()注册事件方法时会为每一个Handler生成一个Listener，然后将该Lisener中添加到Processer中，每一个Listener中有两个channel：addCh和nextCh。Listener通过select监听在这两个channel上，当Controller从队列中取出新的元素时会调用processer来给它的listener发送“通知”，这个“通知”就是向addCh中添加一个元素，即add()，然后一个goroutine就会将这个元素从addCh转移到nextCh，即pop()，从而触发另一个goroutine执行注册的事件方法，即run()。

```go
    # k8s.io/client-go/tools/cache/shared_informer.go

    func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
        p.listenersLock.RLock()
        defer p.listenersLock.RUnlock()


        if sync {
            for _, listener := range p.syncingListeners {
                listener.add(obj)
            }
        } else {
            for _, listener := range p.listeners {
                listener.add(obj)
            }
        }
    }


    func (p *processorListener) add(notification interface{}) {
        p.addCh <- notification
    }


    func (p *processorListener) pop() {
        defer utilruntime.HandleCrash()
        defer close(p.nextCh) // Tell .run() to stop


        var nextCh chan<- interface{}
        var notification interface{}
        for {
            select {
            case nextCh <- notification:
                // Notification dispatched
                var ok bool
                notification, ok = p.pendingNotifications.ReadOne()
                if !ok { // Nothing to pop
                    nextCh = nil // Disable this select case
                }
            case notificationToAdd, ok := <-p.addCh:
                if !ok {
                    return
                }
                if notification == nil { // No notification to pop (and pendingNotifications is empty)
                    // Optimize the case - skip adding to pendingNotifications
                    notification = notificationToAdd
                    nextCh = p.nextCh
                } else { // There is already a notification waiting to be dispatched
                    p.pendingNotifications.WriteOne(notificationToAdd)
                }
            }
        }
    }


    func (p *processorListener) run() {
        // this call blocks until the channel is closed.  When a panic happens during the notification
        // we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
        // the next notification will be attempted.  This is usually better than the alternative of never
        // delivering again.
        stopCh := make(chan struct{})
        wait.Until(func() {
            for next := range p.nextCh {
                switch notification := next.(type) {
                case updateNotification:
                    p.handler.OnUpdate(notification.oldObj, notification.newObj)
                case addNotification:
                    p.handler.OnAdd(notification.newObj)
                case deleteNotification:
                    p.handler.OnDelete(notification.oldObj)
                default:
                    utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
                }
            }
            // the only way to get here is if the p.nextCh is empty and closed
            close(stopCh)
        }, 1*time.Second, stopCh)
    }
```
#### Indexer

Indexer是对缓存进行增删查改的接口，缓存本质上就是用map构建的key:value键值对，都存在items这个map中，key为<namespace>/<name>：

```go
    type threadSafeMap struct {
        lock  sync.RWMutex
        items map[string]interface{}
        // indexers maps a name to an IndexFunc
        indexers Indexers
        // indices maps a name to an Index
        indices Indices
    }
```
而为了加速查询，还可以选择性的给这些缓存添加索引，索引存储在indecies中，所谓索引，就是在向缓存中添加记录时，就将其key添加到索引结构中，在查找时，可以根据索引条件，快速查找到指定的key记录，比如默认有个索引是按照namespace进行索引，可以根据快速找出属于某个namespace的某种对象，而不用去遍历所有的缓存。

### 总结

本篇对Kubernetes Informer的使用方法和实现原理，进行了深入分析，整体上看，Informer的设计是相当不错的，基于事件机制，一方面构建本地缓存，一方面触发事件方法，使得控制器能够快速响应和快速获取数据，此外，还有诸如共享Informer, resync, index, watch timeout等机制，使得Informer更加高效和稳定，有了Informer，控制器模式可以说是如虎添翼。 


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

## client-go的DeltaFIFO

```go
    // 代码源自client-go/tools/cache/delta_fifo.go

    // 下面类型出现顺序是为了方便读者理解
    type Delta struct {
        Type   DeltaType       // Delta类型，比如增、减，后面有详细说明
        Object interface{}     // 对象，Delta的粒度是一个对象
    }

    type DeltaType string    // Delta的类型用字符串表达
    const (
        Added   DeltaType = "Added"    // 增加
        Updated DeltaType = "Updated"  // 更新
        Deleted DeltaType = "Deleted"  // 删除
        Sync DeltaType = "Sync"        // 同步
    )

    type Deltas []Delta // Delta数组

    // 就是返回所有的keys。
    type KeyLister interface {
        ListKeys() []string
    }

    // 就是通过key获取对象
    type KeyGetter interface {
        GetByKey(key string) (interface{}, bool, error)
    }

    // 这个接口类型就是上面两个接口类型的组合了
    type KeyListerGetter interface {
        KeyLister
        KeyGetter
    }
```
Delta其实就是kubernetes系统中对象的变化(增、删、改、同步)，FIFO比较好理解，是一个先入先出的队列，那么DeltaFIFO就是一个按序的(先入先出)kubernetes对象变化的队列。

```go
    // 代码源自client-go/tools/cache/fifo.go

    // 这个才是FIFO的抽象，DeltaFIFO只是FIFO的一种实现。
    type Queue interface {
        Store    // 实现了存储接口,这个很好理解，FIFO也是一种存储
        Pop(PopProcessFunc) (interface{}, error) // 在存储的基础上增加了Pop接口，用于弹出对象
        AddIfNotPresent(interface{}) error  // 对象如果不在队列中就添加
        HasSynced() bool  // 通过Replace()放入第一批对象到队列中并且已经被Pop()全部取走
        Close()  // 关闭队列
    }
```
Queue是在Store基础上扩展了Pop接口可以让对象有序的弹出，Indexer是在Store基础上建立了索引，可以快速检索对象。

### DeltaFIFO实现

```go
    // 代码源自client-go/tools/cache/delta_fifo.go

    type DeltaFIFO struct {
        lock sync.RWMutex             // 读写锁，因为涉及到同时读写，读写锁性能要高
        cond sync.Cond                // 给Pop()接口使用，在没有对象的时候可以阻塞，内部锁复用读写锁
        items map[string]Deltas       // 这个应该是Store的本质了，按照kv的方式存储对象，但是存储的是对象的Deltas数组
        queue []string                // 这个是为先入先出实现的，存储的就是对象的键
        populated bool                // 通过Replace()接口将第一批对象放入队列，或者第一次调用增、删、改接口时标记为true
        initialPopulationCount int    // 通过Replace()接口将第一批对象放入队列的对象数量
        keyFunc KeyFunc               // 对象键计算函数
        knownObjects KeyListerGetter  // 前面介绍就是为了这是用，该对象指向的就是Indexer，
        closed     bool               // 是否已经关闭的标记
        closedLock sync.Mutex         // 专为关闭设计的所，为什么不复用读写锁？
    }
```

## client-go的Indexer

## client-go的workqueue
