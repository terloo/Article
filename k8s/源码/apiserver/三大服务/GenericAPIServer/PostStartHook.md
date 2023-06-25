# PostStartHook
PostStartHook是GenericAPIServer保存的，在开始监听端口之后，执行的一些钩子函数。  
每个钩子函数都会被GenericAPIServer用goroutine进行执行，不严格保证顺序

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/server/genericapiserver.go
// 在GenericAPIServer用于保存postStartHooks的字段
type GenericAPIServer struct {
    // ...

    // 保护map并发操作的锁
	postStartHookLock      sync.Mutex
    // name -> postStartHookEntry的映射，name用于在Hook出错时进行报告
	postStartHooks         map[string]postStartHookEntry
    // 记录postStartHook是否均执行
	postStartHooksCalled   bool
    // 记录需要进行运行的Hook name
	disabledPostStartHooks sets.String

    // ...
}

// vendor/k8s.io/apiserver/pkg/server/hooks.go
// PostStartHook的函数定义，参数为PostStartHookContext
type PostStartHookFunc func(context PostStartHookContext) error

// PostStartHookContext保存了GenericAPIServer相关的一些配置供钩子函数使用
type PostStartHookContext struct {
	// LoopbackClientConfig is a config for a privileged loopback connection to the API server
	LoopbackClientConfig *restclient.Config
    // StopCh会在GenericAPIServer在关闭时收到消息
	StopCh <-chan struct{}
}

// postStartHook实际保存时的结构体
type postStartHookEntry struct {
	hook PostStartHookFunc
	// originatingStack holds the stack that registered postStartHooks. This allows us to show a more helpful message
	// for duplicate registration.
    // 保存了注册Hook时的调用栈
	originatingStack string

	// done will be closed when the postHook is finished
	done chan struct{}
}

// 一个接口，用于提供一个PostStartHookFunc
type PostStartHookProvider interface {
	PostStartHook() (string, PostStartHookFunc, error)
}
```

## 委托Hook
在构造GenericAPIServer，会将DelegationTarget里面的所有Hook遍历出来添加到该GenericAPIServer中。这也就意味着AggregatorServer会持有所有的Hook，并将其全部运行。
```go
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {

	// ...

	// first add poststarthooks from delegated targets
	// 添加所有DelegationTarget的Hook
	for k, v := range delegationTarget.PostStartHooks() {
		s.postStartHooks[k] = v
	}

	for k, v := range delegationTarget.PreShutdownHooks() {
		s.preShutdownHooks[k] = v
	}

	// add poststarthooks that were preconfigured.  Using the add method will give us an error if the same name has already been registered.
	// 添加所有配置中的Hook
	for name, preconfiguredPostStartHook := range c.PostStartHooks {
		if err := s.AddPostStartHook(name, preconfiguredPostStartHook.hook); err != nil {
			return nil, err
		}
	}

	// ...
}
```

## 添加Hook
```go
// vendor/k8s.io/apiserver/pkg/server/hooks.go
// *GenericAPIServer的一个方法，用于向GenericAPIServer中添加一个PostStartHookFunc
func (s *GenericAPIServer) AddPostStartHook(name string, hook PostStartHookFunc) error {
	if len(name) == 0 {
		return fmt.Errorf("missing name")
	}
	if hook == nil {
		return fmt.Errorf("hook func may not be nil: %q", name)
	}
    // 如果Hook被禁用，则拒绝注册
	if s.disabledPostStartHooks.Has(name) {
		klog.V(1).Infof("skipping %q because it was explicitly disabled", name)
		return nil
	}

	s.postStartHookLock.Lock()
	defer s.postStartHookLock.Unlock()

    // 如果Hook阶段已经被执行活了，则不允许添加钩子函数
	if s.postStartHooksCalled {
		return fmt.Errorf("unable to add %q because PostStartHooks have already been called", name)
	}
	if postStartHook, exists := s.postStartHooks[name]; exists {
		// this is programmer error, but it can be hard to debug
		return fmt.Errorf("unable to add %q because it was already registered by: %s", name, postStartHook.originatingStack)
	}

	// done is closed when the poststarthook is finished.  This is used by the health check to be able to indicate
	// that the poststarthook is finished
	done := make(chan struct{})
    // 向GenericAPIServer中添加关于此hook的健康检查
	if err := s.AddBootSequenceHealthChecks(postStartHookHealthz{name: "poststarthook/" + name, done: done}); err != nil {
		return err
	}
    // 把钩子函数包装为Entry，并设置到GenericAPIServer的map中
	s.postStartHooks[name] = postStartHookEntry{hook: hook, originatingStack: string(debug.Stack()), done: done}

	return nil
}
```

## 运行Hook
```go
// vendor/k8s.io/apiserver/pkg/server/hooks.go
func (s *GenericAPIServer) RunPostStartHooks(stopCh <-chan struct{}) {
	s.postStartHookLock.Lock()
	defer s.postStartHookLock.Unlock()
	s.postStartHooksCalled = true

	context := PostStartHookContext{
		LoopbackClientConfig: s.LoopbackClientConfig,
		StopCh:               stopCh,
	}

    // 每个Hook都使用goroutine进行运行
	for hookName, hookEntry := range s.postStartHooks {
		go runPostStartHook(hookName, hookEntry, context)
	}
}

func runPostStartHook(name string, entry postStartHookEntry, context PostStartHookContext) {
	var err error
	func() {
        // hook意外出现panic时不退出服务器
		// don't let the hook *accidentally* panic and kill the server
		defer utilruntime.HandleCrash()
		err = entry.hook(context)
	}()
    // 如果Hook出现err，说明是有意关闭服务
	// if the hook intentionally wants to kill server, let it.
	if err != nil {
		klog.Fatalf("PostStartHook %q failed: %v", name, err)
	}
	close(entry.done)
}
```

## 示例
```go
// cmd/kube-apiserver/app/aggregator.go
// PostStartHook的示例，用于轮询CRD，将其转化为aggregator的apiservice对象
err = aggregatorServer.GenericAPIServer.AddPostStartHook("kube-apiserver-autoregistration", func(context genericapiserver.PostStartHookContext) error {
    go crdRegistrationController.Run(5, context.StopCh)
    go func() {
        // let the CRD controller process the initial set of CRDs before starting the autoregistration controller.
        // this prevents the autoregistration controller's initial sync from deleting APIServices for CRDs that still exist.
        // we only need to do this if CRDs are enabled on this server.  We can't use discovery because we are the source for discovery.
        if aggregatorConfig.GenericConfig.MergedResourceConfig.AnyVersionForGroupEnabled("apiextensions.k8s.io") {
            crdRegistrationController.WaitForInitialSync()
        }
        autoRegistrationController.Run(5, context.StopCh)
    }()
    return nil
})
```