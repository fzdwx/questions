## Go框架之go-zero

# 前言
今天在重写[burst](github.com/fzdwx/burst)，准备用`Golang`直接重写服务端和客户端，因为在写
[加密访问](https://github.com/fzdwx/burst/issues/21)这需求的时候，我发现如果客户端也实现类似服务端的效果，那么我为什么不直接全部重写呢？

然后就调研了几款`Golang`的Web框架
1. gin
2. go-zero

为什么选择了`go-zero`？
1. 主要文档比较友好
2. 有一个很好的开发辅助工具`goctl`
3. 对容器的支持友好

# 一个demo
1. 首先创建一个API服务
```
goctl api new burst
cd burst
```
它会在这个demo中自动添加一个路由:`/from/:name`
2. 添加逻辑
在`internal/logic/burstlogic.go`这个文件中
```go
func (l *BurstLogic) Burst(req *types.Request) (resp *types.Response, err error) {
	return &types.Response{Message: "hello " + req.Name}, nil
}
```
3. 访问测试
```bash
curl -i -X GET http://localhost:8888/from/you 
```
结果:

  ```
  HTTP/1.1 200 OK
  Content-Type: application/json; charset=utf-8
  Traceparent: 00-9d7e3beb747f1cd93428691e049f1097-78d71cd2d615ab52-00
  Date: Sat, 25 Jun 2022 16:28:28 GMT
  Content-Length: 23
  
  {"message":"hello you"}
  
  ```
**注意** 这个参数只能是 `you|me`,因为在`./internal/types/types.go `中对`Requst`的`Name`字段做了一个校验.
```go
type Request struct {
	Name string `path:"name,options=you|me"`
}
```

# 结束
一个简单的demo就到这里。