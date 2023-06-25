# 资源注册表Scheme
k8s中所有的资源必须注册进资源注册表Scheme中才能被k8s所管理。  
Scheme是一个内存型的注册表，有如下特点：
1. 支持注册多种版本资源，包括内外部版本
2. 支持资源多版本转化
3. 支持不同资源的序列化、反序列化

## 无/有版本资源
Scheme支持的资源类型被分为两类：
1. UnversionedTypes：无版本资源。一种早期的资源，在metav1元数据中仍存在部分类型，它们既属于meta.k8s.io/v1又属于UnversionedType。例如metav1.Status、metav1.APIVersion、metav1.APIGroupList、metav1.APIGroup、metav1.APIResourceList
2. KnownTypes：有版本资源，普通的资源对象
> k8s中UnVersionType通过scheme.AddUnversionedTypes，KnownType通过scheme.AddKnownType进行注册

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime.scheme.go
// Scheme主体由四个map结构组成
type Scheme struct {
	// versionMap allows one to figure out the go type of an object with
	// the given version and name.
	gvkToType map[schema.GroupVersionKind]reflect.Type

	// typeToGroupVersion allows one to find metadata for a given go object.
	// The reflect.Type we index by should *not* be a pointer.
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	// unversionedTypes are transformed without conversion in ConvertToVersion.
	unversionedTypes map[reflect.Type]schema.GroupVersionKind

	// unversionedKinds are the names of kinds that can be created in the context of any group
	// or version
	// TODO: resolve the status of unversioned types.
	unversionedKinds map[string]reflect.Type

	// Map from version and resource to the corresponding func to convert
	// resource field labels in that version to internal version.
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc

	// defaulterFuncs is an array of interfaces to be called with an object to provide defaulting
	// the provided object must be a pointer.
	defaulterFuncs map[reflect.Type]func(interface{})

	// converter stores all registered conversion functions. It also has
	// default converting behavior.
	converter *conversion.Converter

	// versionPriority is a map of groups to ordered lists of versions for those groups indicating the
	// default priorities of these versions as registered in the scheme
	versionPriority map[string][]string

	// observedVersions keeps track of the order we've seen versions during type registration
	observedVersions []schema.GroupVersion

	// schemeName is the name of this scheme.  If you don't specify a name, the stack of the NewScheme caller will be used.
	// This is useful for error reporting to indicate the origin of the scheme.
	schemeName string
}
```
1. gvkToType：GVK -> goType。注意无版本资源对象也会保存在这个map中
2. typeToGVK：goType -> []GVK。一个goType可能对应多个GVK，表明这多个GVK是共享了一个结构体。注意无版本资源对象也会保存在这个map中
3. unversionedTypes：goType -> unversionedTypes的GVK
4. unversionedKinds：unversionedTypes的Kind名 -> goType
5. defaulterFuncs：保存Type的默认值函数
6. converter：版本转化器，资源的版本转化主要委托给版本转化器处理
7. versionPriority：版本优先级  group -> version优先级数组
8. observedVersions：已注册过的GroupVersion
9. schemeName：scheme的名字

## 构造函数
```go
// k8s.io/apimachinery/pkg/runtime/scheme.go
func NewScheme() *Scheme {
	s := &Scheme{
		gvkToType:                 map[schema.GroupVersionKind]reflect.Type{},
		typeToGVK:                 map[reflect.Type][]schema.GroupVersionKind{},
		unversionedTypes:          map[reflect.Type]schema.GroupVersionKind{},
		unversionedKinds:          map[string]reflect.Type{},
		fieldLabelConversionFuncs: map[schema.GroupVersionKind]FieldLabelConversionFunc{},
		defaulterFuncs:            map[reflect.Type]func(interface{}){},
		versionPriority:           map[string][]string{},
		// scheme的名字是调用此函数的文件名:行数
		schemeName:                naming.GetNameFromCallsite(internalPackages...),
	}
	s.converter = conversion.NewConverter(nil)

	// Enable couple default conversions by default.
	// 注册了几个通用转化函数
	utilruntime.Must(RegisterEmbeddedConversions(s))
	utilruntime.Must(RegisterStringConversions(s))
	return s
}
```

## 注册方法
1. scheme.AddUnversionedTypes：注册无版本资源类型
2. scheme.AddKnownTypes：注册普通资源，可以同时注册多个resource
3. scheme.AddKnownTypeWithName：注册资源，并指定Kind名。**如果未指定，则以结构体的名字作为类型名**。AddUnversionedTypes内部调用该方法进行注册
4. 通过AddUnversionedTypes注册的资源会同时存在于四个map结构中，AddKnownTypes注册的资源不会存在于Unversion相关的资源中

## 资源注册查询方法
1. scheme.KnownTypes(gv schema.GroupVersion)：通过GV查询其下所有的map[KindName]reflect.Type
2. scheme.AllKnownTypes()：直接返回gvkToType
3. scheme.ObjectKinds(obj Object)：通过Object查询所属的GVK，可能有多个GVK，返回[]schema.GroupVersionKind
4. scheme.New(kind schema.GroupVersionKind)：通过GVK获取一个资源对象的实例
5. scheme.IsGroupRegistered(group string)：判断指定组是否已注册
6. scheme.IsVersionRegistered(version schema.GroupVersion)：判断指定的GV是否已注册
7. scheme.Recognizes(gvk schema.GroupVersionKind)：判断指定的GVK是否已注册
8. scheme.IsUnversioned(obj Object)：判断Object是否为Unversioned资源
