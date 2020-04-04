# gin的中间件: 认证、鉴权、限流、缓存的实现


- 0.0 有了token/cookie的认证方式，就可以采用user/guest的鉴权中间件，设定每个路由入口的用户等级的访问限制。

- 代码library/net/http/blademaster/routergroup.go
```go
// GET is a shortcut for router.Handle("GET", path, handle).
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle("GET", relativePath, handlers...)
}
```

- 代码library/net/http/blademaster/server.go
```go
// NewServer returns a new blank Engine instance without any middleware attached.
func NewServer(conf *ServerConfig) *Engine {
	engine := &Engine{
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		address:       ip.InternalIP(),
		mux:           http.NewServeMux(),
		metastore:     make(map[string]map[string]interface{}),
		methodConfigs: make(map[string]*MethodConfig),
	}
	return engine
}

```

- 代码library/net/http/blademaster/middleware/auth/auth_test.go
```go
func create() *Auth {
	return New(&Config{
		Identify:    &warden.ClientConfig{},
		DisableCSRF: false,
	})
}

func engine() *bm.Engine {
	e := bm.NewServer(nil)
	authn := create()
	e.GET("/user", authn.User, func(ctx *bm.Context) {
		mid, _ := ctx.Get("mid")
		ctx.JSON(fmt.Sprintf("%d", mid), nil)
	})
	e.GET("/metadata/user", authn.User, func(ctx *bm.Context) {
		mid := metadata.Value(ctx, metadata.Mid)
		ctx.JSON(fmt.Sprintf("%d", mid.(int64)), nil)
	})
	return e
}    
```

- 代码library/net/http/blademaster/middleware/auth/auth.go
```go
// User is used to mark path as access required.
// If `access_key` is exist in request form, it will using mobile access policy.
// Otherwise to web access policy.
func (a *Auth) User(ctx *bm.Context) {
	req := ctx.Request
	if req.Form.Get("access_key") == "" {
		a.UserWeb(ctx)
		return
	}
	a.UserMobile(ctx)
}

// UserWeb is used to mark path as web access required.
func (a *Auth) UserWeb(ctx *bm.Context) {
	a.midAuth(ctx, a.AuthCookie)
}

func (a *Auth) midAuth(ctx *bm.Context, auth authFunc) {
	mid, err := auth(ctx)
	if err != nil {
		ctx.JSON(nil, err)
		ctx.Abort()
		return
	}
	setMid(ctx, mid)
}

// set mid into context
// NOTE: This method is not thread safe.
func setMid(ctx *bm.Context, mid int64) {
	ctx.Set("mid", mid)
	if md, ok := metadata.FromContext(ctx); ok {
		md[metadata.Mid] = mid
		return
	}
}

```

- 0.1 限流中间件，用到了 library/rate/limite 库对最近的一段时间的访问成功率的统计。
- library/net/http/blademaster/middleware/limit/aqm/aqm_test.go
```go
func TestAQM(t *testing.T) {
	var group sync.WaitGroup
	rand.Seed(time.Now().Unix())
	eng := bm.Default()
	router := eng.Use(New(nil).Limit())
	router.GET("/aqm", testaqm)
	go eng.Run(":9999")
    group.Wait()
}

func testaqm(ctx *bm.Context) {
	count := rand.Intn(100)
	time.Sleep(time.Millisecond * time.Duration(count))
}
```

- library/net/http/blademaster/middleware/limit/aqm/aqm.go
```go
// Limit return a bm handler func.
func (a *AQM) Limit() bm.HandlerFunc {
	return func(c *bm.Context) {
		done, err := a.limiter.Allow(c)
		if err != nil {
			stats.Incr(_family, c.Request.URL.Path[1:])
			// TODO: priority request.
			// c.JSON(nil, err)
			// c.Abort()
			return
		}
		defer func() {
			if c.Error != nil && !ecode.Deadline.Equal(c.Error) && c.Err() != context.DeadlineExceeded {
				done(rate.Ignore)
				return
			}
			done(rate.Success)
		}()
		c.Next()
	}
}
```

- 0.2 缓存中间件，可以采用文件缓存和 memcache 缓存中任何一种，都具有 Get 和 Set 方法。文件缓存的每一个对象，都会有一个独立的tmp文件保存缓存信息，文件缓存并不提供过期(expire)的方法。

- library/net/http/blademaster/middleware/cache/cache.go
```go
// Cache is used to mark path as customized cache policy
func (c *Cache) Cache(policy Policy, filter Filter) bm.HandlerFunc {
	return func(ctx *bm.Context) {
		if filter != nil && !filter(ctx) {
			return
		}
		policy.Handler(c.store)(ctx)
	}
}
```