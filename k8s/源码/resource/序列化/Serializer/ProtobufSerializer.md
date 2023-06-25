# ProtobufSerializer
Protobuf格式的序列化器

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/protobuf/protobuf.go
type Serializer struct {
    // protobuf格式前缀
	prefix  []byte
	creater runtime.ObjectCreater
	typer   runtime.ObjectTyper
}

// 构造函数
func NewSerializer(creater runtime.ObjectCreater, typer runtime.ObjectTyper) *Serializer {
	return &Serializer{
        // protoEncodingPrefix = []byte{0x6b, 0x38, 0x73, 0x00}
		prefix:  protoEncodingPrefix,
		creater: creater,
		typer:   typer,
	}
}
```

## Decode
```go
func (s *Serializer) Decode(originalData []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	prefixLen := len(s.prefix)
	switch {
	case len(originalData) == 0:
		// TODO: treat like decoding {} from JSON with defaulting
		return nil, nil, fmt.Errorf("empty data")
	case len(originalData) < prefixLen || !bytes.Equal(s.prefix, originalData[:prefixLen]):
		return nil, nil, fmt.Errorf("provided data does not appear to be a protobuf message, expected prefix %v", s.prefix)
	case len(originalData) == prefixLen:
		// TODO: treat like decoding {} from JSON with defaulting
		return nil, nil, fmt.Errorf("empty body")
	}

	data := originalData[prefixLen:]
	unk := runtime.Unknown{}
	if err := unk.Unmarshal(data); err != nil {
		return nil, nil, err
	}

	actual := unk.GroupVersionKind()
	copyKindDefaults(&actual, gvk)

	if intoUnknown, ok := into.(*runtime.Unknown); ok && intoUnknown != nil {
		*intoUnknown = unk
		if ok, _, _ := s.RecognizesData(unk.Raw); ok {
			intoUnknown.ContentType = runtime.ContentTypeProtobuf
		}
		return intoUnknown, &actual, nil
	}

	if into != nil {
		types, _, err := s.typer.ObjectKinds(into)
		switch {
		case runtime.IsNotRegisteredError(err):
			pb, ok := into.(proto.Message)
			if !ok {
				return nil, &actual, errNotMarshalable{reflect.TypeOf(into)}
			}
			if err := proto.Unmarshal(unk.Raw, pb); err != nil {
				return nil, &actual, err
			}
			return into, &actual, nil
		case err != nil:
			return nil, &actual, err
		default:
			copyKindDefaults(&actual, &types[0])
			// if the result of defaulting did not set a version or group, ensure that at least group is set
			// (copyKindDefaults will not assign Group if version is already set). This guarantees that the group
			// of into is set if there is no better information from the caller or object.
			if len(actual.Version) == 0 && len(actual.Group) == 0 {
				actual.Group = types[0].Group
			}
		}
	}

	if len(actual.Kind) == 0 {
		return nil, &actual, runtime.NewMissingKindErr(fmt.Sprintf("%#v", unk.TypeMeta))
	}
	if len(actual.Version) == 0 {
		return nil, &actual, runtime.NewMissingVersionErr(fmt.Sprintf("%#v", unk.TypeMeta))
	}

	return unmarshalToObject(s.typer, s.creater, &actual, into, unk.Raw)
}
```

## Encode
```go
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
	if co, ok := obj.(runtime.CacheableObject); ok {
		return co.CacheEncode(s.Identifier(), s.doEncode, w)
	}
	return s.doEncode(obj, w)
}

func (s *Serializer) doEncode(obj runtime.Object, w io.Writer) error {
	prefixSize := uint64(len(s.prefix))

	var unk runtime.Unknown
	switch t := obj.(type) {
	case *runtime.Unknown:
		estimatedSize := prefixSize + uint64(t.Size())
		data := make([]byte, estimatedSize)
		i, err := t.MarshalTo(data[prefixSize:])
		if err != nil {
			return err
		}
		copy(data, s.prefix)
		_, err = w.Write(data[:prefixSize+uint64(i)])
		return err
	default:
		kind := obj.GetObjectKind().GroupVersionKind()
		unk = runtime.Unknown{
			TypeMeta: runtime.TypeMeta{
				Kind:       kind.Kind,
				APIVersion: kind.GroupVersion().String(),
			},
		}
	}

	switch t := obj.(type) {
	case bufferedMarshaller:
		// this path performs a single allocation during write but requires the caller to implement
		// the more efficient Size and MarshalToSizedBuffer methods
		encodedSize := uint64(t.Size())
		estimatedSize := prefixSize + estimateUnknownSize(&unk, encodedSize)
		data := make([]byte, estimatedSize)

		i, err := unk.NestedMarshalTo(data[prefixSize:], t, encodedSize)
		if err != nil {
			return err
		}

		copy(data, s.prefix)

		_, err = w.Write(data[:prefixSize+uint64(i)])
		return err

	case proto.Marshaler:
		// this path performs extra allocations
		data, err := t.Marshal()
		if err != nil {
			return err
		}
		unk.Raw = data

		estimatedSize := prefixSize + uint64(unk.Size())
		data = make([]byte, estimatedSize)

		i, err := unk.MarshalTo(data[prefixSize:])
		if err != nil {
			return err
		}

		copy(data, s.prefix)

		_, err = w.Write(data[:prefixSize+uint64(i)])
		return err

	default:
		// TODO: marshal with a different content type and serializer (JSON for third party objects)
		return errNotMarshalable{reflect.TypeOf(obj)}
	}
}
```