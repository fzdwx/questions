## 浅析 flamego的动态注入

#1. 如何实现动态注入？

flamego的handle设计很巧妙,它的`Handler`的类型是一个`interface{}`的定义
```go
type Handler interface{}
```
所以它可以支持以下的添加Handler的方法：
```go
f.Get("/", func() string {
	return "Hello, Flamego!"
})

f.Get("/ping", func(c flamego.Context) string {
	return "hello world"
})
```

那么它是在什么时候注入的？有两种:
1.  在服务启动时，有一个默认的`type`匹配
2. 在方法调用时进行注入

## 1.默认匹配
可以直接看https://github.com/flamego/flamego/blob/main/router.go 里面的`Route`的这个方法的实现:
```go
func (r *router) Route(method, routePath string, handlers []Handler) *Route {
    // ... 省略
    
    // 就是在这里进行默认匹配的
	validateAndWrapHandlers(handlers, r.handlerWrapper)

	return r.addRoute(method, routePath, func(w http.ResponseWriter, req *http.Request, params route.Params) {
		r.contextCreator(w, req, params, handlers, r.URLPath).run()
	})
}
```
最后会调用到这个方法
```go
func validateAndWrapHandler(h Handler, wrapper func(Handler) Handler) Handler {
    // ... 省略

	switch v := h.(type) {
	case func(Context):
		return ContextInvoker(v)
	case func(http.ResponseWriter, *http.Request):
		return httpHandlerFuncInvoker(v)
	case http.HandlerFunc:
		return httpHandlerFuncInvoker(v)
	case func() (int, string):
		return teapotInvoker(v)
	}

	if wrapper != nil {
		h = wrapper(h)
	}
	return h
}

```
如果我们写一个这样的路由,他就会走第一个匹配:
```go
f.Get("/test", func(ctx flamego.Context) {
		fmt.Println("test")
})
```

## 2.而动态注入呢？
还是`router.go#Route`的这个方法:
```go
func (r *router) Route(method, routePath string, handlers []Handler) *Route {
	// ... 省略

	return r.addRoute(method, routePath, func(w http.ResponseWriter, req *http.Request, params route.Params) {
        // 具体都在`run`这个方法里面
		r.contextCreator(w, req, params, handlers, r.URLPath).run()
	})
}
```
接着看:
```go
func (c *context) run() {
        // ... 省略

		vals, err := c.Invoke(h)

        // ... 省略
	}
}
```
最后会调用`callInvoke`这个方法,
```go
func (inj *injector) callInvoke(f interface{}, t reflect.Type, numIn int) ([]reflect.Value, error) {
	var in []reflect.Value
	if numIn > 0 {
		in = make([]reflect.Value, numIn)
		var argType reflect.Type
		var val reflect.Value
		for i := 0; i < numIn; i++ {
			argType = t.In(i)
            // 这里是关键
			val = inj.Value(argType)
			if !val.IsValid() {
				return nil, fmt.Errorf("value not found for type %v", argType)
			}

			in[i] = val
		}
	}
	return reflect.ValueOf(f).Call(in), nil

```
那么`inj.Value(argType)`里面是怎么实现的呢？
它里面其实就是一个`map`,保存了类型以及对应类型的实例。
```go
func (inj *injector) Value(t reflect.Type) reflect.Value {
	val := inj.values[t]

	// ... 省略 校验val 以及返回val
}
```

# 2.如何注入自定义的实例？
`flamego`提供了一个`Map`方法,可以添加到`inj.values`里面去:

```go
type P struct {
	name string
}

f.Map(P{name: "fzdwx"})

f.Get("/test", func(ctx flamego.Context, p P) {
	fmt.Println(p.name)
})
```
日志:
```output
[Flamego] Listening on 0.0.0.0:2830 (development)
[Flamego] 2022-07-14 18:29:42: Started GET /test for [::1]
fzdwx
[Flamego] 2022-07-14 18:29:45: Completed GET /test 0  in 3.1801852s
```


---
如有说的不对的，欢迎批评指正！