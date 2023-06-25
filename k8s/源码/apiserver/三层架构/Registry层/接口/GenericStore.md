# GenericStore
GenericStore定义了用于获取增改删策略的接口  
策略接口分别定义在create.go update.go delete.go create_update.go中，文件中还定义了使用策略接口的函数

## 接口
```go
// vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
// 所有策略接口的集合接口
type GenericStore interface {
	GetCreateStrategy() rest.RESTCreateStrategy
	GetUpdateStrategy() rest.RESTUpdateStrategy
	GetDeleteStrategy() rest.RESTDeleteStrategy
}
```

## rest.RESTCreateStrategy
```go
// k8s.io/apiserver/pkg/registry/generic/rest.create.go
type RESTCreateStrategy interface {
	runtime.ObjectTyper
	// The name generator is used when the standard GenerateName field is set.
	// The NameGenerator will be invoked prior to validation.
	names.NameGenerator

	// NamespaceScoped returns true if the object must be within a namespace.
	NamespaceScoped() bool
	// PrepareForCreate is invoked on create before validation to normalize
	// the object.  For example: remove fields that are not to be persisted,
	// sort order-insensitive list fields, etc.  This should not remove fields
	// whose presence would be considered a validation error.
	//
	// Often implemented as a type check and an initailization or clearing of
	// status. Clear the status because status changes are internal. External
	// callers of an api (users) should not be setting an initial status on
	// newly created objects.
	PrepareForCreate(ctx context.Context, obj runtime.Object)
	// Validate returns an ErrorList with validation errors or nil.  Validate
	// is invoked after default fields in the object have been filled in
	// before the object is persisted.  This method should not mutate the
	// object.
	Validate(ctx context.Context, obj runtime.Object) field.ErrorList
	// WarningsOnCreate returns warnings to the client performing a create.
	// WarningsOnCreate is invoked after default fields in the object have been filled in
	// and after Validate has passed, before Canonicalize is called, and the object is persisted.
	// This method must not mutate the object.
	//
	// Be brief; limit warnings to 120 characters if possible.
	// Don't include a "Warning:" prefix in the message (that is added by clients on output).
	// Warnings returned about a specific field should be formatted as "path.to.field: message".
	// For example: `spec.imagePullSecrets[0].name: invalid empty name ""`
	//
	// Use warning messages to describe problems the client making the API request should correct or be aware of.
	// For example:
	// - use of deprecated fields/labels/annotations that will stop working in a future release
	// - use of obsolete fields/labels/annotations that are non-functional
	// - malformed or invalid specifications that prevent successful handling of the submitted object,
	//   but are not rejected by validation for compatibility reasons
	//
	// Warnings should not be returned for fields which cannot be resolved by the caller.
	// For example, do not warn about spec fields in a subresource creation request.
	WarningsOnCreate(ctx context.Context, obj runtime.Object) []string
	// Canonicalize allows an object to be mutated into a canonical form. This
	// ensures that code that operates on these objects can rely on the common
	// form for things like comparison.  Canonicalize is invoked after
	// validation has succeeded but before the object has been persisted.
	// This method may mutate the object. Often implemented as a type check or
	// empty method.
	Canonicalize(obj runtime.Object)
}
```
1. ObjectTyper：传入Scheme
2. NameGenerator：生成GR对象名
3. NamespaceScoped：返回GK是否为命名空间级别
4. PrepareForCreate：创建前预准备
5. Validate：创建前验证
6. WarningsOnCreate：是否需要返回Warning
7. Canonicalize：规范化对象格式

## rest.RESTUpdateStrategy
```go
// k8s.io/apiserver/pkg/resigtry/rest/update.go
type RESTUpdateStrategy interface {
	runtime.ObjectTyper
	// NamespaceScoped returns true if the object must be within a namespace.
	NamespaceScoped() bool
	// AllowCreateOnUpdate returns true if the object can be created by a PUT.
	AllowCreateOnUpdate() bool
	// PrepareForUpdate is invoked on update before validation to normalize
	// the object.  For example: remove fields that are not to be persisted,
	// sort order-insensitive list fields, etc.  This should not remove fields
	// whose presence would be considered a validation error.
	PrepareForUpdate(ctx context.Context, obj, old runtime.Object)
	// ValidateUpdate is invoked after default fields in the object have been
	// filled in before the object is persisted.  This method should not mutate
	// the object.
	ValidateUpdate(ctx context.Context, obj, old runtime.Object) field.ErrorList
	// WarningsOnUpdate returns warnings to the client performing the update.
	// WarningsOnUpdate is invoked after default fields in the object have been filled in
	// and after ValidateUpdate has passed, before Canonicalize is called, and before the object is persisted.
	// This method must not mutate either object.
	//
	// Be brief; limit warnings to 120 characters if possible.
	// Don't include a "Warning:" prefix in the message (that is added by clients on output).
	// Warnings returned about a specific field should be formatted as "path.to.field: message".
	// For example: `spec.imagePullSecrets[0].name: invalid empty name ""`
	//
	// Use warning messages to describe problems the client making the API request should correct or be aware of.
	// For example:
	// - use of deprecated fields/labels/annotations that will stop working in a future release
	// - use of obsolete fields/labels/annotations that are non-functional
	// - malformed or invalid specifications that prevent successful handling of the submitted object,
	//   but are not rejected by validation for compatibility reasons
	//
	// Warnings should not be returned for fields which cannot be resolved by the caller.
	// For example, do not warn about spec fields in a status update.
	WarningsOnUpdate(ctx context.Context, obj, old runtime.Object) []string
	// Canonicalize allows an object to be mutated into a canonical form. This
	// ensures that code that operates on these objects can rely on the common
	// form for things like comparison.  Canonicalize is invoked after
	// validation has succeeded but before the object has been persisted.
	// This method may mutate the object.
	Canonicalize(obj runtime.Object)
	// AllowUnconditionalUpdate returns true if the object can be updated
	// unconditionally (irrespective of the latest resource version), when
	// there is no resource version specified in the object.
	AllowUnconditionalUpdate() bool
}
```
1. AllowCreateOnUpdate：是否允许在对象不存在时新建
2. AllowUnconditionalUpdate：是否允许无条件更新

## rest.RESTDeleteStrategy
删除策略有两个选择实现的接口
1. GarbageCollectionDeleteStrategy：实现了此策略接口的资源，在删除后还需要进行垃圾收集
2. RESTGracefulDeleteStrategy：实现类此策略接口的资源，在收到删除操作后会首先进行优雅删除操作，**目前只有Pod实现了此接口**
```go
// k8s.io/apiserver/pkg/registry/rest/delete.go
type RESTDeleteStrategy interface {
	runtime.ObjectTyper
}

type GarbageCollectionPolicy string

const (
	DeleteDependents GarbageCollectionPolicy = "DeleteDependents"
	OrphanDependents GarbageCollectionPolicy = "OrphanDependents"
	// Unsupported means that the resource knows that it cannot be GC'd, so the finalizers
	// should never be set in storage.
	Unsupported GarbageCollectionPolicy = "Unsupported"
)

type GarbageCollectionDeleteStrategy interface {
    // 返回资源被删除后应该进行的垃圾收集行为
	DefaultGarbageCollectionPolicy(ctx context.Context) GarbageCollectionPolicy
}

type RESTGracefulDeleteStrategy interface {
    // 是否允许优雅删除，并修改DeleteOptions中字段
	CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool
}
```