# Indexer
Indexer是client-go用来存储资源对象的本地存储，Reflector将从DeltaFIFO中消费出来的资源对象存储至Indexer中。  
Indexer的数据与Etcd中存储的数据完全一致。Indexer是个接口，由cache.cache结构体进行实现

## Indexer接口
```go
// vendor/k8s.io/client-go/tools/cache/index.go
type Indexer interface {
	// 继承Store的增删改查方法
	Store
	// 增加了一些与索引器有关的方法
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexedValue string) ([]string, error)
	ListIndexFuncValues(indexName string) []string
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	GetIndexers() Indexers
	AddIndexers(newIndexers Indexers) error
}

// vendor/k8s.io/client-go/tools/cache/store.go
type Store interface {
	// 普通的增删查改和重同步接口
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)
	Replace([]interface{}, string) error
	Resync() error
}
```

## cache.cache结构体
```go
// vendor/k8s.io/client-go/tools/cache/store.go
// cache实现了Indexer接口，存储功能主要使用ThreadSafeStore属性进行实现
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	// 一个线程安全的存储
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// should be deterministic.
	// 通过资源对象来计算其索引用的key的函数
	keyFunc KeyFunc
}

// 通过资源对象来计算其索引用的key的函数
type KeyFunc func(obj interface{}) (string, error)
```

## ThreadSafeMap
实现了并发安全的存储，Indexer就是在ThreadSafeMap的基础上进行封装，继承了与ThreadSafeMap相关的操作并实现了Indexer Func等功能，  
例如Index、IndexKeys、GetIndexers等。

### 接口
```go
// vendor/k8s.io/client-go/tools/cache/thread_safe_store.go
type ThreadSafeStore interface {
	// 与Store不同，ThreadSafeStore的增删改查都是使用一个索引作为参数进行操作的
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	// Resync is a no-op and is deprecated
	Resync() error
}
```

### 结构体
```go
// vendor/k8s.io/client-go/toos/cache/thread_safe_store.go
type threadSafeMap struct {
	lock  sync.RWMutex
    // 存储资源对象数据，key通过KeyFunc计算获取。
    // keyFunc默认使用MetaNamespaceKeyFunc函数，计算结果为<namespace>/<name>，没有命名空间则为<name>
	items map[string]interface{}

	// 一个key -> KeyFunc的映射，同一个Object通过不同的KeyFunc可以计算出不同的key
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}

// Indexers maps a name to an IndexFunc
type Indexers map[string]IndexFunc

// Indices maps a name to an Index
type Indices map[string]Index

// IndexFunc knows how to compute the set of indexed values for an object.
type IndexFunc func(obj interface{}) ([]string, error)

// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String
```
1. items：资源对象的key到资源对象的映射，key同样通过一个KeyFunc计算，一般为cache.MetaNamespaceKeyFunc。
2. IndexFunc：一个函数，传入一个资源对象，计算出该对象所使用的索引key值，可以是多个key。
3. Indexers：一个key -> KeyFunc的映射，同一个资源对象通过不同的KeyFunc计算可以得出不一样的keyFunc。
4. sets.String：k8s内部实现的stringSet，结构为map[string]struct{}
5. Index：一个 key -> stringSet的映射
6. Indices：一个 key -> Index的映射。与Indexers的key是同一个key

### 数据结构说明
1. Indexers：key可以理解为用户在缓存结构中查找一个资源对象时所使用的查找方法名(KeyFunc对应的名字)，这个查找方式是用户自定义并在初始化缓存结构时传入的。
2. Indices：key为查找方法名(KeyFunc对应的名字)，value为Index
3. Index：key可以理解资源对象通过该查找方式(KeyFunc)所计算出的key，value为资源对象在items中存储时所使用的key的集合

## Indexer的存储逻辑

### 存储
```go
// vendor/k8s.io/client-go/tools/cache/store.go
// cache计算出资源对象对应的key后，存储到自己的cacheStorage(ThreadSafeMap)中
func (c *cache) Add(obj interface{}) error {
	// 通过初始化时传入的KeyFunc来计算资源对象的key，该key是items所使用的key
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Add(key, obj)
	return nil
}

// vendor/k8s.io/client-go/tools/cache/thread_safe_store.go
// threadSafeMap线程安全的取出旧Object，并将新Object设置进去，最后更新Indices
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	oldObject := c.items[key]
	c.items[key] = obj
	c.updateIndices(oldObject, obj, key)
}

// vendor/k8s.io/client-go/tools/cache/thread_safe_store.go
func (c *threadSafeMap) updateIndices(oldObj interface{}, newObj interface{}, key string) {
	var oldIndexValues, indexValues []string
	var err error
	// 遍历所有的索引器KeyFunc
	for name, indexFunc := range c.indexers {
		if oldObj != nil {
			oldIndexValues, err = indexFunc(oldObj)
		} else {
			oldIndexValues = oldIndexValues[:0]
		}
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}

		if newObj != nil {
			// 通过索引器KeyFunc计算出索引StringSet
			indexValues, err = indexFunc(newObj)
		} else {
			indexValues = indexValues[:0]
		}
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}

		index := c.indices[name]
		if index == nil {
			index = Index{}
			// 如果索引器的名字对应的Indices还未实例化，则实例化一个
			c.indices[name] = index
		}

		// 将旧key对应的值全部删除
		for _, value := range oldIndexValues {
			// We optimize for the most common case where index returns a single value.
			if len(indexValues) == 1 && value == indexValues[0] {
				continue
			}
			c.deleteKeyFromIndex(key, value, index)
		}

		// 将新key对应的值全部添加
		for _, value := range indexValues {
			// We optimize for the most common case where index returns a single value.
			if len(oldIndexValues) == 1 && value == oldIndexValues[0] {
				continue
			}
			// 将索引设置到index中
			c.addKeyToIndex(key, value, index)
		}
	}
}

func (c *threadSafeMap) addKeyToIndex(key, indexValue string, index Index) {
	// 将该索引器计算出的key加入索引器对应的StringSet中
	set := index[indexValue]
	if set == nil {
		set = sets.String{}
		index[indexValue] = set
	}
	set.Insert(key)
}
```

### 查找
```go
// vendor/k8s.io/client-go/tools/cache/store.go
// 在查找时需要指定一个查找方式indexName，和查找所使用的key
func (c *cache) ByIndex(indexName, indexKey string) ([]interface{}, error) {
	return c.cacheStorage.ByIndex(indexName, indexKey)
}

// vendor/k8s.io/client-go/tools/cache/thread_safe_store.go
func (c *threadSafeMap) ByIndex(indexName, indexedValue string) ([]interface{}, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()

	// 通过查找方式获取keyFunc，判断keyFunc是否存在
	indexFunc := c.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}

	// 通过查找方式查找对应的index
	index := c.indices[indexName]

	// 通过查找使用的key在index中获取对应的StringSet
	set := index[indexedValue]
	list := make([]interface{}, 0, set.Len())
	for key := range set {
		// 遍历StringSet，从items中获取对应的资源对象，包装成list返回
		list = append(list, c.items[key])
	}

	return list, nil
}

```