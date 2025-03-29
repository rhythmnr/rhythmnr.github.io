---
title: 把swagger文档的访问放到路由中
date: 2022-12-13 15:14:00
categories:
- golang
---

**应该使用Raw函数而不是Exec函数，我尝试在连接clickhouse之后运行原始SQL语句，发现使用Exec会报错，但是用Raw的话就不会报错了，使用Exec报错似乎是在Scan调用的时候报错的**

**关于具体如何写，参考 **[https://gorm.io/docs/sql_builder.html](https://gorm.io/docs/sql_builder.html)

## 需求

需要增加一条单独的路由，通过该路由可以访问swagger格式的api文档。且支持配置该路由是否开启，默认开启，在生产环节中不开启。

PS：我项目使用的是go-zero框架。

## 解决步骤

收集了一些资料，首先发现了这种写法：

首先是生成的router文件：

```
func RegisterHandlers(server *rest.Server, serverCtx *svc.ServiceContext) {
	server.AddRoutes(
		[]rest.Route{
			{
				Method:  http.MethodGet,
				Path:    "/apiswagger",
				Handler: apiSwaggerHandler(serverCtx),
			},
}
```

```
func apiSwaggerHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fs := http.FileServer(http.Dir("./swaggerui"))
		fs.ServeHTTP(w, r)
	}
}
```

apiSwaggerHandler实现部分可以参考上面的写法，这样访问 `/apiswagger`路由的时候就会返回运行目录下的swaggerui目录下的index.html目录。测试访问了 `apiswagger`路由，在浏览器查看发现页面空白。但是postman可以显示index.html内容。

又继续收集了一些资料，尝试了另一种方法启动：

```
func main() {
	fs := http.FileServer(http.Dir("./apiswagger"))
	http.Handle("/", fs)

	err := http.ListenAndServe(":9000", nil)
	if err != nil {
		log.Fatal(err)
	}
}
```

发现这样写是可以正常显示页面的。

于是将index.html修改了一下，里面只有 `hello world`没有一些css和js的引用，发现可以正常显示。猜测也许go-zero只能显示单独的index.html，于是首先尝试将swaggerui目录下的所有css,js等文件集合到html中，发现集合起来太过复杂且文件很长很多，无法维护，最终也没有集合成功。

然后在浏览器打开开发者模式查看，发现获取css文件和js都失败了，猜测go zero可能对路由控制过于严格。

[![zdqWDO.png](https://s1.ax1x.com/2022/11/29/zdqWDO.png)](https://imgse.com/i/zdqWDO)

继续搜罗资料，发现go zero需要一个个路由的定义，改成了如下写法：

```
  // 这是main函数中
  server := rest.MustNewServer(c.RestConf)
	if c.SwaggerEnabled {
		staticFileHandler(server)
	}
```

```
func staticFileHandler(engine *rest.Server) {
	path := "apiswagger"
	rd, _ := ioutil.ReadDir(path)
	for _, f := range rd {
		filename := f.Name()
		path := "/apiswagger/" + filename
		engine.AddRoute(
			rest.Route{
				Method:  http.MethodGet,
				Path:    path,
				Handler: dirhandler("/apiswagger/", "apiswagger"),
			})
	}
}

func dirhandler(patern, filedir string) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		handler := http.StripPrefix(patern, http.FileServer(http.Dir(filedir)))
		handler.ServeHTTP(w, req)
	}
}
```

仍然在.api文件定义了swagger的路由，然后apiSwaggerHandler单独开启路由，发现这样可以访问 `apiswagger`路由可以成功显示页面，查看F12也可以看到成功获取到了css和js等文件

```
func apiSwaggerHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if svcCtx.Config.SwaggerEnabled {
			fs := http.FileServer(http.Dir("./apiswagger"))
			http.StripPrefix("/apiswagger/", fs).ServeHTTP(w, r)
		}
	}
}
```

## 注意事项

唯一需要注意的可能就是http.StripPrefix的用法，比如

```
http.StripPrefix("/tmpfiles/", http.FileServer(http.Dir("/tmp")))
```

的功能就是当访问路由是/tmpfiles开头的时候，比如访问了/tmpfiles/a，那么这个函数将会自动返回目录/tmp下a的数据，访问/tmpfiles/b路由时，将会自动返回目录/tmp下b的数据，这个函数在go开启静态服务器的时候很有用。

## 回顾

主要问题还在于对前端不熟悉吧，居然没第一时间想到要去看看css和js有没有成功被获取到。

## 参考

[Serving Static Sites with Go](https://www.alexedwards.net/blog/serving-static-sites-with-go)

[go-zero部署静态资源页](https://www.cnblogs.com/pangxiaox/p/16281197.html)

[Why do I need to use http.StripPrefix to access my static files?](https://stackoverflow.com/questions/27945310/why-do-i-need-to-use-http-stripprefix-to-access-my-static-files)
