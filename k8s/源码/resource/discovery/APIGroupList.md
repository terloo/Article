# APIGroupList

## Discovery路径
`/apis`

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
type APIGroupList struct {
	TypeMeta `json:",inline"`
	// groups is a list of APIGroup.
	Groups []APIGroup `json:"groups" protobuf:"bytes,1,rep,name=groups"`
}
```