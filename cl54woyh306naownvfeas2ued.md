## gin 中如何优雅的流转各种公共实体

# gin 中如何优雅的流转

在一个`golang`交流群中有一位老哥是这么提问的：

> ![faf14250f338da8b14177f88329309e](https://user-images.githubusercontent.com/65269574/177026447-1b625c1a-99f3-4549-9447-dd068a333dae.jpg)
> 像我这种文件结构，如果要在user.go或者role.go中使用db，应该以什么方式传递进去呢？


**解决方案**

由于我最近使用`go-zero`比较多，所以我就用`go-zero`的解决思路来实现一下

1.首先有一个`serviceContext`来保存`config`,这个`config`就是当前项目所有需要的配置信息
2.然后有`serviceContext`中还有其他本项目所有的一些公共的组件，比如`db`,`cache`,`logger`等等

```go
package svc

import (
	"gin-showcase/config"
	"gin-showcase/mysql"
	"gin-showcase/redis"
)

type (
	ServiceContext struct {
		c     config.Config
		mysql mysql.Conn
		redis redis.Conn
	}
)

func NewServiceContext(c config.Config) *ServiceContext {
	return &ServiceContext{
		C: c,
		Mysql: mysql.Conn{
			Url: c.MysqlUrl,
		}.Open(),
		Redis: redis.Conn{
			Url: c.RedisUrl,
		}.Open(),
	}
}
```

3.简易的一个使用流程

```go
package main

import (
	"fmt"
	"gin-showcase/config"
	"gin-showcase/svc"
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	fmt.Println("Hello, World!")

	c := config.Config{
		MysqlUrl: "mysql://root:root@localhost:3306/test",
		RedisUrl: "redis://localhost:6379",
	}
	e := gin.Default()

	context := svc.NewServiceContext(c)
	defer context.Mysql.Close()
	defer context.Redis.Close()

	e.GET(Ping(context))
}

func Ping(svContext *svc.ServiceContext) (string, gin.HandlerFunc) {
	return "ping", func(c *gin.Context) {
		c.String(http.StatusOK, svContext.Redis.Get("ping"))
	}
}
```

总结起来就是把所有需要用到的一些公共组件都挂载到`serviceContext`中，然后在每个组件中都可以直接使用`serviceContext`中的组件.

代码地址： https://github.com/fzdwx/gin-showcase