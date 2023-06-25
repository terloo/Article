# Selector
Selector定义了Labels匹配相关的操作

## 接口
```go
// k8s.io/apimachinery/pkg/labels/selector.go
type Selector interface {
	// Matches returns true if this selector matches the given set of labels.
	Matches(Labels) bool

	// Empty returns true if this selector does not restrict the selection space.
	Empty() bool

	// String returns a human readable string that represents this selector.
	String() string

	// Add adds requirements to the Selector
	Add(r ...Requirement) Selector

	// Requirements converts this interface into Requirements to expose
	// more detailed selection information.
	// If there are querying parameters, it will return converted requirements and selectable=true.
	// If this selector doesn't want to select anything, it will return selectable=false.
	Requirements() (requirements Requirements, selectable bool)

	// Make a deep copy of the selector.
	DeepCopySelector() Selector

	// RequiresExactMatch allows a caller to introspect whether a given selector
	// requires a single specific label to be set, and if so returns the value it
	// requires.
	RequiresExactMatch(label string) (value string, found bool)
}
```
1. Matches：该Selector是否能匹配传入的Labels
2. Empty：如果该Selector没有匹配条件，返回true
3. Add：添加一个条件到该Selector
4. Requirements：将该Selecotor解析为requirement切片的形式
5. RequiresExactMatch：查找一个指定key在Selector匹配的值

## 实现类
1. internalSelector：Requirement的切片，对其所有Requirement进行与操作
2. nothingSelector：无法匹配到任何标签组

## 表现形式
1. internalSelector(Requirement切片)形式：以标准的Requirement结构体切片的形式存在。能直接使用Match方法来判断是否与标签组匹配
2. 字符串形式：以字符串形式存在，多用于请求的url参数中
3. map形式：以map形式存在，只能表示`=`匹配条件，多用于资源对象的`selector.matchLabels`字段中
4. Match表达式形式：以结构体`LabelSelectorRequirement`、`NodeSelectorRequirement`、`TopologySelectorLabelRequirement`形式存在，多用于资源对象的`selector.matchExpressions`字段中

## 转化关系
1. 在运行时，所有Selector的表现形式都必须转化为internalSelector形式才能对标签组进行匹配操作
2. internalSelector -> 字符串：`internalSelector.String()`
3. internalSelector -> map：`labels.ConvertSelectorToLabelsMap()`     注意无法转化除了等于以外的匹配操作
4. 字符串 -> internalSelector：`labels.Parse()`
5. map形式 -> internalSelector：`Set.AsSelector()`或者`Set.AsValidatedSelector()`
6. Match表达式形式 -> internalSelector：需要通过`NewRequirement`进行手动新建
