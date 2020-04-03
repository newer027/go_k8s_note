# opentracing 和熔断的原理和go语言实现

opentracing 的核心原理，是运用 http.RoundTripper 将微服务对外API 的 http 请求的根 span 注入到请求的头部，该API在微服务集群内部请求，也带有相同的头部信息。这样各端中的 span 可以合并到一个追踪过程中。
以下通过分析bilibili开源的kratos，了解熔断的go语言实现方法。

- 创建一个 toml 文件，在其中填写 http client熔断的条件参数。通过 toml.DecodeFile 解析 toml 文件，通过解析到的 struct 构建 client ，并采用 client 发起 POST 请求。
- 代码0.0 

client_config.toml
```
[HttpClientConfig]
    key = "key"
    secret = "secret"
    dial = "100ms"
    timeout = "10s"
    keepAlive = "60s"
    timer = 1024
    [HttpClientConfig.breaker]
        window  = "3s"
        sleep   = "100ms"
        bucket  = 10
        ratio   = 1.0
        request = 100
```
```
type Conf struct {
    HTTPClientConfig *blademaster.ClientConfig
}

confPath := "client_config.toml"
c := new(Conf)
_, err = toml.DecodeFile(confPath, c)
httpClient := bm.NewClient(c.HTTPClientConfig)
// httpClient.Post(c context.Context, uri, ip string, params url.Values, res interface{}) (err error)中调用 client.Do(c, req, res)，该函数在0.0.0.1部分会详细讲解
err := httpClient.Post(c, d.c.AccRecover.UpPwdURL, metadata.String(c, metadata.RemoteIP), params, &res) 
```

- bm.NewClient 的实现如下。在 client 结构体中包含 http.Client，http.Client将http的标签关联到头部信息。
- 代码0.0.0
pkg/net/http/blademaster/client.go

```
// NewClient new a http client.
func NewClient(c *ClientConfig) *Client {
	client := new(Client)

	originTransport := &http.Transport{
		DialContext:     client.dialer.DialContext,
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}

	// wraps RoundTripper for tracer
	client.transport = &TraceTransport{RoundTripper: originTransport}
	client.client = &http.Client{
		Transport: client.transport,
	}
	client.breaker = breaker.NewGroup(c.Breaker)
	return client
}
```
- http的标签包括http和rpc的区分，包括GET/POST/PUT等请求方法的区分，包括请求URL的格式化字符串，包括client/server的区分，包括微服务的名称。
- 代码0.0.0.0
library/net/http/blademaster/trace.go

```
// TraceTransport 里包含 RoundTripper。 If a request is being traced with
// Tracer, Transport will inject the current span into the headers,
// and set HTTP related tags on the span.
type TraceTransport struct {
	peerService  string
	internalTags []trace.Tag
	// The actual RoundTripper to use for the request. A nil
	// RoundTripper defaults to http.DefaultTransport.
	http.RoundTripper
}

func (t *TraceTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	rt := t.RoundTripper
	if rt == nil {
		rt = http.DefaultTransport
	}
	tr, ok := trace.FromContext(req.Context())
	if !ok {
		return rt.RoundTrip(req)
	}
	operationName := "HTTP:" + req.Method
	// fork new trace
	tr = tr.Fork("", operationName)
    // http和rpc的区分
	tr.SetTag(trace.TagString(trace.TagComponent, _defaultComponentName))
	tr.SetTag(trace.TagString(trace.TagHTTPMethod, req.Method))
	tr.SetTag(trace.TagString(trace.TagHTTPURL, req.URL.String()))
	tr.SetTag(trace.TagString(trace.TagSpanKind, "client"))
    // peerService 是微服务的名称
	if t.peerService != "" {
		tr.SetTag(trace.TagString(trace.TagPeerService, t.peerService))
	}
	tr.SetTag(t.internalTags...)

	// 将 trace 注入到 http header
	trace.Inject(tr, trace.HTTPFormat, req.Header)
}
```

- 在启动 tracer 组件时，通过 SetGlobalTracer 设置 Tracer 结构体，_tracer.Inject(t, format, carrier) 中 format表示http/rpc的区分，carrier表示http 头部。
- 代码0.0.0.0.0
library/net/trace/tracer.go

```
// SetGlobalTracer SetGlobalTracer
func SetGlobalTracer(tracer Tracer) {
	_tracer = tracer
}

// Inject takes the Trace instance and injects it for
// propagation within `carrier`. The actual type of `carrier` depends on
// the value of `format`.
func Inject(t Trace, format interface{}, carrier interface{}) error {
	return _tracer.Inject(t, format, carrier)
}
```

- 以下代码创建 httpPropagator 结构体，并调用其 Inject 方法。
- 代码0.0.0.0.1
library/net/trace/dapper.go

```
func (d *dapper) Inject(t Trace, format interface{}, carrier interface{}) error {
	// if carrier implement Carrier use direct, ignore format
	carr, ok := carrier.(Carrier)
	if ok {
		t.Visit(carr.Set)
		return nil
	}
	// use Built-in propagators
	pp, ok := d.propagators[format]
	if !ok {
		return ErrUnsupportedFormat
	}
	carr, err := pp.Inject(carrier)
	if err != nil {
		return err
	}
	if t != nil {
		t.Visit(carr.Set)
	}
	return nil
}
```

- httpPropagator 的Inject 方法，通过 httpCarrier(header)，让 httpCarrier 接口拥有了 http.Header 对键值对 Get/Set 的调用方法。
- 代码0.0.0.0.2
library/net/trace/propagation.go

```
type httpCarrier http.Header

func (h httpCarrier) Set(key, val string) {
	http.Header(h).Set(key, val)
}

func (h httpCarrier) Get(key string) string {
	return http.Header(h).Get(key)
}

func (httpPropagator) Inject(carrier interface{}) (Carrier, error) {
	header, ok := carrier.(http.Header)
	if !ok {
		return nil, ErrInvalidCarrier
	}
	if header == nil {
		return nil, ErrInvalidTrace
	}
	return httpCarrier(header), nil
}
```

- client.Post 函数调用 client.Do，间接调用 client.Raw。
- 代码0.0.0.1

```
// 请求得到的响应格式为 []byte。 client.Do(c, req, res) 会调用 Raw（c, req, v），通过json.Unmarshal()获取json格式内容。
func (client *Client) Raw(c context.Context, req *http.Request, v ...string) (bs []byte, err error) {
    brk := client.breaker.Get(uri)
    if err = brk.Allow(); err != nil {
		code = "breaker"
		return
	}
	defer client.onBreaker(brk, &err) //发送请求失败，服务端5**错误，或读取响应内容失败
	if resp, err = client.client.Do(req); err != nil {
        return
	}
	return
}
```

- 创建属于该 uri 的熔断器。
- 代码0.0.0.1.0

```
func (g *Group) Get(key string) Breaker {
	brk = newBreaker(conf)
	g.mu.Lock()
	if _, ok = g.brks[key]; !ok {
		g.brks[key] = brk
	}
	g.mu.Unlock()
	return brk
}

- 熔断器统计单位时间内的请求总数和成功请求数，并判断当前请求是否熔断。
- 代码0.0.0.1.1

```
func (b *sreBreaker) Allow() error {
	success, total := b.summary()
    // 请求失败多于1/3，而且请求数大于配置的最大请求数目，有可能返回错误。返回错误的比例和请求失败比例正相关。
}
```





