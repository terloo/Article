# Informer
Informer是client-go的客户端缓存机制，客户端可以使用Informer与api-server进行实时全量或增量同步，得k8s的所有组件在不依赖中间件的情况下，保证了消息的实时性、可靠性、顺序性等。

## 核心组件
1. Reflector：Reflector用于监控(Watch)指定的k8s资源，当监控的资源发生变化时，触发相应的事件。例如Added、Updated、Deleted，并将其资源对象存放在本地缓存DeltaFIFO中。
2. DeltaFIFO：FIFO是一个先进先出队列，拥有Add、Update、Delete、List、Pop、Close等方法。Delta是一个资源对象的存储，可以保存资源对象的操作类型。例如Added、Updated、Deleted、Sync等。
3. Indexer：Indexer是client-go用来存储资源对象并自带索引功能的本地存储。Reflector从DeltaFIFO中消费出来的存储对象保存至Indexer。Indexer与ETCD中资源完全一致。client-go可以从Indexer中获取资源对象，而无需同api-server(和ETCD)交互

## SharedInformerFactory
k8s中资源的Informer可以通过SharedInformerFactory进行获取。  
SharedInformerFactory会缓存同一种资源的Informer，使其共享一个Reflecter，节省资源。否则会运行过多的ListAndWatch方法，给api-server造成压力
```go
// 创建一个共享Informer工厂，第一个参数是k8s客户端，第二个参数是重新同步时间，为0时禁用该功能
informerFactory := informers.NewSharedInformerFactory(clientSet, time.Minute * 0)
// 通过工厂来创建资源的Informer
informer := informerFactory.Core().V1().Pods().Informer()

// vendor/k8s.io/client-go/informers/factory.go
type sharedInformerFactory struct {
    // 缓存map
	informers map[reflect.Type]cache.SharedIndexInformer
}

func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
    // 如果能找到informer，则直接返回
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

    // 找不到再调用函数创建
	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```

## 资源Informer
每一个k8s资源上都实现了Informer机制。每一个Informer上都会实现Informer和Lister
```go
// vendor/k8s.io/client-go/informers/core/v1/pod.go
type PodInformer interface {
	// 用于构建出Informer
	Informer() cache.SharedIndexInformer
	// 用于获取Indexer
	Lister() v1.PodLister
}

// 实现类
type podInformer struct {
    // Informer工厂，Informer()方法通过该工厂的InformerFor()来获取informer
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}
```


## Informer添加Handle过程
1. 将handle函数包装成processListener
2. 将processListener添加到informer的process字段中

## Informer启动过程
1. 新建一个DeltaFIFO
   1. New一个DeltaFIFO
   2. 将DeltaFIFO包装到controller中
   3. 将controller设置informer中
2. 启动controller
   1. New一个Reflect，ListAndWatch、DeltaFIFO使用informer中的ListAndWatch和DeltaFIFO
   2. 启动Reflect，Reflect开始执行ListAndWatch
      1. ListAndWatch将数据传输到DeltaFIFO
   3. 启动controller
      1. controller从DeltaFIFO中消费数据，调用informer的HandleDelta方法进行事件分发