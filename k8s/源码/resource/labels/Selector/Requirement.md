# Requirement
Requirement是进行匹配操作的底层结构体，每一个Requirement代表着一个匹配条件

## 结构体
```go
// k8s.io/apimachinery/pkg/labels/selector.go
type Requirement struct {
	key      string
	operator selection.Operator
	// In huge majority of cases we have at most one value here.
	// It is generally faster to operate on a single-element slice
	// than on a single-element map, so we have a slice here.
	strValues []string
}

// k8s.io/apimachinery/pkg/selection/operator.go
type Operator string

const (
	DoesNotExist Operator = "!"
	Equals       Operator = "="
	DoubleEquals Operator = "=="
	In           Operator = "in"
	NotEquals    Operator = "!="
	NotIn        Operator = "notin"
	Exists       Operator = "exists"
	GreaterThan  Operator = "gt"
	LessThan     Operator = "lt"
)

```
1. key：匹配条件的key
2. operator：匹配操作
3. strValues：所有值

## 构造函数
```go
// k8s.io/apimachinery/pkg/labels/selector.go
func NewRequirement(key string, op selection.Operator, vals []string, opts ...field.PathOption) (*Requirement, error) {
	var allErrs field.ErrorList
	path := field.ToPath(opts...)
	if err := validateLabelKey(key, path.Child("key")); err != nil {
		allErrs = append(allErrs, err)
	}

	valuePath := path.Child("values")
	switch op {
	case selection.In, selection.NotIn:
		if len(vals) == 0 {
			allErrs = append(allErrs, field.Invalid(valuePath, vals, "for 'in', 'notin' operators, values set can't be empty"))
		}
	case selection.Equals, selection.DoubleEquals, selection.NotEquals:
		if len(vals) != 1 {
			allErrs = append(allErrs, field.Invalid(valuePath, vals, "exact-match compatibility requires one single value"))
		}
	case selection.Exists, selection.DoesNotExist:
		if len(vals) != 0 {
			allErrs = append(allErrs, field.Invalid(valuePath, vals, "values set must be empty for exists and does not exist"))
		}
	case selection.GreaterThan, selection.LessThan:
		if len(vals) != 1 {
			allErrs = append(allErrs, field.Invalid(valuePath, vals, "for 'Gt', 'Lt' operators, exactly one value is required"))
		}
		for i := range vals {
			if _, err := strconv.ParseInt(vals[i], 10, 64); err != nil {
				allErrs = append(allErrs, field.Invalid(valuePath.Index(i), vals[i], "for 'Gt', 'Lt' operators, the value must be an integer"))
			}
		}
	default:
		allErrs = append(allErrs, field.NotSupported(path.Child("operator"), op, validRequirementOperators))
	}

	for i := range vals {
		if err := validateLabelValue(key, vals[i], valuePath.Index(i)); err != nil {
			allErrs = append(allErrs, err)
		}
	}

    // 构建一个Requirement实例
	return &Requirement{key: key, operator: op, strValues: vals}, allErrs.ToAggregate()
}
```

## 匹配
```go
// k8s.io/apimachinery/pkg/labels/selector.go
// Matches returns true if the Requirement matches the input Labels.
// There is a match in the following cases:
// (1) The operator is Exists and Labels has the Requirement's key.
// (2) The operator is In, Labels has the Requirement's key and Labels'
//     value for that key is in Requirement's value set.
// (3) The operator is NotIn, Labels has the Requirement's key and
//     Labels' value for that key is not in Requirement's value set.
// (4) The operator is DoesNotExist or NotIn and Labels does not have the
//     Requirement's key.
// (5) The operator is GreaterThanOperator or LessThanOperator, and Labels has
//     the Requirement's key and the corresponding value satisfies mathematical inequality.
func (r *Requirement) Matches(ls Labels) bool {
	switch r.operator {
	case selection.In, selection.Equals, selection.DoubleEquals:
		if !ls.Has(r.key) {
			return false
		}
		return r.hasValue(ls.Get(r.key))
	case selection.NotIn, selection.NotEquals:
		if !ls.Has(r.key) {
			return true
		}
		return !r.hasValue(ls.Get(r.key))
	case selection.Exists:
		return ls.Has(r.key)
	case selection.DoesNotExist:
		return !ls.Has(r.key)
	case selection.GreaterThan, selection.LessThan:
		if !ls.Has(r.key) {
			return false
		}
		lsValue, err := strconv.ParseInt(ls.Get(r.key), 10, 64)
		if err != nil {
			klog.V(10).Infof("ParseInt failed for value %+v in label %+v, %+v", ls.Get(r.key), ls, err)
			return false
		}

		// There should be only one strValue in r.strValues, and can be converted to an integer.
		if len(r.strValues) != 1 {
			klog.V(10).Infof("Invalid values count %+v of requirement %#v, for 'Gt', 'Lt' operators, exactly one value is required", len(r.strValues), r)
			return false
		}

		var rValue int64
		for i := range r.strValues {
			rValue, err = strconv.ParseInt(r.strValues[i], 10, 64)
			if err != nil {
				klog.V(10).Infof("ParseInt failed for value %+v in requirement %#v, for 'Gt', 'Lt' operators, the value must be an integer", r.strValues[i], r)
				return false
			}
		}
		return (r.operator == selection.GreaterThan && lsValue > rValue) || (r.operator == selection.LessThan && lsValue < rValue)
	default:
		return false
	}
}
```

## 字符串形式
```go
// k8s.io/apimachinery/pkg/labels/selector.go
// 将Reuirement转化为字符串的形式
func (r *Requirement) String() string {
	var sb strings.Builder
	sb.Grow(
		// length of r.key
		len(r.key) +
			// length of 'r.operator' + 2 spaces for the worst case ('in' and 'notin')
			len(r.operator) + 2 +
			// length of 'r.strValues' slice times. Heuristically 5 chars per word
			+5*len(r.strValues))
	if r.operator == selection.DoesNotExist {
		sb.WriteString("!")
	}
	sb.WriteString(r.key)

	switch r.operator {
	case selection.Equals:
		sb.WriteString("=")
	case selection.DoubleEquals:
		sb.WriteString("==")
	case selection.NotEquals:
		sb.WriteString("!=")
	case selection.In:
		sb.WriteString(" in ")
	case selection.NotIn:
		sb.WriteString(" notin ")
	case selection.GreaterThan:
		sb.WriteString(">")
	case selection.LessThan:
		sb.WriteString("<")
	case selection.Exists, selection.DoesNotExist:
		return sb.String()
	}

	switch r.operator {
	case selection.In, selection.NotIn:
		sb.WriteString("(")
	}
	if len(r.strValues) == 1 {
		sb.WriteString(r.strValues[0])
	} else { // only > 1 since == 0 prohibited by NewRequirement
		// normalizes value order on output, without mutating the in-memory selector representation
		// also avoids normalization when it is not required, and ensures we do not mutate shared data
		sb.WriteString(strings.Join(safeSort(r.strValues), ","))
	}

	switch r.operator {
	case selection.In, selection.NotIn:
		sb.WriteString(")")
	}
	return sb.String()
}
```