# RESTUpdateStrategy

## 更新对象的策略
```go
// vendor/k8s.io/apiserver/pkg/registry/rest/update.go
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
1. NamespaceScoped：判断资源是否是命名空间级别，如果是，会校验命名空间
2. AllowCreateOnUpdate：在更新当前资源时，如果资源对象未存在，是否允许Create
3. PrepareForUpdate：更新当前资源对象之前的处理函数
4. ValidateUpdate：更新资源对象前的校验
5. WarningsOnUpdate：如果需要，给调用方警告信息
6. AllowUnconditionalUpdate：如果资源对象未指定资源版本，是否允许更新

## 更新对象策略执行流程
在UpdateStrategy中有一个包装好的创建对象策略流程函数BeforeUpdate供apiserver调用
```go
// vendor/k8s.io/apiserver/pkg/registry/rest/update.go
// 执行流程 AllowCreateOnUpdate -> AllowUnconditionalUpdate -> NamespaceScoped -> PrepareForUpdate -> ValidateUpdate
func BeforeUpdate(strategy RESTUpdateStrategy, ctx context.Context, obj, old runtime.Object) error {
	objectMeta, kind, kerr := objectMetaAndKind(strategy, obj)
	if kerr != nil {
		return kerr
	}
	if strategy.NamespaceScoped() {
		if !ValidNamespace(ctx, objectMeta) {
			return errors.NewBadRequest("the namespace of the provided object does not match the namespace sent on the request")
		}
	} else if len(objectMeta.GetNamespace()) > 0 {
		objectMeta.SetNamespace(metav1.NamespaceNone)
	}

	// Ensure requests cannot update generation
	oldMeta, err := meta.Accessor(old)
	if err != nil {
		return err
	}
	objectMeta.SetGeneration(oldMeta.GetGeneration())

	// Ensure managedFields state is removed unless ServerSideApply is enabled
	if !utilfeature.DefaultFeatureGate.Enabled(features.ServerSideApply) {
		oldMeta.SetManagedFields(nil)
		objectMeta.SetManagedFields(nil)
	}

	strategy.PrepareForUpdate(ctx, obj, old)

	// ClusterName is ignored and should not be saved
	if len(objectMeta.GetClusterName()) > 0 {
		objectMeta.SetClusterName("")
	}
	// Use the existing UID if none is provided
	if len(objectMeta.GetUID()) == 0 {
		objectMeta.SetUID(oldMeta.GetUID())
	}
	// ignore changes to timestamp
	if oldCreationTime := oldMeta.GetCreationTimestamp(); !oldCreationTime.IsZero() {
		objectMeta.SetCreationTimestamp(oldMeta.GetCreationTimestamp())
	}
	// an update can never remove/change a deletion timestamp
	if !oldMeta.GetDeletionTimestamp().IsZero() {
		objectMeta.SetDeletionTimestamp(oldMeta.GetDeletionTimestamp())
	}
	// an update can never remove/change grace period seconds
	if oldMeta.GetDeletionGracePeriodSeconds() != nil && objectMeta.GetDeletionGracePeriodSeconds() == nil {
		objectMeta.SetDeletionGracePeriodSeconds(oldMeta.GetDeletionGracePeriodSeconds())
	}

	// Ensure some common fields, like UID, are validated for all resources.
	errs, err := validateCommonFields(obj, old, strategy)
	if err != nil {
		return errors.NewInternalError(err)
	}

	errs = append(errs, strategy.ValidateUpdate(ctx, obj, old)...)
	if len(errs) > 0 {
		return errors.NewInvalid(kind.GroupKind(), objectMeta.GetName(), errs)
	}

	for _, w := range strategy.WarningsOnUpdate(ctx, obj, old) {
		warning.AddWarning(ctx, "", w)
	}

	strategy.Canonicalize(obj)

	return nil
}
```