# CodecFactory
通过CodecFactory来获取Codec，以实现资源到指定版本的序列化与反序列化

## 接口
```go
// vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go
type StorageSerializer interface {
	// SupportedMediaTypes are the media types supported for reading and writing objects.
	SupportedMediaTypes() []SerializerInfo

	// 返回一个Deserializer，支持多种数据类型资源的返回序列化
	UniversalDeserializer() Decoder

	// 传入一个serializer返回一个codec，能保证资源以指定的版本进行序列化
	EncoderForVersion(serializer Encoder, gv GroupVersioner) Encoder
	// 传入一个serializer返回一个codec，能保证资源以指定的版本进行反序列化
	DecoderToVersion(serializer Decoder, gv GroupVersioner) Decoder
}

// runtime包常数
const (
	ContentTypeJSON     string = "application/json"
	ContentTypeYAML     string = "application/yaml"
	ContentTypeProtobuf string = "application/vnd.kubernetes.protobuf"
)
```

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go
type CodecFactory struct {
    // scheme
	scheme    *runtime.Scheme
    // 解码器，实际上是recognizer.RecognizionDecoder，里面保存了所有的解码器，自动识别该用哪个解码器进行解码
	universal runtime.Decoder
    // 所有的序列化器相关的信息，方便索引
	accepts   []runtime.SerializerInfo

    // 存放默认的解析器(json格式)
	legacySerializer runtime.Serializer
}
```

## 构造函数
```go
// CodecFactory通过NewCodecFactory进行实例化
// 在实例化的过程中会将三种序列化器全部实例化，并保存在CodecFactory中 NewCodecFactory -> newSerializersForScheme
// vendor/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go
func NewCodecFactory(scheme *runtime.Scheme, mutators ...CodecFactoryOptionsMutator) CodecFactory {
	options := CodecFactoryOptions{Pretty: true}
	for _, fn := range mutators {
		fn(&options)
	}

	// 实例化三个Serializer
	serializers := newSerializersForScheme(scheme, json.DefaultMetaFactory, options)
	// 实例化CodecFactory
	return newCodecFactory(scheme, serializers)
}

func newSerializersForScheme(scheme *runtime.Scheme, mf json.MetaFactory, options CodecFactoryOptions) []serializerType {
    // jsonSerializer初始化
	jsonSerializer := json.NewSerializerWithOptions(
		mf, scheme, scheme,
		json.SerializerOptions{Yaml: false, Pretty: false, Strict: options.Strict},
	)
	jsonSerializerType := serializerType{
		AcceptContentTypes: []string{runtime.ContentTypeJSON},
		ContentType:        runtime.ContentTypeJSON,
		FileExtensions:     []string{"json"},
		EncodesAsText:      true,
		Serializer:         jsonSerializer,

		Framer:           json.Framer,
		StreamSerializer: jsonSerializer,
	}
    // ... jsonSerializerType其他字段的初始化

    // yamlSerializer初始化
	yamlSerializer := json.NewSerializerWithOptions(
		mf, scheme, scheme,
		json.SerializerOptions{Yaml: true, Pretty: false, Strict: options.Strict},
	)
    // ... yamlSerializerType其他字段的初始化

    // protoSerializer初始化
	protoSerializer := protobuf.NewSerializer(scheme, scheme)
	protoRawSerializer := protobuf.NewRawSerializer(scheme, scheme)

    // 将三个序列化器都包装进[]serializerType
	serializers := []serializerType{
		jsonSerializerType,
		{
			AcceptContentTypes: []string{runtime.ContentTypeYAML},
			ContentType:        runtime.ContentTypeYAML,
			FileExtensions:     []string{"yaml"},
			EncodesAsText:      true,
			Serializer:         yamlSerializer,
			StrictSerializer:   strictYAMLSerializer,
		},
		{
			AcceptContentTypes: []string{runtime.ContentTypeProtobuf},
			ContentType:        runtime.ContentTypeProtobuf,
			FileExtensions:     []string{"pb"},
			Serializer:         protoSerializer,
			// note, strict decoding is unsupported for protobuf,
			// fall back to regular serializing
			StrictSerializer: protoSerializer,

			Framer:           protobuf.LengthDelimitedFramer,
			StreamSerializer: protoRawSerializer,
		},
	}

    // 扩展函数
	for _, fn := range serializerExtensions {
		if serializer, ok := fn(scheme); ok {
			serializers = append(serializers, serializer)
		}
	}
	return serializers
}

func newCodecFactory(scheme *runtime.Scheme, serializers []serializerType) CodecFactory {
	decoders := make([]runtime.Decoder, 0, len(serializers))
	var accepts []runtime.SerializerInfo
	alreadyAccepted := make(map[string]struct{})

	var legacySerializer runtime.Serializer
	for _, d := range serializers {
		// 遍历上面获取到的三种序列化器
		decoders = append(decoders, d.Serializer)
		for _, mediaType := range d.AcceptContentTypes {
			if _, ok := alreadyAccepted[mediaType]; ok {
				continue
			}
			alreadyAccepted[mediaType] = struct{}{}
			info := runtime.SerializerInfo{
				MediaType:        d.ContentType,
				EncodesAsText:    d.EncodesAsText,
				Serializer:       d.Serializer,
				PrettySerializer: d.PrettySerializer,
				StrictSerializer: d.StrictSerializer,
			}

			mediaType, _, err := mime.ParseMediaType(info.MediaType)
			if err != nil {
				panic(err)
			}
			parts := strings.SplitN(mediaType, "/", 2)
			info.MediaTypeType = parts[0]
			info.MediaTypeSubType = parts[1]

			if d.StreamSerializer != nil {
				info.StreamSerializer = &runtime.StreamSerializerInfo{
					Serializer:    d.StreamSerializer,
					EncodesAsText: d.EncodesAsText,
					Framer:        d.Framer,
				}
			}
			// 提取出序列化器相关的参数，方便进行索引使用
			accepts = append(accepts, info)
			if mediaType == runtime.ContentTypeJSON {
				legacySerializer = d.Serializer
			}
		}
	}
	if legacySerializer == nil {
		legacySerializer = serializers[0].Serializer
	}

	return CodecFactory{
		scheme:    scheme,
		universal: recognizer.NewDecoder(decoders...),

		accepts: accepts,

		legacySerializer: legacySerializer,
	}
}
```

## 获取Codec
```go
// vendor/k8s.io/apimachinery/pgk/runtime/serializer/codec_factory.go
// CodecFactory的核心方法
// 传入serializer和deserializer以及对应的GroupVersioner，返回一个codec
// decodeGroupVersioner如果为空，将会反序列化为内部版本
// encodeGroupVersioner如果为空，无法序列化
func (f CodecFactory) CodecForVersions(encoder runtime.Encoder, decoder runtime.Decoder, encode runtime.GroupVersioner, decode runtime.GroupVersioner) runtime.Codec {
	if encode == nil {
		// 未指定encode版本时，将资源一律视作空版本。在编码时由于目标版本为空，无法识别，会直接编码失败
		encode = runtime.DisabledGroupVersioner
	}
	if decode == nil {
		// 未指定decode版本时，将资源一律视作内部版本
		decode = runtime.InternalGroupVersioner
	}
	// 实例化versioning.Codec
	return versioning.NewDefaultingCodecForScheme(f.scheme, encoder, decoder, encode, decode)
}

// 不指定decoder，返回的codec无法反序列化
func (f CodecFactory) DecoderToVersion(decoder runtime.Decoder, gv runtime.GroupVersioner) runtime.Decoder {
	return f.CodecForVersions(nil, decoder, nil, gv)
}

// 不指定encoder，返回的codec无法序列化
func (f CodecFactory) EncoderForVersion(encoder runtime.Encoder, gv runtime.GroupVersioner) runtime.Encoder {
	return f.CodecForVersions(encoder, nil, gv, nil)
}

// 需要的decoder和encoder可以通过runtime.SerializerInfoForMediaType
// 第一个参数是codecFactory.SerializerInfo，第二个参数是数据格式
// vendor/k8s.io/apimachinery/pkg/runtime/codec.go
func SerializerInfoForMediaType(types []SerializerInfo, mediaType string) (SerializerInfo, bool) {
	for _, info := range types {
		if info.MediaType == mediaType {
			return info, true
		}
	}
	for _, info := range types {
		if len(info.MediaType) == 0 {
			return info, true
		}
	}
	return SerializerInfo{}, false
}
```

## 使用示例
```go
func TestCodec(t *testing.T) {
	scheme := runtime.NewScheme()
	internalapps.AddToScheme(scheme)
	appsv1.AddToScheme(scheme)
	appsv1beta1.AddToScheme(scheme)
	codecFactory := serializer.NewCodecFactory(scheme)

	// 通过codecFactory获取ContentType对应的Serializer
	info, _ := runtime.SerializerInfoForMediaType(codecFactory.SupportedMediaTypes(), runtime.ContentTypeYAML)
	yamlSerializer := info.Serializer

	// 通过Serializer获取Codec，需要指定Codec的默认GVK
	codec := codecFactory.CodecForVersions(nil, yamlSerializer, nil, internalapps.SchemeGroupVersion)

	content, err := ioutil.ReadFile("deployment.yaml")
	if err != nil {
		panic(err)
	}

	object, err := runtime.Decode(codec, content)
	if err != nil {
		panic(err)
	}

	fmt.Println(scheme.ObjectKinds(object))
}
```