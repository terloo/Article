# Requirement
Requirement用于进行匹配操作的结构体

## 结构体
```go
// k8s.io/apimachinery/pkg/fields/requirements.go

// Requirement contains a field, a value, and an operator that relates the field and value.
// This is currently for reading internal selection information of field selector.
type Requirement struct {
	Operator selection.Operator
	Field    string
	Value    string
}

// Requirements is AND of all requirements.
type Requirements []Requirement
```