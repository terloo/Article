# data
Heap的底层实现，该数据结构实现了go标准库中的`container/heap`接口，且data的方法仅用于实现`container/heap`，只能通过`container/heap`中提供的函数进行调用  
`container/heap`中提供的函数会将data中的元素升序排列，优先弹出队列最前方(即最小的值)

## 结构体
```go
// pkg/scheduler/internal/heap/heap.go
// data is an internal struct that implements the standard heap interface
// and keeps the data stored in the heap.
type data struct {
	// items is a map from key of the objects to the objects and their index.
	// We depend on the property that items in the map are in the queue and vice versa.
	items map[string]*heapItem
	// queue implements a heap data structure and keeps the order of elements
	// according to the heap invariant. The queue keeps the keys of objects stored
	// in "items".
	queue []string

	// keyFunc is used to make the key used for queued item insertion and retrieval, and
	// should be deterministic.
	keyFunc KeyFunc
	// lessFunc is used to compare two objects in the heap.
	lessFunc lessFunc
}

type heapItem struct {
    // 保存的元素
	obj   interface{} // The object which is stored in the heap.
    // 对象在queue中的索引位置
	index int         // The index of the object's key in the Heap.queue.
}

type KeyFunc func(obj interface{}) (string, error)
type lessFunc = func(item1, item2 interface{}) bool
```
1. items：保存对象的映射，映射的key是由keyFunc计算的key
2. queue：切片，保存所有对象的key
3. lessFunc：用于对象比较

## 方法
data的方法均为标准库`container/heap`接口的实现，不能被直接调用，只能通过`container/heap`中提供的操作heap的函数进行调用
```go
// Less compares two objects and returns true if the first one should go
// in front of the second one in the heap.
func (h *data) Less(i, j int) bool {
	if i > len(h.queue) || j > len(h.queue) {
		return false
	}
	itemi, ok := h.items[h.queue[i]]
	if !ok {
		return false
	}
	itemj, ok := h.items[h.queue[j]]
	if !ok {
		return false
	}
    // 调用lessFunc进行比较
	return h.lessFunc(itemi.obj, itemj.obj)
}

// Len returns the number of items in the Heap.
func (h *data) Len() int { return len(h.queue) }

// Swap implements swapping of two elements in the heap. This is a part of standard
// heap interface and should never be called directly.
func (h *data) Swap(i, j int) {
    // 交换queue中元素位置
	h.queue[i], h.queue[j] = h.queue[j], h.queue[i]
    // 然后更新元素中索引值
	item := h.items[h.queue[i]]
	item.index = i
	item = h.items[h.queue[j]]
	item.index = j
}

// Push is supposed to be called by heap.Push only.
func (h *data) Push(kv interface{}) {
	keyValue := kv.(*itemKeyValue)
	n := len(h.queue)
	h.items[keyValue.key] = &heapItem{keyValue.obj, n}
	h.queue = append(h.queue, keyValue.key)
}

// 注意此处弹出的是队列最后一个元素(即最大的元素，该元素作为Less第一个参数时应该始终返回true)
// Pop is supposed to be called by heap.Pop only.
func (h *data) Pop() interface{} {
	key := h.queue[len(h.queue)-1]
	h.queue = h.queue[0 : len(h.queue)-1]
	item, ok := h.items[key]
	if !ok {
		// This is an error
		return nil
	}
	delete(h.items, key)
	return item.obj
}

// Peek is supposed to be called by heap.Peek only.
func (h *data) Peek() interface{} {
	if len(h.queue) > 0 {
		return h.items[h.queue[0]].obj
	}
	return nil
}
```