# Codec
Codec在序列化与反序列化时，指定的资源版本优先级更高

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/versioning/versioning.go
type codec struct {
	encoder   runtime.Encoder
	decoder   runtime.Decoder
	convertor runtime.ObjectConvertor
	creater   runtime.ObjectCreater
	typer     runtime.ObjectTyper
	defaulter runtime.ObjectDefaulter

	encodeVersion runtime.GroupVersioner
	decodeVersion runtime.GroupVersioner

	identifier runtime.Identifier

	// originalSchemeName is optional, but when filled in it holds the name of the scheme from which this codec originates
    // 原始sheme的名称
	originalSchemeName string
}
```
1. 实现了Codec接口，Encode和Decode实现了正常版本到内部版本相互转化后的序列化和反序列化
2. encoder、decoder：完成正常版本下的资源的某种格式的序列化和反序列化，例如json yaml
3. convertor：完成正常版本和内部版本之间的相互转化
4. creater、defaulter：创建正常版本的资源对象，并赋默认值
5. typer：确定资源对象的GVK
6. encodeVersion、decodeVersion：指定资源转化的版本、正常版本或内部版本
7. identifier：该codec的标识符
8. convertor、creater、typer、defaulter其实都是靠scheme实现

## 解码
GVK计算优先级 into(转化) > c.decodeVersion(转化) > (data > defaultGVK > into)
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/versioning/versioning.go
// 返回的GVK是Serializer序列化出来的GVK
func (c *codec) Decode(data []byte, defaultGVK *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	// If the into object is unstructured and expresses an opinion about its group/version,
	// create a new instance of the type so we always exercise the conversion path (skips short-circuiting on `into == obj`)
	decodeInto := into
	if into != nil {
		if _, ok := into.(runtime.Unstructured); ok && !into.GetObjectKind().GroupVersionKind().GroupVersion().Empty() {
			decodeInto = reflect.New(reflect.TypeOf(into).Elem()).Interface().(runtime.Object)
		}
	}

	var strictDecodingErr error
    // 先调用解码器进行解码
	obj, gvk, err := c.decoder.Decode(data, defaultGVK, decodeInto)
	if err != nil {
		if obj != nil && runtime.IsStrictDecodingError(err) {
			// save the strictDecodingError and the caller decide what to do with it
			strictDecodingErr = err
		} else {
			return nil, gvk, err
		}
	}

	if d, ok := obj.(runtime.NestedObjectDecoder); ok {
		if err := d.DecodeNestedObjects(runtime.WithoutVersionDecoder{c.decoder}); err != nil {
			return nil, gvk, err
		}
	}

	// if we specify a target, use generic conversion.
	// 如果指定了into对象，将反序列化之后的对象转为into对象
	if into != nil {
		// perform defaulting if requested
		if c.defaulter != nil {
			c.defaulter.Default(obj)
		}

		// Short-circuit conversion if the into object is same object
		if into == obj {
			return into, gvk, strictDecodingErr
		}

        // 如果有into，则转换为into对象
		if err := c.convertor.Convert(obj, into, c.decodeVersion); err != nil {
			return nil, gvk, err
		}

		return into, gvk, strictDecodingErr
	}

	// perform defaulting if requested
	if c.defaulter != nil {
		c.defaulter.Default(obj)
	}

    // 没有into，则转化为decodeVersion字段指定的版本
	out, err := c.convertor.ConvertToVersion(obj, c.decodeVersion)
	if err != nil {
		return nil, gvk, err
	}
	return out, gvk, strictDecodingErr
}
```

## 编码
GVK计算优先级 c.encodeVersion(转化) > obj在Scheme中注册的版本
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/versioning/versioning.go
func (c *codec) Encode(obj runtime.Object, w io.Writer) error {
	if co, ok := obj.(runtime.CacheableObject); ok {
		return co.CacheEncode(c.Identifier(), c.doEncode, w)
	}
	return c.doEncode(obj, w)
}

func (c *codec) doEncode(obj runtime.Object, w io.Writer) error {
	switch obj := obj.(type) {
	case *runtime.Unknown:
		return c.encoder.Encode(obj, w)
	case runtime.Unstructured:
		// An unstructured list can contain objects of multiple group version kinds. don't short-circuit just
		// because the top-level type matches our desired destination type. actually send the object to the converter
		// to give it a chance to convert the list items if needed.
		if _, ok := obj.(*unstructured.UnstructuredList); !ok {
			// avoid conversion roundtrip if GVK is the right one already or is empty (yes, this is a hack, but the old behaviour we rely on in kubectl)
			objGVK := obj.GetObjectKind().GroupVersionKind()
			if len(objGVK.Version) == 0 {
				return c.encoder.Encode(obj, w)
			}
			targetGVK, ok := c.encodeVersion.KindForGroupVersionKinds([]schema.GroupVersionKind{objGVK})
			if !ok {
				return runtime.NewNotRegisteredGVKErrForTarget(c.originalSchemeName, objGVK, c.encodeVersion)
			}
			if targetGVK == objGVK {
				return c.encoder.Encode(obj, w)
			}
		}
	}

	gvks, isUnversioned, err := c.typer.ObjectKinds(obj)
	if err != nil {
		return err
	}

	objectKind := obj.GetObjectKind()
	old := objectKind.GroupVersionKind()
	// restore the old GVK after encoding
	defer objectKind.SetGroupVersionKind(old)

	if c.encodeVersion == nil || isUnversioned {
		if e, ok := obj.(runtime.NestedObjectEncoder); ok {
			if err := e.EncodeNestedObjects(runtime.WithVersionEncoder{Encoder: c.encoder, ObjectTyper: c.typer}); err != nil {
				return err
			}
		}
		objectKind.SetGroupVersionKind(gvks[0])
		return c.encoder.Encode(obj, w)
	}

	// Perform a conversion if necessary
    // 转为资源版本为encodeVersion的版本
	out, err := c.convertor.ConvertToVersion(obj, c.encodeVersion)
	if err != nil {
		return err
	}

	if e, ok := out.(runtime.NestedObjectEncoder); ok {
		if err := e.EncodeNestedObjects(runtime.WithVersionEncoder{Version: c.encodeVersion, Encoder: c.encoder, ObjectTyper: c.typer}); err != nil {
			return err
		}
	}

	// Conversion is responsible for setting the proper group, version, and kind onto the outgoing object
	return c.encoder.Encode(out, w)
}
```