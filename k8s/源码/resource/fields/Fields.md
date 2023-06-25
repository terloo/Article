# Fields
Fields定义了一个键值读操作的功能，用于操作键值对属性

## 接口
```go
type Fields interface {
	// Has returns whether the provided field exists.
	Has(field string) (exists bool)

	// Get returns the value for the provided field.
	Get(field string) (value string)
}
```
1. Has：判断是否有此field
2. Get：获取field的值

## 实现类
```go
type Set map[string]string

func (ls Set) Has(field string) bool {
	_, exists := ls[field]
	return exists
}

func (ls Set) Get(field string) string {
	return ls[field]
}

// 返回所有键值对，格式k1=v1,k2=v2
func (ls Set) String() string {
	selector := make([]string, 0, len(ls))
	for key, value := range ls {
		selector = append(selector, key+"="+value)
	}
	// Sort for determinism.
	sort.StringSlice(selector).Sort()
	return strings.Join(selector, ",")
}
```