---
title: go zero使用记录
date: 2022-10-22 13:24:00
categories:
- golang
---
## 格式化api文件

```go
 goctl api format --dir=.  
```

## 根据.api文件生成swagger

安装：

```shell
GOPROXY=https://goproxy.cn/,direct go install github.com/zeromicro/goctl-swagger@latest
```

生成文件：

```shell
~/mylocalfile/aWork/code/flora-gopacket-api >> goctl api plugin -plugin goctl-swagger="swagger -filename packet.json" -api service/packet/api/packet.api -dir doc
```

生成json文件的swagger文档后，Cmd+Shift+P后选择Preview Swagger，可以预览效果

## 创建微服务实战

### protoc的使用

如果使用any类型和value类型，需要类似如下操作，待补充：

参考了https://www.liwenzhou.com/posts/Go/protobuf/

```shell
goctl rpc protoc xx.proto --go_out=./types --go-grpc_out=./types --zrpc_out=.   --proto_path=/Users/rhettnina/go/src/google/protobuf/src/  --proto_path=.
```

value类型的使用处转换方法参考如下： `count = int(value.GetNumberValue())`

如果想把int类型unmarshal到proto定义生成的字段，需要在proto里把字段定义为value类型而不是any类型，使用any类型会报错，value类型对于number string bool list这些单值都可以成功unmarshal。

proto定义参考如下：

```protobuf
syntax = "proto3";

import "google/protobuf/any.proto";
import "google/protobuf/struct.proto";

package p;

option go_package = "./p";

message Edge {
  map<string, google.protobuf.Any> properties = 1;
  google.protobuf.Value group_value = 2;
}
```

我发现修改proto或者模板之后重新用goctl生成的话，如果新生成的文件和原来生成的文件不同，也不会覆盖掉原来生成的文件，除非把原来生成的文件删掉。

## rpc服务启动太慢，调用rpc经常超时的解决

发现每次启动rpc服务的时候，都要好几秒才能启动成功。api服务调用rpc服务的接口，还经常超时，即使设置了5秒甚至10秒超时时间的context。

api服务启动也慢，不知道是不是因为api服务在启动中需要连接rpc服务，因为rpc太慢导致的。

参考了 [issue](https://github.com/zeromicro/go-zero/issues/1326)，发现调用rpc的默认超时时间是2秒，因为我的 rpc服务有时候正常处理的情况下也得花费超过2秒以上的时间，所以需要修改超时时间，需要在调用方修改超时时间也需要在rpc方修改超时时间。

#### 调用方修改

```diff
Name: api
Host: 0.0.0.0
Port: 8888
XXRpc:
  Etcd:
    Hosts:
    - 127.0.0.1:2379
    Key: xx.rpc
 + Timeout: 10000 # 新增的配置，指定调用XXRpc超时时间为10秒
```

调用方的代码：

internal/config/config.go里的

```
type Config struct {
	rest.RestConf
	XXRpc zrpc.RpcClientConf
	Log       logx.LogConf
}
```

main.go里的

```go
	ctx := svc.NewServiceContext(c)
	server := rest.MustNewServer(c.RestConf)
	defer server.Stop()
```

关于MustNewServer的源码：

```go
func MustNewServer(c RestConf, opts ...RunOption) *Server {
	server, err := NewServer(c, opts...)
	if err != nil {
		log.Fatal(err)
	}

	return server
}
```

```go
	RestConf struct {
		service.ServiceConf
		Host     string `json:",default=0.0.0.0"`
		Port     int
		CertFile string `json:",optional"`
		KeyFile  string `json:",optional"`
		Verbose  bool   `json:",optional"`
		MaxConns int    `json:",default=10000"`
		MaxBytes int64  `json:",default=1048576"`
		// milliseconds
		Timeout      int64         `json:",default=3000"`
		CpuThreshold int64         `json:",default=900,range=[0:1000]"`
		Signature    SignatureConf `json:",optional"`
	}
```

修改完毕后，调用rpc的部分正常写即可，不用再指定context.WithTimeout了，参考下面：

```go
func (l *HostStatusLogic) HostStatus() (resp []types.HostStatus, err error) {
	instanceGroupResp, err := l.svcCtx.XXRpc.InstanceGroup(l.ctx, &xx.InstanceGroupRequest{
	})
```

#### rpc方修改

配置文件修改：

```diff
Name: xx.rpc
ListenOn: 127.0.0.1:8080
Etcd:
  Hosts:
  - 127.0.0.1:2379
  Key: xx.rpc
+Timeout: 10000
```

代码部分不需要调整。

补充和配置文件相关的代码调用：

internal/config.config.go里的

```go
type Config struct {
	zrpc.RpcServerConf
	Log logx.LogConf
}
```

main.go里的

```
	s := zrpc.MustNewServer(c.RpcServerConf, func(grpcServer *grpc.Server) {
		youwei.RegisterXXServer(grpcServer, svr)

		if c.Mode == service.DevMode || c.Mode == service.TestMode {
			reflection.Register(grpcServer)
		}
	})
	defer s.Stop()

```

关于RpcServerConf的源码定义：

```go
	// A RpcServerConf is a rpc server config.
	RpcServerConf struct {
		service.ServiceConf
		ListenOn      string
		Etcd          discov.EtcdConf    `json:",optional"`
		Auth          bool               `json:",optional"`
		Redis         redis.RedisKeyConf `json:",optional"`
		StrictControl bool               `json:",optional"`
		// setting 0 means no timeout
		Timeout      int64 `json:",default=2000"`
		CpuThreshold int64 `json:",default=900,range=[0:1000]"`
		// grpc health check switch
		Health bool `json:",default=true"`
	}
```
