# RESTDeleteStrategy

## 删除策略
```go
// vendor/k8s.io/apiserver/pkg/registry/rest/delete.go
// RESTDeleteStrategy defines deletion behavior on an object that follows Kubernetes
// API conventions.
type RESTDeleteStrategy interface {
	runtime.ObjectTyper
}

type RESTGracefulDeleteStrategy interface {
	// CheckGracefulDelete should return true if the object can be gracefully deleted and set
	// any default values on the DeleteOptions.
	// NOTE: if return true, `options.GracePeriodSeconds` must be non-nil (nil will fail),
	// that's what tells the deletion how "graceful" to be.
	CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool
}

```
1. 如果一个资源对象同时实现了RESTDeleteStrategy、RESTGracefulDeleteStrategy，那么就会检查能否执行优雅删除
2. CheckGracefulDelete：检查能否执行优雅删除

## 删除流程
```go
func BeforeDelete(strategy RESTDeleteStrategy, ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
	objectMeta, gvk, kerr := objectMetaAndKind(strategy, obj)
	if kerr != nil {
		return false, false, kerr
	}
	if errs := validation.ValidateDeleteOptions(options); len(errs) > 0 {
		return false, false, errors.NewInvalid(schema.GroupKind{Group: metav1.GroupName, Kind: "DeleteOptions"}, "", errs)
	}
	// Checking the Preconditions here to fail early. They'll be enforced later on when we actually do the deletion, too.
	if options.Preconditions != nil {
		if options.Preconditions.UID != nil && *options.Preconditions.UID != objectMeta.GetUID() {
			return false, false, errors.NewConflict(schema.GroupResource{Group: gvk.Group, Resource: gvk.Kind}, objectMeta.GetName(), fmt.Errorf("the UID in the precondition (%s) does not match the UID in record (%s). The object might have been deleted and then recreated", *options.Preconditions.UID, objectMeta.GetUID()))
		}
		if options.Preconditions.ResourceVersion != nil && *options.Preconditions.ResourceVersion != objectMeta.GetResourceVersion() {
			return false, false, errors.NewConflict(schema.GroupResource{Group: gvk.Group, Resource: gvk.Kind}, objectMeta.GetName(), fmt.Errorf("the ResourceVersion in the precondition (%s) does not match the ResourceVersion in record (%s). The object might have been modified", *options.Preconditions.ResourceVersion, objectMeta.GetResourceVersion()))
		}
	}

	// Negative values will be treated as the value `1s` on the delete path.
	if gracePeriodSeconds := options.GracePeriodSeconds; gracePeriodSeconds != nil && *gracePeriodSeconds < 0 {
		options.GracePeriodSeconds = utilpointer.Int64(1)
	}
	if deletionGracePeriodSeconds := objectMeta.GetDeletionGracePeriodSeconds(); deletionGracePeriodSeconds != nil && *deletionGracePeriodSeconds < 0 {
		objectMeta.SetDeletionGracePeriodSeconds(utilpointer.Int64(1))
	}

    // 进行一次接口断言，看是否实现RESTGracefulDeleteStrategy
	gracefulStrategy, ok := strategy.(RESTGracefulDeleteStrategy)
	if !ok {
		// If we're not deleting gracefully there's no point in updating Generation, as we won't update
		// the obcject before deleting it.
		return false, false, nil
	}
	// if the object is already being deleted, no need to update generation.
	if objectMeta.GetDeletionTimestamp() != nil {
		// if we are already being deleted, we may only shorten the deletion grace period
		// this means the object was gracefully deleted previously but deletionGracePeriodSeconds was not set,
		// so we force deletion immediately
		// IMPORTANT:
		// The deletion operation happens in two phases.
		// 1. Update to set DeletionGracePeriodSeconds and DeletionTimestamp
		// 2. Delete the object from storage.
		// If the update succeeds, but the delete fails (network error, internal storage error, etc.),
		// a resource was previously left in a state that was non-recoverable.  We
		// check if the existing stored resource has a grace period as 0 and if so
		// attempt to delete immediately in order to recover from this scenario.
		if objectMeta.GetDeletionGracePeriodSeconds() == nil || *objectMeta.GetDeletionGracePeriodSeconds() == 0 {
			return false, false, nil
		}
		// only a shorter grace period may be provided by a user
		if options.GracePeriodSeconds != nil {
			period := int64(*options.GracePeriodSeconds)
			if period >= *objectMeta.GetDeletionGracePeriodSeconds() {
				return false, true, nil
			}
			newDeletionTimestamp := metav1.NewTime(
				objectMeta.GetDeletionTimestamp().Add(-time.Second * time.Duration(*objectMeta.GetDeletionGracePeriodSeconds())).
					Add(time.Second * time.Duration(*options.GracePeriodSeconds)))
			objectMeta.SetDeletionTimestamp(&newDeletionTimestamp)
			objectMeta.SetDeletionGracePeriodSeconds(&period)
			return true, false, nil
		}
		// graceful deletion is pending, do nothing
		options.GracePeriodSeconds = objectMeta.GetDeletionGracePeriodSeconds()
		return false, true, nil
	}

	// `CheckGracefulDelete` will be implemented by specific strategy
	if !gracefulStrategy.CheckGracefulDelete(ctx, obj, options) {
		return false, false, nil
	}

	if options.GracePeriodSeconds == nil {
		return false, false, errors.NewInternalError(fmt.Errorf("options.GracePeriodSeconds should not be nil"))
	}

	now := metav1.NewTime(metav1.Now().Add(time.Second * time.Duration(*options.GracePeriodSeconds)))
	objectMeta.SetDeletionTimestamp(&now)
	objectMeta.SetDeletionGracePeriodSeconds(options.GracePeriodSeconds)
	// If it's the first graceful deletion we are going to set the DeletionTimestamp to non-nil.
	// Controllers of the object that's being deleted shouldn't take any nontrivial actions, hence its behavior changes.
	// Thus we need to bump object's Generation (if set). This handles generation bump during graceful deletion.
	// The bump for objects that don't support graceful deletion is handled in pkg/registry/generic/registry/store.go.
	if objectMeta.GetGeneration() > 0 {
		objectMeta.SetGeneration(objectMeta.GetGeneration() + 1)
	}

	return true, false, nil
}
```