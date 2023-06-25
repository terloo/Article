# Visitor

## 基本接口Visitor
基础的Visitor接口，传入一个VisitorFunc，实现对*Info的处理步骤
```go
// vendor/k8s.io/cli-runtime/pkg/resource/intrefaces.go
// Visitor基础接口
type Visitor interface {
    Visit(VisitorFunc) error
}

// *Info结构用于存储与服务端交互获得的信息
type VisitorFunc func(*Info, error) error
```

## Visitor实现类概览
存放于`vendor/k8s.io/cli-runtime/pkg/resource/visitor.go`
| Visitor                | 说明                                                                                                                                                   |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| StreamVisitor          | 从io.Reader中获取数据流，将其转为JSON格式，再通过schema进行检查，最后将检查无误的数据转为Info对象                                                      |
| FileVisitor            | 从本地文件获取资源描述文件(`-f deployment.yaml`)，调用StreamVisitor将其转为Info对象                                                                    |
| URLVisitor             | 通过网络下载资源描述文件，调用StreamVisitor将其转为Info对象                                                                                            |
| KustomizeVisitor       | 从kustomize获取资源描述文件，调用StreamVisitor将其转为Info对象                                                                                         |
| DecoratedVisitor       | 提供多种装饰器(decorator)函数，通过遍历自身的装饰器对Info和error进行处理，如果任一一个装饰器函数报错，则终止并返回。装饰器函数通过NewDecorator进行注册 |
| ContinueOnErrorVisitor | 在执行多层Visitor匿名函数时，如果发生一个错误，不会立即返回，而是将所有错误收集进error数组并聚合为一条error信息                                        |
| FlattenListVisitor     | 如果runtime.Object是数组，将其展平方便处理                                                                                                             |
| FilteredVisitor        | 通过filters函数检查Info是否满足条件；如果满足，则继续向下执行，否则返回error信息。filters函数通过NewFilterVisitor进行注册                              |
| Selector               | 与api-server交互，生成Info对象，并设置Info对象的Object字段                                                                                             |


### VisitorList
Visitor的集合，循环调用集合中所有Visitor的Visit方法来处理*Info，如果某个Visitor中有错误，则直接返回
```go
type VisitorList []Visitor

func (l VisitorList) Visit(fn VisitorFunc) error {
	for i := range l {
		if err := l[i].Visit(fn); err != nil {
			return err
		}
	}
	return nil
}
```

### EagerVisitorList
循环调用集合中所有Visitor来处理*Info，如果某个Visitor中有错误，则将其保存到errs中并继续执行，直到所有Visitor执行完成再返回所有错误。
```go
type EagerVisitorList []Visitor

func (l EagerVisitorList) Visit(fn VisitorFunc) error {
	errs := []error(nil)
	for i := range l {
		if err := l[i].Visit(func(info *Info, err error) error {
			if err != nil {
				errs = append(errs, err)
				return nil
			}
			if err := fn(info, nil); err != nil {
				errs = append(errs, err)
			}
			return nil
		}); err != nil {
			errs = append(errs, err)
		}
	}
	return utilerrors.NewAggregate(errs)
}
```

### DecoratedVisitor
使用装饰模式，封装多个VisitorFunc步骤进一个Visitor中，出现错误后立即返回
```go
type DecoratedVisitor struct {
	visitor    Visitor
	decorators []VisitorFunc
}

// 将多个VisitorFunc封装到一个Visitor
func NewDecoratedVisitor(v Visitor, fn ...VisitorFunc) Visitor {
	if len(fn) == 0 {
		return v
	}
	return DecoratedVisitor{v, fn}
}

func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
        // 循环调用decorators的所有VisitorFunc，有错误立即返回
		for i := range v.decorators {
			if err := v.decorators[i](info, nil); err != nil {
				return err
			}
		}
        // 最后再调用原始Visitor中的VisitorFunc，有错误立即返回
		return fn(info, nil)
	})
}
```

### ContinueOnErrorVisitor
将一个Visitor包装为出现错误继续执行Visitor
```go
type ContinueOnErrorVisitor struct {
	Visitor
}

func (v ContinueOnErrorVisitor) Visit(fn VisitorFunc) error {
	errs := []error{}
	err := v.Visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			errs = append(errs, err)
			return nil
		}
		if err := fn(info, nil); err != nil {
			errs = append(errs, err)
		}
		return nil
	})
	if err != nil {
		errs = append(errs, err)
	}
	if len(errs) == 1 {
		return errs[0]
	}
	return utilerrors.NewAggregate(errs)
}
```

### FlattenListVisitor
```go
type FlattenListVisitor struct {
	visitor Visitor
	typer   runtime.ObjectTyper
	mapper  *mapper
}

func NewFlattenListVisitor(v Visitor, typer runtime.ObjectTyper, mapper *mapper) Visitor {
	return FlattenListVisitor{v, typer, mapper}
}

func (v FlattenListVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
		if info.Object == nil {
			return fn(info, nil)
		}
		if !meta.IsListType(info.Object) {
			return fn(info, nil)
		}

		items := []runtime.Object{}
		itemsToProcess := []runtime.Object{info.Object}

		for i := 0; i < len(itemsToProcess); i++ {
			currObj := itemsToProcess[i]
			if !meta.IsListType(currObj) {
				items = append(items, currObj)
				continue
			}

			currItems, err := meta.ExtractList(currObj)
			if err != nil {
				return err
			}
			if errs := runtime.DecodeList(currItems, v.mapper.decoder); len(errs) > 0 {
				return utilerrors.NewAggregate(errs)
			}
			itemsToProcess = append(itemsToProcess, currItems...)
		}

		// If we have a GroupVersionKind on the list, prioritize that when asking for info on the objects contained in the list
		var preferredGVKs []schema.GroupVersionKind
		if info.Mapping != nil && !info.Mapping.GroupVersionKind.Empty() {
			preferredGVKs = append(preferredGVKs, info.Mapping.GroupVersionKind)
		}
		errs := []error{}
		for i := range items {
			item, err := v.mapper.infoForObject(items[i], v.typer, preferredGVKs)
			if err != nil {
				errs = append(errs, err)
				continue
			}
			if len(info.ResourceVersion) != 0 {
				item.ResourceVersion = info.ResourceVersion
			}
			// propagate list source to items source
			if len(info.Source) != 0 {
				item.Source = info.Source
			}
			if err := fn(item, nil); err != nil {
				errs = append(errs, err)
			}
		}
		return utilerrors.NewAggregate(errs)

	})
}
```

## 创建资源时Visitor的包装层级
通过Result的Build的VisitByPath来包装Visitor
```
DecoratedVisitor{
	ContinueOnErrorVisitor{
		FlattenListVisitor{
			FlattenListVisitor{
				EagerVisitorList[
					FileVisitor{
						StreamVisitor{
							原始Visitor(与api-server交互逻辑在此处)
						}
					},
				]
			}
		}
	}
}
```

## 获取资源时Visitor的包装层级
通过Result的Build的VisitBySelector来包装Visitor
```
DecoratedVisitor{
	ContinueOnErrorVisitor{
		FlattenListVisitor{
			EagerVisitorList[
				Selector{(与api-server交互逻辑在此处)
					原始Visitor
				}
			]
		}
	}
}
```
> 在一次获取多种资源时(如kubectl get po,svc)，会生成多个Selector

## 创建或获取资源时Decorator中所有的装饰器
1. SetNamespace.func1
```go
// vendor/k8s.io/cli-runtime/pkg/resource/visitor.go
// 判断资源是否是命名空间级对象，如果不是，给资源设置命名空间
func SetNamespace(namespace string) VisitorFunc {
	return func(info *Info, err error) error {
		if err != nil {
			return err
		}
		if !info.Namespaced() {
			return nil
		}
		if len(info.Namespace) == 0 {
			info.Namespace = namespace
			UpdateObjectNamespace(info, nil)
		}
		return nil
	}
}
```
2. FilterNamespace
```go
// vendor/k8s.io/cli-runtime/pkg/resource/visitor.go
// 如果资源不是命名空间级别的，更新资源的命名空间为""
func FilterNamespace(info *Info, err error) error {
	if err != nil {
		return err
	}
	if !info.Namespaced() {
		info.Namespace = ""
		UpdateObjectNamespace(info, nil)
	}
	return nil
}
```
3. RetrieveLazy
```go
// vendor/k8s.io/cli-runtime/pkg/resource/visitor.go
// 检查info中是否含有资源对象，如果还没有，获取资源对象
func RetrieveLazy(info *Info, err error) error {
	if err != nil {
		return err
	}
	if info.Object == nil {
		return info.Get()
	}
	return nil
}
```