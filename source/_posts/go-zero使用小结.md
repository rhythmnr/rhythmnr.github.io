---
title: go-zero使用小结
date: 2022-11-24 20:00:00
categories:
- golang
---

## 

## 开发规范

- 变量命名应尽量描述其内容，而不是类型
- 当变量名称在定义和最后一次使用之间的距离很短时，简短的名称看起来会更好。
- 在遇到for，if等循环或分支时，推荐单个字母命名来标识参数和返回值
- 文件名尽量小写，除unit test外避免下划线(_)
- **不可exported的必须首字母小写**
- **import 按照 `官方包`，NEW LINE，`当前工程包`，NEW LINE，`第三方依赖包`顺序引入**
- 避免下划线(_)接收error
- 建议一个block（比如if是一个block）结束空一行，如if、for等
- return前空一行

## 快速开始

直接执行：

```
goctl api new packet
go mod tidy
```

修改packet/internal/logic/packetlogic.go写具体的业务逻辑

```go
 return &types.Response{
        Message: "Hello go-zero",
    }, nil
```

运行：

```shell
 go run packet.go -f etc/packet-api.yaml 
```

GET http://localhost:8888/from/me 或者 GET http://localhost:8888/from/you 即可看到效果。

上面的是单体服务的创建，go-zero还提供了微服务的创建方法，微服务和单体服务最大的不同就是微服务之间会互相调用，即会用一些rpc调用。

> 微服务的创建暂时没有看

## 进阶指南

### 系统设计的一些建议

一般我们需要按照业务横向拆分，将一个系统拆分成多个子系统，每个子系统应拥有独立的持久化存储，缓存系统。

在设计系统时，尽量做到服务之间调用链是单向的，而非循环调用

### 进阶操作

官方提供的demo下载后的目录结构为，下面的操作基于该demo，而不是我自己自定义的服务：

````shell
├── README.md
├── common
├── go.mod
├── go.sum
└── service
    ├── search
    │   └── api
    └── user
        ├── api
        │   └── user.api
        ├── model
        │   └── user.sql
        └── rpc
            └── user.proto
````

#### models生成

```shell
$ cd service/user/model
$ goctl model mysql ddl -src user.sql -dir . -c
```

#### API编写

```shell
vim service/user/api/user.api
```

补充api的定义

```shell
$ cd service/user/api
$ goctl api go -api user.api -dir . 
```

#### 业务编码

**添加Mysql配置**

```shell
vim service/user/api/internal/config/config.go
```

声明Datasource和CacheRedis

````go
package config

import (
    "github.com/zeromicro/go-zero/rest"
    "github.com/zeromicro/go-zero/core/stores/cache"
    )


type Config struct {
    rest.RestConf
    Mysql struct{
        DataSource string
    }
  
    CacheRedis cache.CacheConf
}
````

**完善yaml配置：**

```shell
vim service/user/api/etc/user-api.yaml
```

添加和上面对应的Mysql和CacheRedis的配置

```yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Mysql:
  DataSource: root:xxx@tcp(127.0.0.1)/gozero?charset=utf8mb4&parseTime=true&loc=Asia%2FShanghai
CacheRedis:
  - Host: $host
    Pass: $pass
    Type: node
```

**完善服务依赖：**

```shell
$ vim service/user/api/internal/svc/servicecontext.go
```

根据上面的Mysql和Cache Redis的配置，初始化需要使用到的UserModel

```go
type ServiceContext struct {
    Config    config.Config
    UserModel model.UserModel
}

func NewServiceContext(c config.Config) *ServiceContext {
    conn:=sqlx.NewMysql(c.Mysql.DataSource)
    return &ServiceContext{
        Config: c,
        UserModel: model.NewUserModel(conn,c.CacheRedis),
    }
}
```

**填充登录逻辑：**

```shell
vim service/user/api/internal/logic/loginlogic.go
```

其中一段代码涉及到JWT鉴权的操作，需要参考下面的内容

#### jwt鉴权

> JSON Web令牌（JWT）是一个开放标准（RFC 7519）

go-zero中，jwt鉴权一般在api层使用，一般在登录时生成JWT，请求后续接口时验证JWT。

```shell
vim service/user/api/internal/config/config.go
```

添加JWT的配置：

```diff
type Config struct {
    rest.RestConf
    Mysql struct{
        DataSource string
    }
    CacheRedis cache.CacheConf
+    Auth      struct {
+       AccessSecret string
+       AccessExpire int64
+   }
}
```

```shell
vim service/user/api/etc/user-api.yaml
```

添加对应的yaml配置项：

````diff
Name: user-api
Host: 0.0.0.0
Port: 8888
...
+Auth:
+  AccessSecret: $AccessSecret
+  AccessExpire: 1111 # 单位 秒
````

完善《业务编码》那块的getJwtToken方法：

```go
func (l *LoginLogic) getJwtToken(secretKey string, iat, seconds, userId int64) (string, error) {....}
```

下面完善search服务的JWT相关内容：

```
touch service/search/api/search.api
```

添加search API的定义，需要指定

```api
@server(
    jwt: Auth
)
```

创建 service/search/api/etc/search-api.yaml 文件，添加相关配置。

下面验证 jwt token

```shell
$ cd service/user/api
$ go run user.go -f etc/user-api.yaml
```

请求POST http://127.0.0.1:8888/user/login 获取JWT。

然后生成search的代码，启动search服务：

```shell
cd service/search/api
go run search.go -f etc/search-api.yaml
```

 GET http://127.0.0.1:8889/search/do?name=xxx 的 headers 加上 Authorization 才可以成功验证通过，否则会报 401Unauthorized。

#### 中间件使用

**路由中间件**

在search.api添加中间件声明：

```diff
@server(
    jwt: Auth
+   middleware: Example // 路由中间件声明
)
```

`internal`目录下多一个 `middleware`的目录

service/search/api/internal/svc/servicecontext.go添加Middleware

```diff
type ServiceContext struct {
    Config config.Confi
+   Example rest.Middleware
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config: c,
+       Example: middleware.NewExampleMiddleware().Handle,
    }
}
```

service/search/api/internal/middleware/examplemiddleware.go添加中间件详细功能：

```diff
package middleware

import "net/http"

type ExampleMiddleware struct {
}

func NewExampleMiddleware() *ExampleMiddleware {
        return &ExampleMiddleware{}
}

func (m *ExampleMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
+    logx.Info("example middle")
    return func(w http.ResponseWriter, r *http.Request) {
        // TODO generate middleware implement function, delete after code implementation
        // Passthrough to next handler if need
        next(w, r)
    }
}
```

**全局中间件**

通过rest.Server提供的Use方法即可

```diff
func main() {
    flag.Parse()

    var c config.Config
    conf.MustLoad(*configFile, &c)

    ctx := svc.NewServiceContext(c)
    server := rest.MustNewServer(c.RestConf)
    defer server.Stop()

    // 全局中间件
+    server.Use(func(next http.HandlerFunc) http.HandlerFunc {
+        return func(w http.ResponseWriter, r *http.Request) {
+            logx.Info("global middleware")
+            next(w, r)
+        }
+    })
    handler.RegisterHandlers(server, ctx)

    fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
    server.Start()
}
```

#### 错误处理

错误处理模块是将原来以text返回的错误包装了一下，以json返回了

在平时的业务开发中，我们可以认为http状态码不为 `2xx`系列的，都可以认为是http请求错误， 并伴随响应的错误信息，但这些错误信息都是以plain text形式返回的。

- 业务处理正常

  ```json
  {
    "code": 0,
    "msg": "successful",
    "data": {
      ....
    }
  }
  ```
- 业务处理异常

  ```json
  {
    "code": 10001,
    "msg": "参数错误"
  }
  ```

  之前的错误都是以plain text的格式返回的，如果想以json格式返回，可以在common中添加一个 `errorx/baseerror.go`文件，**注意go-zero这里给出的例子中的文件名是直接将多个单词连接在一起的。**

  **注意这个包叫errorx**

  ```go
  package errorx
  
  const defaultCode = 1001
  
  type CodeError struct {
      Code int    `json:"code"`
      Msg  string `json:"msg"`
  }
  
  type CodeErrorResponse struct {
      Code int    `json:"code"`
      Msg  string `json:"msg"`
  }
  
  func NewCodeError(code int, msg string) error {
      return &CodeError{Code: code, Msg: msg}
  }
  
  func NewDefaultError(msg string) error {
      return NewCodeError(defaultCode, msg)
  }
  
  func (e *CodeError) Error() string {
      return e.Msg
  }
  
  func (e *CodeError) Data() *CodeErrorResponse {
      return &CodeErrorResponse{
          Code: e.Code,
          Msg:  e.Msg,
      }
  }
  ```

将service/user/api/internal/logic/loginlogic.go里的错误返回部分都改成封装的错误：

```diff
    case model.ErrNotFound:
-        return nil, error.New("用户名不存在")
+        return nil, errorx.NewDefaultError("用户名不存在")
```

此时请求后返回的错误内容为：

![WX20220701-094754@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h3r6pqndjcj31io07etal.jpg)

然后在service/user/api/user.go的main函数开启自定义错误：

````diff
func main() {
    handler.RegisterHandlers(server, ctx)

+    // 自定义错误
+    httpx.SetErrorHandler(func(err error) (int, interface{}) {
+        switch e := err.(type) {
+        case *errorx.CodeError:
+            return http.StatusOK, e.Data()
+        default:
+            return http.StatusInternalServerError, nil
+        }
+    })
    ....
}
````

![WX20220701-095057@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h3r6szc1u5j31io0aogo2.jpg)

此时Status为200而不是之前的400

#### 模板修改

模板修改模块是实现无论是否出错了，都返回统一的模板

```json
{
  "code": 0,
  "msg": "OK",
  "data": {}
}
```

主要是为了实现在正常和错误的情况下返回的数据格式是一样的，《错误处理》那块改完了，在正常情况下返回的数据是这样的：

```json
{
    "id": 1,
    "name": "小明",
    "gender": "男",
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTY2NDYzNjEsImlhdCI6MTY1NjY0MDM2MSwidXNlcklkIjoxfQ.yhiPdecQFnBjjPnRLWEpWIeoI5sZPuDJxaZAQlO2-R0",
    "accessExpire": 1656646361,
    "refreshAfter": 1656643361
}
```

是没有code和msg字段的，正常和错误情况还是不统一。

现在要统一成如下格式：

```json
{
  "code": 0,
  "msg": "OK",
  "data": {} // 实际响应数据
}
```

下面使用我之前定义的packet服务，创建 `packet/response/response.go`文件，指定Response包装形式，代码如下：

```go
package response

import (
    "net/http"

    "github.com/zeromicro/go-zero/rest/httpx"
)

type Body struct {
    Code int         `json:"code"`
    Msg  string      `json:"msg"`
    Data interface{} `json:"data,omitempty"`
}

func Response(w http.ResponseWriter, resp interface{}, err error) {
    var body Body
    if err != nil {
        body.Code = -1
        body.Msg = err.Error()
    } else {
        body.Msg = "OK"
        body.Data = resp
    }
    httpx.OkJson(w, body)
}
```

然后修改handler模板，我的电脑是这个： `~/.goctl/1.3.5/api/handler.tpl`

修改之前的

```go
package {{.PkgName}}

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
	{{.ImportPackages}}
)

func {{.HandlerName}}(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		{{if .HasRequest}}var req types.{{.RequestType}}
		if err := httpx.Parse(r, &req); err != nil {
			httpx.Error(w, err)
			return
		}

		{{end}}l := {{.LogicName}}.New{{.LogicType}}(r.Context(), svcCtx)
		{{if .HasResp}}resp, {{end}}err := l.{{.Call}}({{if .HasRequest}}&req{{end}})
		if err != nil {
			httpx.Error(w, err)
		} else {
			{{if .HasResp}}httpx.OkJson(w, resp){{else}}httpx.Ok(w){{end}}
		}
	}
}
```

改成

```tpl
package handler

import (
    "net/http"
     "github.com/repo/hello/response"// 这里要写response所在包！！也就是response在项目里的包。。。。。。。。。。。。。
     // 这个要看在哪个目录下执行goctl，这里会用到项目hello/response目录里的Response方法
    {% raw %}
    {{.ImportPackages}}
    {% endraw %}
)

{% raw %}
func {{.HandlerName}}(ctx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        {{if .HasRequest}}var req types.{{.RequestType}}
        if err := httpx.Parse(r, &req); err != nil {
            httpx.Error(w, err)
            return
        }{{end}}

        l := logic.New{{.LogicType}}(r.Context(), ctx)
        {{if .HasResp}}resp, {{end}}err := l.{{.Call}}({{if .HasRequest}}req{{end}})
        {{if .HasResp}}response.Response(w, resp, err){{else}}response.Response(w, nil, err){{end}}
        // 就是这里用到了response.Response方法
    }
}
{% endraw %}
```

修改handler

```diff
func queryHandler(ctx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.Request
        if err := httpx.Parse(r, &req); err != nil {
            httpx.Error(w, err)
            return
        }

        l := logic.NewGreetLogic(r.Context(), ctx)
        resp, err := l.Greet(req)
-       if err != nil {
-            httpx.Error(w, err)
-        } else {
-            httpx.OkJson(w, resp)
-        }
+        response.Response(w, resp, err)
    }
}
```

重启服务：

```
{
    "code": 0,
    "msg": "OK",
    "data": {
        "pagination": {
            "current_page": 1,
            "page_size": 100,
            "max_page": 3
        },
        "data": []
}
```

```json
{
    "code": -1,
    "msg": "参数错误"
}
```

修改完毕后，如果修改api文件再次执行goctl，那么就可以直接生成新的已经包装好的代码了。

#### 日志优化

默认的日志会在终端打印很多我可能用不到的东西：

````json
{"@timestamp":"2022-10-21T15:31:47.554+08:00","caller":"stat/usage.go:61","content":"CPU: 0m, MEMORY: Alloc=3.4Mi, TotalAlloc=6.7Mi, Sys=16.1Mi, NumGC=5","level":"stat"}
{"@timestamp":"2022-10-21T15:31:47.620+08:00","caller":"load/sheddingstat.go:61","content":"(api) shedding_stat [1m], cpu: 0, total: 0, pass: 0, drop: 0","level":"stat"}
{"@timestamp":"2022-10-21T15:31:56.875+08:00","caller":"stat/metrics.go:210","content":"(screen-api) - qps: 0.0/s, drops: 0, avg time: 0.0ms, med: 0.0ms, 90th: 0.0ms, 99th: 0.0ms, 99.9th: 0.0ms","level":"stat"}
````

目前需求是不打印这些用不到的信息，把日志输出到文件而不是终端。

在main函数中执行 `logx.DisableStat()`，可以不打印这些stat信息。

在main函数只执行logx.SetLevel(0)，可以打印出Info Error Slow日志，如果手动打印.Error .Info之类的话

Ps：go-zero里关于level的定义：

```go
const (
	// InfoLevel logs everything
	InfoLevel uint32 = iota
	// ErrorLevel includes errors, slows, stacks
	ErrorLevel
	// SevereLevel only log severe messages
	SevereLevel
)
```

[参考](https://go-zero.dev/cn/docs/component/logx)并总结了更好的写法：

在main一开始就对logx初始化，如果在之后初始化的话好像配置会被覆盖。我之前在RegisterHandlers之后给logx初始化，发现logx的配置不生效，指定了写到文件里也没有写到文件而是输出到了终端。

```go
func main() {
	flag.Parse()

	logx.DisableStat()
	logConf := logx.LogConf{
		ServiceName: "screen",
		Mode:        "file",
		Path:        "screen_logs",
		Encoding:    "json",
		Level:       "info",
		KeepDays:    3,
	}
	logx.MustSetup(logConf)
	defer logx.Close()
```

这里的配置是将日志写到了文件里，screen_logs目录下，screen_logs目录会在运行程序的同级目录下生成

## TODO

服务监控这块基于prometheus，可以看看

go-zero链路追踪，可以看看

---

参考

- [官方文档](https://go-zero.dev/cn/docs/develop/naming-spec)
