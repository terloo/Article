# Request
RESTClient使用的Reqeust对象

## 结构体
```go
// k8s.io/client-go/rest/request.go
type Request struct {
	c *RESTClient

	warningHandler WarningHandler

	rateLimiter flowcontrol.RateLimiter
	backoff     BackoffManager
	timeout     time.Duration

	// generic components accessible via method setters
	verb       string
	pathPrefix string
	subpath    string
	params     url.Values
	headers    http.Header

	// structural elements of the request that are part of the Kubernetes API conventions
	namespace    string
	namespaceSet bool
	resource     string
	resourceName string
	subresource  string

	// output
	err   error
	body  io.Reader
	retry WithRetry
}
```
1. pathPrefix：为RESTClient.versionedAPIPath

## 构建函数
一般都由RESTClient进行调用
```go
// k8s.io/client-go/rest/request.go
func NewRequest(c *RESTClient) *Request {
	var backoff BackoffManager
	if c.createBackoffMgr != nil {
		backoff = c.createBackoffMgr()
	}
	if backoff == nil {
		backoff = noBackoff
	}

	var pathPrefix string
	if c.base != nil {
		pathPrefix = path.Join("/", c.base.Path, c.versionedAPIPath)
	} else {
		pathPrefix = path.Join("/", c.versionedAPIPath)
	}

	var timeout time.Duration
	if c.Client != nil {
		timeout = c.Client.Timeout
	}

	r := &Request{
		c:              c,
		rateLimiter:    c.rateLimiter,
		backoff:        backoff,
		timeout:        timeout,
		pathPrefix:     pathPrefix,
		retry:          &withRetry{maxRetries: 10},
		warningHandler: c.warningHandler,
	}

	switch {
	case len(c.content.AcceptContentTypes) > 0:
		r.SetHeader("Accept", c.content.AcceptContentTypes)
	case len(c.content.ContentType) > 0:
        // 没有显示指定Accept，则使用ContentType + */*
		r.SetHeader("Accept", c.content.ContentType+", */*")
	}
	return r
}
```

## 一些链式方法
```go
// k8s.io/client-go/rest/request.go

// 设置请求类型
func (r *Request) Verb(verb string) *Request {
	r.verb = verb
	return r
}

// 设置命名空间
func (r *Request) Namespace(namespace string) *Request {
	if r.err != nil {
		return r
	}
	if r.namespaceSet {
		r.err = fmt.Errorf("namespace already set to %q, cannot change to %q", r.namespace, namespace)
		return r
	}
	if msgs := IsValidPathSegmentName(namespace); len(msgs) != 0 {
		r.err = fmt.Errorf("invalid namespace %q: %v", namespace, msgs)
		return r
	}
	r.namespaceSet = true
	r.namespace = namespace
	return r
}

// 设置资源名
func (r *Request) Resource(resource string) *Request {
	if r.err != nil {
		return r
	}
	if len(r.resource) != 0 {
		r.err = fmt.Errorf("resource already set to %q, cannot change to %q", r.resource, resource)
		return r
	}
	if msgs := IsValidPathSegmentName(resource); len(msgs) != 0 {
		r.err = fmt.Errorf("invalid resource %q: %v", resource, msgs)
		return r
	}
	r.resource = resource
	return r
}

// 设置请求的绝对路径，注意这会覆盖pathPrefix(即/api /apis)
func (r *Request) AbsPath(segments ...string) *Request {
	if r.err != nil {
		return r
	}
	r.pathPrefix = path.Join(r.c.base.Path, path.Join(segments...))
	if len(segments) == 1 && (len(r.c.base.Path) > 1 || len(segments[0]) > 1) && strings.HasSuffix(segments[0], "/") {
		// preserve any trailing slashes for legacy behavior
		r.pathPrefix += "/"
	}
	return r
}

// 设置请求体
func (r *Request) Body(obj interface{}) *Request {
	if r.err != nil {
		return r
	}
	switch t := obj.(type) {
	case string:
		data, err := ioutil.ReadFile(t)
		if err != nil {
			r.err = err
			return r
		}
		glogBody("Request Body", data)
		r.body = bytes.NewReader(data)
	case []byte:
		glogBody("Request Body", t)
		r.body = bytes.NewReader(t)
	case io.Reader:
		r.body = t
	case runtime.Object:
		// callers may pass typed interface pointers, therefore we must check nil with reflection
		if reflect.ValueOf(t).IsNil() {
			return r
		}
		encoder, err := r.c.content.Negotiator.Encoder(r.c.content.ContentType, nil)
		if err != nil {
			r.err = err
			return r
		}
		data, err := runtime.Encode(encoder, t)
		if err != nil {
			r.err = err
			return r
		}
		glogBody("Request Body", data)
		r.body = bytes.NewReader(data)
		r.SetHeader("Content-Type", r.c.content.ContentType)
	default:
		r.err = fmt.Errorf("unknown type used for body: %+v", obj)
	}
	return r
}
```

## VersionedParams
使用listOptions来添加查询参数
```go
// k8s.io/client-go/rest/request.go
func (r *Request) VersionedParams(obj runtime.Object, codec runtime.ParameterCodec) *Request {
	return r.SpecificallyVersionedParams(obj, codec, r.c.content.GroupVersion)
}

func (r *Request) SpecificallyVersionedParams(obj runtime.Object, codec runtime.ParameterCodec, version schema.GroupVersion) *Request {
	if r.err != nil {
		return r
	}
	// 将参数编码为map[string][]string的数据格式
	params, err := codec.EncodeParameters(obj, version)
	if err != nil {
		r.err = err
		return r
	}
	for k, v := range params {
		if r.params == nil {
			r.params = make(url.Values)
		}
		r.params[k] = append(r.params[k], v...)
	}
	return r
}
```

## Body
设置请求体，根据请求体类型按照如下规则处理
1. 如果是string，将string作为路径读取文件
2. 如果是[]byte，直接使用
3. 如果是io.Reader，直接使用
4. 如果是runtime.Object，将其编码为Content-Type后使用
```go
// k8s.io/client-go/rest/request.go
func (r *Request) Body(obj interface{}) *Request {
	if r.err != nil {
		return r
	}
	switch t := obj.(type) {
	case string:
		data, err := ioutil.ReadFile(t)
		if err != nil {
			r.err = err
			return r
		}
		glogBody("Request Body", data)
		r.body = bytes.NewReader(data)
	case []byte:
		glogBody("Request Body", t)
		r.body = bytes.NewReader(t)
	case io.Reader:
		r.body = t
	case runtime.Object:
		// callers may pass typed interface pointers, therefore we must check nil with reflection
		if reflect.ValueOf(t).IsNil() {
			return r
		}
		encoder, err := r.c.content.Negotiator.Encoder(r.c.content.ContentType, nil)
		if err != nil {
			r.err = err
			return r
		}
		data, err := runtime.Encode(encoder, t)
		if err != nil {
			r.err = err
			return r
		}
		glogBody("Request Body", data)
		r.body = bytes.NewReader(data)
		r.SetHeader("Content-Type", r.c.content.ContentType)
	default:
		r.err = fmt.Errorf("unknown type used for body: %+v", obj)
	}
	return r
}
```

## Do
执行请求，返回Result对象
```go
// k8s.io/client-go/rest/request.go
func (r *Request) Do(ctx context.Context) Result {
	var result Result
	err := r.request(ctx, func(req *http.Request, resp *http.Response) {
		result = r.transformResponse(resp, req)
	})
	if err != nil {
		return Result{err: err}
	}
	return result
}

// 连接服务器，收到Response之后调用fn
func (r *Request) request(ctx context.Context, fn func(*http.Request, *http.Response)) error {
	//Metrics for total request latency
	start := time.Now()
	defer func() {
		metrics.RequestLatency.Observe(ctx, r.verb, r.finalURLTemplate(), time.Since(start))
	}()

	if r.err != nil {
		klog.V(4).Infof("Error in request: %v", r.err)
		return r.err
	}

	// 预检操作，检测命名空间的设置是否合法
	if err := r.requestPreflightCheck(); err != nil {
		return err
	}

	client := r.c.Client
	if client == nil {
		// 如果client是nil，则使用一个空的http.Client
		client = http.DefaultClient
	}

	// Throttle the first try before setting up the timeout configured on the
	// client. We don't want a throttled client to return timeouts to callers
	// before it makes a single request.
	// 在设置超时时间之前，先等待限流策略
	if err := r.tryThrottle(ctx); err != nil {
		return err
	}

	if r.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, r.timeout)
		defer cancel()
	}

	// Right now we make about ten retry attempts if we get a Retry-After response.
	var retryAfter *RetryAfter
	for {
		// 创建标注库的http请求对象
		req, err := r.newHTTPRequest(ctx)
		if err != nil {
			return err
		}

		// 进行重试
		r.backoff.Sleep(r.backoff.CalculateBackoff(r.URL()))
		if retryAfter != nil {
			// We are retrying the request that we already send to apiserver
			// at least once before.
			// This request should also be throttled with the client-internal rate limiter.
			if err := r.tryThrottleWithInfo(ctx, retryAfter.Reason); err != nil {
				return err
			}
			retryAfter = nil
		}
		// 获取http.Response对象
		resp, err := client.Do(req)
		updateURLMetrics(ctx, r, resp, err)
		if err != nil {
			r.backoff.UpdateBackoff(r.URL(), err, 0)
		} else {
			r.backoff.UpdateBackoff(r.URL(), err, resp.StatusCode)
		}

		done := func() bool {
			defer readAndCloseResponseBody(resp)

			// if the the server returns an error in err, the response will be nil.
			f := func(req *http.Request, resp *http.Response) {
				if resp == nil {
					return
				}
				fn(req, resp)
			}

			var retry bool
			retryAfter, retry = r.retry.NextRetry(req, resp, err, func(req *http.Request, err error) bool {
				// "Connection reset by peer" or "apiserver is shutting down" are usually a transient errors.
				// Thus in case of "GET" operations, we simply retry it.
				// We are not automatically retrying "write" operations, as they are not idempotent.
				if r.verb != "GET" {
					return false
				}
				// For connection errors and apiserver shutdown errors retry.
				// 如果是以下错误，不进行重试
				if net.IsConnectionReset(err) || net.IsProbableEOF(err) {
					return true
				}
				return false
			})
			if retry {
				// 等待到下次重试时间
				err := r.retry.BeforeNextRetry(ctx, r.backoff, retryAfter, req.URL.String(), r.body)
				if err == nil {
					return false
				}
				klog.V(4).Infof("Could not retry request - %v", err)
			}

			f(req, resp)
			return true
		}()
		if done {
			return err
		}
	}
}

// 解码响应
func (r *Request) transformResponse(resp *http.Response, req *http.Request) Result {
	var body []byte
	// 先把响应读出来
	if resp.Body != nil {
		data, err := ioutil.ReadAll(resp.Body)
		switch err.(type) {
		case nil:
			body = data
		case http2.StreamError:
			// This is trying to catch the scenario that the server may close the connection when sending the
			// response body. This can be caused by server timeout due to a slow network connection.
			// TODO: Add test for this. Steps may be:
			// 1. client-go (or kubectl) sends a GET request.
			// 2. Apiserver sends back the headers and then part of the body
			// 3. Apiserver closes connection.
			// 4. client-go should catch this and return an error.
			klog.V(2).Infof("Stream error %#v when reading response body, may be caused by closed connection.", err)
			streamErr := fmt.Errorf("stream error when reading response body, may be caused by closed connection. Please retry. Original error: %w", err)
			return Result{
				err: streamErr,
			}
		default:
			klog.Errorf("Unexpected error when reading response body: %v", err)
			unexpectedErr := fmt.Errorf("unexpected error when reading response body. Please retry. Original error: %w", err)
			return Result{
				err: unexpectedErr,
			}
		}
	}

	glogBody("Response Body", body)

	// verify the content type is accurate
	var decoder runtime.Decoder
	contentType := resp.Header.Get("Content-Type")
	if len(contentType) == 0 {
		contentType = r.c.content.ContentType
	}
	if len(contentType) > 0 {
		var err error
		mediaType, params, err := mime.ParseMediaType(contentType)
		if err != nil {
			return Result{err: errors.NewInternalError(err)}
		}
		// 协商出一个decoder，如果失败了，当作unstructed对象处理
		decoder, err = r.c.content.Negotiator.Decoder(mediaType, params)
		if err != nil {
			// if we fail to negotiate a decoder, treat this as an unstructured error
			switch {
			case resp.StatusCode == http.StatusSwitchingProtocols:
				// no-op, we've been upgraded
			case resp.StatusCode < http.StatusOK || resp.StatusCode > http.StatusPartialContent:
				return Result{err: r.transformUnstructuredResponseError(resp, req, body)}
			}
			return Result{
				body:        body,
				contentType: contentType,
				statusCode:  resp.StatusCode,
				warnings:    handleWarnings(resp.Header, r.warningHandler),
			}
		}
	}

	// 处理非200
	switch {
	case resp.StatusCode == http.StatusSwitchingProtocols:
		// no-op, we've been upgraded
	case resp.StatusCode < http.StatusOK || resp.StatusCode > http.StatusPartialContent:
		// calculate an unstructured error from the response which the Result object may use if the caller
		// did not return a structured error.
		retryAfter, _ := retryAfterSeconds(resp)
		err := r.newUnstructuredResponseError(body, isTextResponse(resp), resp.StatusCode, req.Method, retryAfter)
		return Result{
			body:        body,
			contentType: contentType,
			statusCode:  resp.StatusCode,
			decoder:     decoder,
			err:         err,
			warnings:    handleWarnings(resp.Header, r.warningHandler),
		}
	}

	// 构建Result并返回
	return Result{
		body:        body,
		contentType: contentType,
		statusCode:  resp.StatusCode,
		decoder:     decoder,
		warnings:    handleWarnings(resp.Header, r.warningHandler),
	}
}
```