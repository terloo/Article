# JsonYamlSerializer

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go
// JsonYaml公用的序列化器结构体
type Serializer struct {
    // 用于反序列化时推断出资源对象的GVK
	meta    MetaFactory
    // 序列化选项
	options SerializerOptions
    // 负责反序列化后创建对象
	creater runtime.ObjectCreater
    // 负责反序列化后识别类型
	typer   runtime.ObjectTyper

	identifier runtime.Identifier
}

type SerializerOptions struct {
    // 标识是否为yaml序列化
	Yaml bool

    // 是否为美化后的json，仅在json序列化时起作用
	Pretty bool

    // 是否为严格模式
	Strict bool
}

// 构造函数
func NewSerializerWithOptions(meta MetaFactory, creater runtime.ObjectCreater, typer runtime.ObjectTyper, options SerializerOptions) *Serializer {
	return &Serializer{
		meta:       meta,
		creater:    creater,
		typer:      typer,
		options:    options,
		identifier: identifier(options),
	}
}
```

## JsonMetaFactory
从json格式的content中推断出GVK
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/meta.go
func (SimpleMetaFactory) Interpret(data []byte) (*schema.GroupVersionKind, error) {
	findKind := struct {
		// +optional
		APIVersion string `json:"apiVersion,omitempty"`
		// +optional
		Kind string `json:"kind,omitempty"`
	}{}
    // 直接反序列化apiVersion和kind字段
	if err := json.Unmarshal(data, &findKind); err != nil {
		return nil, fmt.Errorf("couldn't get version/kind; json parse error: %v", err)
	}
    // 解析字符串为gvk
	gv, err := schema.ParseGroupVersion(findKind.APIVersion)
	if err != nil {
		return nil, err
	}
	return &schema.GroupVersionKind{Group: gv.Group, Version: gv.Version, Kind: findKind.Kind}, nil
}
```

## YamlMetaFactory
从Yaml格式的content中推断出GVK
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/yaml/meta.go
func (SimpleMetaFactory) Interpret(data []byte) (*schema.GroupVersionKind, error) {
	gvk := runtime.TypeMeta{}
    // 使用sigs.k8s.io/yaml进行反序列化
	if err := yaml.Unmarshal(data, &gvk); err != nil {
		return nil, fmt.Errorf("could not interpret GroupVersionKind; unmarshal error: %v", err)
	}
	gv, err := schema.ParseGroupVersion(gvk.APIVersion)
	if err != nil {
		return nil, err
	}
	return &schema.GroupVersionKind{Group: gv.Group, Version: gv.Version, Kind: gvk.Kind}, nil
}
```

## Decode过程
JsonYamlSerializer的GVK计算优先级 `originalData > default gvk > into`
```go
func (s *Serializer) Decode(originalData []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	data := originalData
    // 先将yaml转成json
	if s.options.Yaml {
		altered, err := yaml.YAMLToJSON(data)
		if err != nil {
			return nil, nil, err
		}
		data = altered
	}

    // 从源数据推断GVK
	actual, err := s.meta.Interpret(data)
	if err != nil {
		return nil, nil, err
	}

	if gvk != nil {
        // 将用户传入的GVK和实际解析出的GVK通过优先级进行整合
		*actual = gvkWithDefaults(*actual, *gvk)
	}

    // 如果希望解析到的obj是一个Unkonw
	if unk, ok := into.(*runtime.Unknown); ok && unk != nil {
		unk.Raw = originalData
		unk.ContentType = runtime.ContentTypeJSON
		unk.GetObjectKind().SetGroupVersionKind(*actual)
		return unk, actual, nil
	}

	if into != nil {
		_, isUnstructured := into.(runtime.Unstructured)
		types, _, err := s.typer.ObjectKinds(into)
		switch {
        // 如果用户传入obj是Unstructured或者未注册到Scheme
		case runtime.IsNotRegisteredError(err), isUnstructured:
			strictErrs, err := s.unmarshal(into, data, originalData)
			if err != nil {
				return nil, actual, err
			} else if len(strictErrs) > 0 {
				return into, actual, runtime.NewStrictDecodingError(strictErrs)
			}
			return into, actual, nil
		case err != nil:
			return nil, actual, err
		default:
			*actual = gvkWithDefaults(*actual, types[0])
		}
	}

	if len(actual.Kind) == 0 {
		return nil, actual, runtime.NewMissingKindErr(string(originalData))
	}
	if len(actual.Version) == 0 {
		return nil, actual, runtime.NewMissingVersionErr(string(originalData))
	}

	// use the target if necessary
    // 核对用户传入的obj的GVK，如果用户传入的obj是nil，则使用creater创建
	obj, err := runtime.UseOrCreateObject(s.typer, s.creater, *actual, into)
	if err != nil {
		return nil, actual, err
	}

    // 反序列化
	strictErrs, err := s.unmarshal(obj, data, originalData)
	if err != nil {
		return nil, actual, err
	} else if len(strictErrs) > 0 {
		return obj, actual, runtime.NewStrictDecodingError(strictErrs)
	}
	return obj, actual, nil
}
```

## Encode过程
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
	if co, ok := obj.(runtime.CacheableObject); ok {
		return co.CacheEncode(s.Identifier(), s.doEncode, w)
	}
	return s.doEncode(obj, w)
}

func (s *Serializer) doEncode(obj runtime.Object, w io.Writer) error {
    // 如果YAML字段为true，则将数据转为json后再转为yaml，
	if s.options.Yaml {
		json, err := json.Marshal(obj)
		if err != nil {
			return err
		}
		data, err := yaml.JSONToYAML(json)
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	}

	if s.options.Pretty {
		data, err := json.MarshalIndent(obj, "", "  ")
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	}
	encoder := json.NewEncoder(w)
	return encoder.Encode(obj)
}
```