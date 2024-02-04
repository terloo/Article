# Result
RESTClient用于保存响应结果的结构体

## 结构体
```go
// k8s.io/client-go/rest/request.go
type Result struct {
	body        []byte
	warnings    []net.WarningHeader
	contentType string
	err         error
	statusCode  int

	decoder runtime.Decoder
}
```
1. body：原始响应数据
2. warnings：响应头Warning处理器
3. contentType：响应数据格式
4. statusCode：状态码
5. decoder：解码器

## Into
将响应体解码为指定的runtime.Object
```go
// k8s.io/client-go/rest/request.go
func (r Result) Into(obj runtime.Object) error {
	if r.err != nil {
		// Check whether the result has a Status object in the body and prefer that.
		return r.Error()
	}
	if r.decoder == nil {
		return fmt.Errorf("serializer for %s doesn't exist", r.contentType)
	}
	if len(r.body) == 0 {
		return fmt.Errorf("0-length response with status code: %d and content type: %s",
			r.statusCode, r.contentType)
	}

    // 使用Decoder将响应体解码到obj中
	out, _, err := r.decoder.Decode(r.body, nil, obj)
	if err != nil || out == obj {
		return err
	}

    // 如果请求是增删改请求，那么响应应该为metav1.Status
	// if a different object is returned, see if it is Status and avoid double decoding
	// the object.
	switch t := out.(type) {
	case *metav1.Status:
		// any status besides StatusSuccess is considered an error.
		if t.Status != metav1.StatusSuccess {
			return errors.FromObject(t)
		}
	}
	return nil
}
```