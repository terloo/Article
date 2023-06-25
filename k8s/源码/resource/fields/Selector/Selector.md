# Selector
Selector是定义了字段匹配相关的能力

## 接口
```go
// k8s.io/apimachinery/pkg/fields/selector.go
type Selector interface {
	// Matches returns true if this selector matches the given set of fields.
	Matches(Fields) bool

	// Empty returns true if this selector does not restrict the selection space.
	Empty() bool

	// RequiresExactMatch allows a caller to introspect whether a given selector
	// requires a single specific field to be set, and if so returns the value it
	// requires.
	RequiresExactMatch(field string) (value string, found bool)

	// Transform returns a new copy of the selector after TransformFunc has been
	// applied to the entire selector, or an error if fn returns an error.
	// If for a given requirement both field and value are transformed to empty
	// string, the requirement is skipped.
	Transform(fn TransformFunc) (Selector, error)

	// Requirements converts this interface to Requirements to expose
	// more detailed selection information.
	Requirements() Requirements

	// String returns a human readable string that represents this selector.
	String() string

	// Make a deep copy of the selector.
	DeepCopySelector() Selector
}

type TransformFunc func(field, value string) (newField, newValue string, err error)
```
1. Matches：是否能匹配该字段
2. Transform：返回所有字段的拷贝，并对其应用TransformFunc函数

## 实现类
1. andTerm
2. hasTerm
3. notHasTerm
4. nothingSelector