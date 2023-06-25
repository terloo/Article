# Mux
从Mux可以获取一个只写channel，向该channel中写入的所有数据都会被调用merger.Merge()进行处理

## 实现类
```go
// pkg/util/config/config.go
func NewMux(merger Merger) *Mux {
	mux := &Mux{
		sources: make(map[string]chan interface{}),
		merger:  merger,
	}
	return mux
}
```

## Channel
Channel方法传入一个来源名字，返回该名字对应的channel，传入此channel的所有改动，都会被Mux调用Merge进行处理
```go
// pkg/util/config/config.go
func (m *Mux) Channel(source string) chan interface{} {
	if len(source) == 0 {
		panic("Channel given an empty name")
	}
	m.sourceLock.Lock()
	defer m.sourceLock.Unlock()
	channel, exists := m.sources[source]
	if exists {
		return channel
	}
    // 如果没有已存在的channel，则新建
	newChannel := make(chan interface{})
	m.sources[source] = newChannel

    // 将新建的channel
	go wait.Until(func() { m.listen(source, newChannel) }, 0, wait.NeverStop)
	return newChannel
}

func (m *Mux) listen(source string, listenChannel <-chan interface{}) {
    // 消费来源channel，调用Merge进行处理
	for update := range listenChannel {
		m.merger.Merge(source, update)
	}
}
```