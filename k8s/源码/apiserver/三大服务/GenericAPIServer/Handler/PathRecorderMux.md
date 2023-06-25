# PathRecorderMux
PathRecorderMux在restful.Container无法处理本次请求时，对本次请求进行处理

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/server/mux/pathrecorder.go
type PathRecorderMux struct {
	// name is used for logging so you can trace requests through
	name string

	lock            sync.Mutex
	notFoundHandler http.Handler
	pathToHandler   map[string]http.Handler
	prefixToHandler map[string]http.Handler

	// mux stores a pathHandler and is used to handle the actual serving.
	// Turns out, we want to accept trailing slashes, BUT we don't care about handling
	// everything under them.  This does exactly matches only unless its explicitly requested to
	// do something different
	mux atomic.Value

	// exposedPaths is the list of paths that should be shown at /
	exposedPaths []string

	// pathStacks holds the stacks of all registered paths.  This allows us to show a more helpful message
	// before the "http: multiple registrations for %s" panic.
	pathStacks map[string]string
}
```
1. mux：atomic.Value是go标准库提供的原子类，用于对其中保存的Value进行原子性的读取和存储。在PathRecorderMux中，保存的Value即是`pathHandler`。
2. lock、notFoundHandler、pathToHandler、perfixToHandler：与mux中保存的数据一致。作为更新mux时的暂存区，每次调用`refreshMuxLocked()`方法时将会全量复制一份数据到mux

## pathHandler
```go
// vendor/k8s.io/apiserver/pkg/server/mux/pathrecorder.go
// pathHandler存储在PathRecorderMux的mux字段中，通过原子操作进行更新
type pathHandler struct {
	// muxName is used for logging so you can trace requests through
	muxName string

	// pathToHandler is a map of exactly matching request to its handler
	pathToHandler map[string]http.Handler

	// this has to be sorted by most slashes then by length
	prefixHandlers []prefixHandler

	// notFoundHandler is the handler to use for satisfying requests with no other match
	notFoundHandler http.Handler
}

```
1. muxName：分发器的名字
2. pathToHandler：url路径到某个handler的映射
3. prefixHandlers：url路径前缀到某个handler的映射
4. notFoundHandler：没有路径匹配时的handler

## pathHandler.ServeHTTP
```go
// vendor/k8s.io/apiserver/pkg/mux/pathrecorder.go
// pathHandler实现了http.Handler接口，对接收到的请求进行分发
func (h *pathHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if exactHandler, ok := h.pathToHandler[r.URL.Path]; ok {
		klog.V(5).Infof("%v: %q satisfied by exact match", h.muxName, r.URL.Path)
		exactHandler.ServeHTTP(w, r)
		return
	}

	for _, prefixHandler := range h.prefixHandlers {
		if strings.HasPrefix(r.URL.Path, prefixHandler.prefix) {
			klog.V(5).Infof("%v: %q satisfied by prefix %v", h.muxName, r.URL.Path, prefixHandler.prefix)
			prefixHandler.handler.ServeHTTP(w, r)
			return
		}
	}

	klog.V(5).Infof("%v: %q satisfied by NotFoundHandler", h.muxName, r.URL.Path)
	h.notFoundHandler.ServeHTTP(w, r)
}
```

## 构造方法
```go
// vendor/k8s.io/apiserver/pkg/server/mux/pathrecorder.go
func NewPathRecorderMux(name string) *PathRecorderMux {
	ret := &PathRecorderMux{
		name:            name,
		pathToHandler:   map[string]http.Handler{},
		prefixToHandler: map[string]http.Handler{},
		mux:             atomic.Value{},
		exposedPaths:    []string{},
		pathStacks:      map[string]string{},
	}

    // 通过Strore将pathHandler存储到mux中
	ret.mux.Store(&pathHandler{notFoundHandler: http.NotFoundHandler()})
	return ret
}
```

## 向PathRecorderMux中注册Handler
PathRecorderMux提供了六个方法向其中注册Handler，分别为`Handle`、`HandleFunc`、`UnlistHandle`、`UnlistHandleFunc`、`HandlePrefix`、`UnlistHandlePrefix`
```go
// vendor/k8s.io/apiserver/pkg/server/pathrecorder.go
func (m *PathRecorderMux) Handle(path string, handler http.Handler) {
    // 加锁，保证操作map的安全并发
	m.lock.Lock()
	defer m.lock.Unlock()
	m.trackCallers(path)

	// 将其添加到exposedPaths
	m.exposedPaths = append(m.exposedPaths, path)
    // 向暂存区pathToHandler中添加一个handler
	m.pathToHandler[path] = handler
    // 更新mux
	m.refreshMuxLocked()
}

// HandleFunc是Handle的易用方法
func (m *PathRecorderMux) HandleFunc(path string, handler func(http.ResponseWriter, *http.Request)) {
	m.Handle(path, http.HandlerFunc(handler))
}

func (m *PathRecorderMux) UnlistedHandle(path string, handler http.Handler) {
	m.lock.Lock()
	defer m.lock.Unlock()
	m.trackCallers(path)

	// 不将其添加到exposedPaths
	m.pathToHandler[path] = handler
	m.refreshMuxLocked()
}

// UnlistedHandleFunc是UnlistedHandle的易用方法
func (m *PathRecorderMux) UnlistedHandleFunc(path string, handler func(http.ResponseWriter, *http.Request)) {
	m.UnlistedHandle(path, http.HandlerFunc(handler))
}

// HandlePrefix is like Handle, but matches for anything under the path.  Like a standard golang trailing slash.
func (m *PathRecorderMux) HandlePrefix(path string, handler http.Handler) {
	if !strings.HasSuffix(path, "/") {
		panic(fmt.Sprintf("%q must end in a trailing slash", path))
	}

	m.lock.Lock()
	defer m.lock.Unlock()
	m.trackCallers(path)

	// 将其添加到exposedPaths
	m.exposedPaths = append(m.exposedPaths, path)
	m.prefixToHandler[path] = handler
	m.refreshMuxLocked()
}

// UnlistedHandlePrefix is like UnlistedHandle, but matches for anything under the path.  Like a standard golang trailing slash.
func (m *PathRecorderMux) UnlistedHandlePrefix(path string, handler http.Handler) {
	if !strings.HasSuffix(path, "/") {
		panic(fmt.Sprintf("%q must end in a trailing slash", path))
	}

	m.lock.Lock()
	defer m.lock.Unlock()
	m.trackCallers(path)

	// 不将其添加到exposedPaths
	m.prefixToHandler[path] = handler
	m.refreshMuxLocked()
}
```

## PathRecorderMux.ServeHTTP
```go
// vendor/k8s.io/apiserver/pkg/server/mux/pathrecorder.go
func (m *PathRecorderMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 直接交由mux进行处理
	m.mux.Load().(*pathHandler).ServeHTTP(w, r)
}

// ServeHTTP makes it an http.Handler
func (h *pathHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 先进行精准匹配，即从pathToHandler中获取Handler
	if exactHandler, ok := h.pathToHandler[r.URL.Path]; ok {
		klog.V(5).Infof("%v: %q satisfied by exact match", h.muxName, r.URL.Path)
		exactHandler.ServeHTTP(w, r)
		return
	}

    // 再进行前缀匹配，即从prefixHandlers中获取Handler
	for _, prefixHandler := range h.prefixHandlers {
		if strings.HasPrefix(r.URL.Path, prefixHandler.prefix) {
			klog.V(5).Infof("%v: %q satisfied by prefix %v", h.muxName, r.URL.Path, prefixHandler.prefix)
			prefixHandler.handler.ServeHTTP(w, r)
			return
		}
	}

    // 没有找到则发送到notFoundHandler，返回404
	klog.V(5).Infof("%v: %q satisfied by NotFoundHandler", h.muxName, r.URL.Path)
	h.notFoundHandler.ServeHTTP(w, r)
}
```