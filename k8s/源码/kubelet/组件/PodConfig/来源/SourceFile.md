# SourceFile
SourceFile观察静态配置文件路径，在资源清单发生变更后，应用/变更资源对象。对文件系统的变化Watch由`"github.com/fsnotify/fsnotify"`实现。

## 结构体
```go
// pkg/kubelet/config/file.go
type podEventType int

const (
	podAdd podEventType = iota
	podModify
	podDelete

	eventBufferLen = 10
)

type watchEvent struct {
	fileName  string
	eventType podEventType
}

type sourceFile struct {
	path           string
	nodeName       types.NodeName
	period         time.Duration
	store          cache.Store
	fileKeyMapping map[string]string
	updates        chan<- interface{}
	watchEvents    chan *watchEvent
}
```
1. period：定时推送全量数据的周期间隔
2. store：UndeltaStore

## 构造函数
```go
// pkg/kubelet/config/file.go
func NewSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) {
	// "github.com/sigma/go-inotify" requires a path without trailing "/"
	path = strings.TrimRight(path, string(os.PathSeparator))

	config := newSourceFile(path, nodeName, period, updates)
	klog.V(1).InfoS("Watching path", "path", path)
	// 开始循环
	config.run()
}

func newSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) *sourceFile {
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		// 将全部数据以SET的类型发送到updates中
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
	}
	// UndeltaStore每次被进行增删改查时，将会List出全部数据执行send函数
	store := cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc)
	return &sourceFile{
		path:           path,
		nodeName:       nodeName,
		period:         period,
		store:          store,
		fileKeyMapping: map[string]string{},
		updates:        updates,
		watchEvents:    make(chan *watchEvent, eventBufferLen),
	}
}
```

## run
```go
// pkg/kubelet/config/file.go
func (s *sourceFile) run() {
	listTicker := time.NewTicker(s.period)

	go func() {
		// Read path immediately to speed up startup.
		// 立即推送一次全量数据
		if err := s.listConfig(); err != nil {
			klog.ErrorS(err, "Unable to read config path", "path", s.path)
		}
		for {
			select {
			case <-listTicker.C:
				// 定时推送全量数据
				if err := s.listConfig(); err != nil {
					klog.ErrorS(err, "Unable to read config path", "path", s.path)
				}
				// 监听s.watchEvents channel，处理watchEvent
			case e := <-s.watchEvents:
				if err := s.consumeWatchEvent(e); err != nil {
					klog.ErrorS(err, "Unable to process watch event")
				}
			}
		}
	}()

	// 开始监听文件是否产生变化，将变化包装为watchEvent，生产进s.watchEvents channel中
	s.startWatch()
}
```

## startWatch
linux环境编译的函数
```go
// pkg/kubelet/config/file_linux.go
const (
	retryPeriod    = 1 * time.Second
	maxRetryPeriod = 20 * time.Second
)

func (s *sourceFile) startWatch() {
	backOff := flowcontrol.NewBackOff(retryPeriod, maxRetryPeriod)
	backOffID := "watch"

	// 每秒执行一次循环，该循环主要作用是重试doWatch
	go wait.Forever(func() {
		// 如果出现错误就使循环时间逐渐放大，直到maxRetryPeriod
		if backOff.IsInBackOffSinceUpdate(backOffID, time.Now()) {
			return
		}

		if err := s.doWatch(); err != nil {
			klog.ErrorS(err, "Unable to read config path", "path", s.path)
			if _, retryable := err.(*retryableError); !retryable {
				// 如果出现错误就使循环时间逐渐放大，直到maxRetryPeriod
				backOff.Next(backOffID, time.Now())
			}
		}
	}, retryPeriod)
}

// 使用fsnotify组件来监控文件系统变化
func (s *sourceFile) doWatch() error {
	_, err := os.Stat(s.path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return &retryableError{"path does not exist, ignoring"}
	}

	// 新建fsnotify Watcher
	w, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("unable to create inotify: %v", err)
	}
	defer w.Close()

	err = w.Add(s.path)
	if err != nil {
		return fmt.Errorf("unable to create inotify for path %q: %v", s.path, err)
	}

	for {
		select {
		case event := <-w.Events:
			// 获取并消费fsnotify事件
			if err = s.produceWatchEvent(&event); err != nil {
				return fmt.Errorf("error while processing inotify event (%+v): %v", event, err)
			}
		case err = <-w.Errors:
			// 出现了Error直接return
			return fmt.Errorf("error while watching %q: %v", s.path, err)
		}
	}
}
```

## listConfig
listConfig会立即解析文件(夹)，并推送全量的数据到updates channel中
```go
// pkg/kubelet/config/file.go
func (s *sourceFile) listConfig() error {
	path := s.path
	statInfo, err := os.Stat(path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		// 路径如果存在但是有其他错误，则发出一个空消息，让podConfig知道fileSource正常工作了
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return fmt.Errorf("path does not exist, ignoring")
	}

	switch {
	case statInfo.Mode().IsDir():
		pods, err := s.extractFromDir(path)
		if err != nil {
			return err
		}
		if len(pods) == 0 {
			// Emit an update with an empty PodList to allow FileSource to be marked as seen
			s.updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
			return nil
		}
		// 触发一次推送
		return s.replaceStore(pods...)

	case statInfo.Mode().IsRegular():
		pod, err := s.extractFromFile(path)
		if err != nil {
			return err
		}
		return s.replaceStore(pod)

	default:
		return fmt.Errorf("path is not a directory or file")
	}
}
```

## extractFromFile
解析文件，生成pod，保存到store中
```go
// pkg/kubelet/config/file.go
func (s *sourceFile) extractFromFile(filename string) (pod *v1.Pod, err error) {
	klog.V(3).InfoS("Reading config file", "path", filename)
	defer func() {
		if err == nil && pod != nil {
			objKey, keyErr := cache.MetaNamespaceKeyFunc(pod)
			if keyErr != nil {
				err = keyErr
				return
			}
			s.fileKeyMapping[filename] = objKey
		}
	}()

	file, err := os.Open(filename)
	if err != nil {
		return pod, err
	}
	defer file.Close()

	data, err := utilio.ReadAtMost(file, maxConfigLength)
	if err != nil {
		return pod, err
	}

	defaultFn := func(pod *api.Pod) error {
		return s.applyDefaults(pod, filename)
	}

	parsed, pod, podErr := tryDecodeSinglePod(data, defaultFn)
	if parsed {
		if podErr != nil {
			return pod, podErr
		}
		return pod, nil
	}

	return pod, fmt.Errorf("%v: couldn't parse as pod(%v), please check config file", filename, podErr)
}
```

## produceWatchEvent
watchEvent的生产者
```go
// pkg/kubelet/config/file_linux.go
func (s *sourceFile) produceWatchEvent(e *fsnotify.Event) error {
	// Ignore file start with dots
	// 忽略隐藏文件
	if strings.HasPrefix(filepath.Base(e.Name), ".") {
		klog.V(4).InfoS("Ignored pod manifest, because it starts with dots", "eventName", e.Name)
		return nil
	}
	var eventType podEventType
	// 将fsnotify事件转化为watchEvent
	switch {
	case (e.Op & fsnotify.Create) > 0:
		eventType = podAdd
	case (e.Op & fsnotify.Write) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Chmod) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Remove) > 0:
		eventType = podDelete
	case (e.Op & fsnotify.Rename) > 0:
		eventType = podDelete
	default:
		// Ignore rest events
		return nil
	}

	s.watchEvents <- &watchEvent{e.Name, eventType}
	return nil
}
```

## consumeWatchEvent
watchEvent事件的消费者
```go
// pkg/kubelet/config/file_linux.go
func (s *sourceFile) consumeWatchEvent(e *watchEvent) error {
	switch e.eventType {
	case podAdd, podModify:
		// 如果是podAdd或者podModify事件，只需要向store中添加pod即可
		pod, err := s.extractFromFile(e.fileName)
		if err != nil {
			return fmt.Errorf("can't process config file %q: %v", e.fileName, err)
		}
		return s.store.Add(pod)
	case podDelete:
		// 如果是删除事件，执行store的Delete方法
		if objKey, keyExist := s.fileKeyMapping[e.fileName]; keyExist {
			pod, podExist, err := s.store.GetByKey(objKey)
			if err != nil {
				return err
			} else if !podExist {
				return fmt.Errorf("the pod with key %s doesn't exist in cache", objKey)
			} else {
				if err = s.store.Delete(pod); err != nil {
					return fmt.Errorf("failed to remove deleted pod from cache: %v", err)
				}
				delete(s.fileKeyMapping, e.fileName)
			}
		}
	}
	return nil
}
```