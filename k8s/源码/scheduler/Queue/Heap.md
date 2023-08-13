# Heap
Heap是scheduler中用于实现优先级队列的数据结构，非线程安全

## 结构体
```go
// pkg/scheduler/internale/heap/heap.go
// Heap is a producer/consumer queue that implements a heap data structure.
// It can be used to implement priority queues and similar data structures.
type Heap struct {
	// data stores objects and has a queue that keeps their ordering according
	// to the heap invariant.
	data *data
	// metricRecorder updates the counter when elements of a heap get added or
	// removed, and it does nothing if it's nil
	metricRecorder metrics.MetricRecorder
}
```
1. data：实现了`container/heap`接口的底层结构体
2. metricRecorder：保存指标，如果为nil，将不会做任何事

## 构造函数
```go
// pkg/scheduler/internal/heap/heap.go
// New returns a Heap which can be used to queue up items to process.
func New(keyFn KeyFunc, lessFn lessFunc) *Heap {
	return NewWithRecorder(keyFn, lessFn, nil)
}

// 直接构造底层data
// NewWithRecorder wraps an optional metricRecorder to compose a Heap object.
func NewWithRecorder(keyFn KeyFunc, lessFn lessFunc, metricRecorder metrics.MetricRecorder) *Heap {
	return &Heap{
		data: &data{
			items:    map[string]*heapItem{},
			queue:    []string{},
			keyFunc:  keyFn,
			lessFunc: lessFn,
		},
		metricRecorder: metricRecorder,
	}
}
```

## Heap增删改查
```go
// pkg/scheduler/internal/heap/heap.go
// Add inserts an item, and puts it in the queue. The item is updated if it
// already exists.
func (h *Heap) Add(obj interface{}) error {
	key, err := h.data.keyFunc(obj)
	if err != nil {
		return cache.KeyError{Obj: obj, Err: err}
	}
	if _, exists := h.data.items[key]; exists {
        // 如果存在即更新，然后把heap重新排序
		h.data.items[key].obj = obj
		heap.Fix(h.data, h.data.items[key].index)
	} else {
        // 不存在即添加
		heap.Push(h.data, &itemKeyValue{key, obj})
		if h.metricRecorder != nil {
			h.metricRecorder.Inc()
		}
	}
	return nil
}

// AddIfNotPresent inserts an item, and puts it in the queue. If an item with
// the key is present in the map, no changes is made to the item.
func (h *Heap) AddIfNotPresent(obj interface{}) error {
	key, err := h.data.keyFunc(obj)
	if err != nil {
		return cache.KeyError{Obj: obj, Err: err}
	}
	if _, exists := h.data.items[key]; !exists {
		heap.Push(h.data, &itemKeyValue{key, obj})
		if h.metricRecorder != nil {
			h.metricRecorder.Inc()
		}
	}
	return nil
}

// Update is the same as Add in this implementation. When the item does not
// exist, it is added.
func (h *Heap) Update(obj interface{}) error {
	return h.Add(obj)
}

// Delete removes an item.
func (h *Heap) Delete(obj interface{}) error {
	key, err := h.data.keyFunc(obj)
	if err != nil {
		return cache.KeyError{Obj: obj, Err: err}
	}
	if item, ok := h.data.items[key]; ok {
		heap.Remove(h.data, item.index)
		if h.metricRecorder != nil {
			h.metricRecorder.Dec()
		}
		return nil
	}
	return fmt.Errorf("object not found")
}

// Peek returns the head of the heap without removing it.
func (h *Heap) Peek() interface{} {
	return h.data.Peek()
}

// Pop returns the head of the heap and removes it.
func (h *Heap) Pop() (interface{}, error) {
	obj := heap.Pop(h.data)
	if obj != nil {
		if h.metricRecorder != nil {
			h.metricRecorder.Dec()
		}
		return obj, nil
	}
	return nil, fmt.Errorf("object was removed from heap data")
}

// Get returns the requested item, or sets exists=false.
func (h *Heap) Get(obj interface{}) (interface{}, bool, error) {
	key, err := h.data.keyFunc(obj)
	if err != nil {
		return nil, false, cache.KeyError{Obj: obj, Err: err}
	}
	return h.GetByKey(key)
}

// GetByKey returns the requested item, or sets exists=false.
func (h *Heap) GetByKey(key string) (interface{}, bool, error) {
	item, exists := h.data.items[key]
	if !exists {
		return nil, false, nil
	}
	return item.obj, true, nil
}

// List returns a list of all the items.
func (h *Heap) List() []interface{} {
	list := make([]interface{}, 0, len(h.data.items))
	for _, item := range h.data.items {
		list = append(list, item.obj)
	}
	return list
}

// Len returns the number of items in the heap.
func (h *Heap) Len() int {
	return len(h.data.queue)
}
```