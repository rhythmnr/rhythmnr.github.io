---
title: Go语言高级编程-第4章 RPC和Protobuf
date: 2021-12-4 00:00:00
tags:
- 读书
categories:
- golang
---

> Go语言高级编程系列是我读《Go语言高级编程》时的一些要点总结。

1. RPC是远程过程调用的缩写(Remote Procedure Call)，通俗 地说就是调用远处的一个函数

2. 因为 RPC涉及的函数可能非常之远，远到它们之间说着完全不同的 语言，语言就成了两边的沟通障碍。**而Protobuf因为支持多种 不同的语言(甚至不支持的语言也可以扩展支持)，其本身特性也非常方便描述服务的接口(也就是方法列表)，因此非常 适合作为RPC世界的接口交流语言**

3. RPC是远程过程调用的简称，是分布式系统中不同节点间流行 的通信方式。

4. Go语言的RPC包的路径为net/rpc，也就是放在了net包目录下 面。因此我们可以猜测该RPC包是建立在net包基础之上的

5. **我们尝试基于rpc实现一个类似的例子。**

   我们先构造一个HelloService类型，其中的Hello方法用于实现 打印功能:

```go
type HelloService struct {}
func(p * HelloService) Hello(request string, reply * string) error { 
  *reply = "hello" + request
    return nil
}
```

其中Hello方法必须满足Go语言的RPC规则:方法只能有两个 可序列化的参数，其中第二个参数是指针类型，并且返回一个 error类型，同时必须是公开的方法。

然后就可以将HelloService类型的对象注册为一个RPC服务:

```go
func main() {
    rpc.RegisterName("HelloService", new(HelloService))
    listener, err: = net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }
    conn, err: = listener.Accept()
    if err != nil {
        log.Fatal("Accept error:", err)
    }
    rpc.ServeConn(conn)
}
```

其中rpc.Register函数调用会将对象类型中所有满足RPC规则的 对象方法注册为RPC函数，所有注册的方法会放 在“HelloService”服务空间之下。然后我们建立一个唯一的TCP 链接，并且通过rpc.ServeConn函数在**该TCP链接**上为对方提供 RPC服务。

下面是客户端请求HelloService服务的代码:

```go
 func main() {
     client, err: = rpc.Dial("tcp", "localhost:1234")
     if err != nil {
         log.Fatal("dialing:", err)

     }
     var reply string
     err = client.Call("HelloService.Hello", "hello", & reply)
     if err != nil {
         log.Fatal(err)
     }
     fmt.Println(reply)
 }
```

首选是通过rpc.Dial拨号RPC服务，然后通过client.Call调用具 体的RPC方法。

6. 标准库的RPC默认采用Go语言特有的gob编码，因此从其它语 言调用Go语言实现的RPC服务将比较困难。

7. Go语言的RPC框架有两个比较有特色的设计:一个是RPC数据 打包时可以通过插件实现自定义的编码和解码;另一个是RPC 建立在抽象的io.ReadWriteCloser接口之上的，我们可以将RPC 架设在不同的通讯协议之上。这里我们将尝试通过官方自带的 net/rpc/jsonrpc扩展实现一个跨语言的PPC。

   首先是基于json编码重新实现RPC服务:

   ```go
   func main() {
       rpc.RegisterName("HelloService", new(HelloService))
       listener, err: = net.Listen("tcp", ":1234")
       if err != nil {
           log.Fatal("ListenTCP error:", err)
       }
       for {
           conn, err: = listener.Accept()
           if err != nil {
               log.Fatal("Accept error:", err)
           }
           go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
       }
   }
   ```

   代码中最大的变化是用rpc.ServeCodec函数替代了 rpc.ServeConn函数，传入的参数是针对服务端的json编解码 器。

   然后是实现json版本的客户端:

   ```go
   func main() {
       conn, err: = net.Dial("tcp", "localhost:1234")
       if err != nil {
           log.Fatal("net.Dial:", err)
       }
       client: = rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
       var reply string
       err = client.Call("HelloService.Hello", "hello", &reply)
       if err != nil {
           log.Fatal(err)
       }
       fmt.Println(reply)
   }
   ```

8. 在确保客户端可以正常调用RPC服务的方法之后，我们用一个 普通的TCP服务代替Go语言版本的RPC服务，这样可以查看客 户端调用时发送的数据格式。**比如通过nc命令 nc -l 1234 在 同样的端口启动一个TCP服务。**

9. 在获取到RPC调用对应的json数据后，我们可以通过直接向架 设了RPC服务的TCP服务器发送json数据模拟RPC方法调用:

   ```
    $ echo -e '{"method":"HelloService.Hello","params":
    ["hello"],"id":1}' | nc localhost 1234
   ```

​   返回的结果也是一个json格式的数据:

​   { "id" : 1 , "result" : "hello:hello" , "error" : null }

因此无论采用何种语言，只要遵循同样的json结构，以同样的 流程就可以和Go语言编写的RPC服务进行通信。这样我们就实 现了跨语言的RPC。

10. RPC的服务架设在“/jsonrpc”路径，在处理函数中基于 http.ResponseWriter和http.Request类型的参数构造一个 io.ReadWriteCloser类型的conn通道。然后基于conn构建针对服 务端的json编码解码器。最后通过rpc.ServeRequest函数为每次 请求处理一次RPC方法调用。

11. Protobuf是Protocol Buffers的简称，它是Google公司开发的一种 数据描述语言，并于2008年对外开源。Protobuf刚开源时的定 位类似于XML、JSON等数据描述语言，通过附带工具生成代 码并实现将结构化数据序列化的功能。

12. Protobuf中最基本 的数据单元是message，是类似Go语言中结构体的存在。在 message中可以嵌套message或其它的基础数据类型的成员。

13. 在XML或JSON等数据描述语言中，一般通过成员的名字来绑 定对应的数据。但是Protobuf编码却是通过成员的唯一编号来 绑定对应的数据，因此Protobuf编码后数据的体积会比较小， 但是也非常不便于人类查阅。我们目前并不关注Protobuf的编 码技术，最终生成的Go结构体可以自由采用JSON或gob等编码 格式，因此大家可以暂时忽略Protobuf的成员编码部分。

    Protobuf核心的工具集是C++语言开发的，在官方的protoc编译 器中并不支持Go语言。要想基于上面的hello.proto文件生成相 应的Go代码，需要安装相应的插件。首先是安装官方的protoc 工具，可以从 <https://github.com/google/protobuf/releases> 下载。 然后是安装针对Go语言的代码生成插件，可以通过 go get github.com/golang/protobuf/protoc-gen-go 命令安装。

    然后通过以下命令生成相应的Go代码:

    ```go
    protoc --go_out=. hello.proto
    ```

    其中 go_out 参数告知protoc编译器去加载对应的protoc-gen- go工具，然后通过该工具生成代码，生成代码放到当前目录。 最后是一系列要处理的protobuf文件的列表。

    这里只生成了一个hello.pb.go文件

14. 其实用Protobuf定义语言无关的RPC服务接口 才是它真正的价值所在!

15. 在protoc-gen-go内部已经集成了一个叫grpc的插件，可以 针对grpc生成代码:

    ```go
    protoc --go_out=plugins=grpc:. hello.proto
    ```

    在生成的代码中多了一些类似HelloServiceServer、 HelloServiceClient的新类型。这些类型是为grpc服务的，并不 符合我们的RPC要求。

    go_out=plugins=grpc 参数来生成grpc相关代码，否则只会针对 message生成相关代码。

16. 因为Go语言的包只能静态导入，我们无法向已经安装的protoc- gen-go添加我们新编写的插件。我们将重新克隆protoc-gen-go 对应的main函数

    我们将我们的可执行 程序命名为protoc-gen-go-netrpc，表示包含了nerpc插件

    插件中的 plugins=netrpc 指示启用内部 唯一的名为netrpc的netrpcPlugin插件。

17. rpc调用中的client.send方法调用是线程安全的，因此可以从多个Goroutine同时向同一 个RPC链接发送调用指令。

18. **反向RPC:**通常的RPC是基于C/S结构，RPC的服务端对应网络的服务 器，RPC的客户端也对应网络客户端。但是对于一些特殊场 景，比如在公司内网提供一个RPC服务，但是在外网无法链接 到内网的服务器。这种时候我们可以参考类似反向代理的技 术，**首先从内网主动链接到外网的TCP服务器，然后基于TCP 链接向外网提供RPC服务**

    反向RPC的内网服务将不再主动提供TCP监听服务，而是首先 主动链接到对方的TCP服务器。然后基于每个建立的TCP链接 向对方提供RPC服务。

    ```go
    func main() {
        rpc.Register(new(HelloService))
    
        for {
            conn, _: = net.Dial("tcp", "localhost:1234")
            if conn == nil {
                time.Sleep(time.Second)
                continue
            }
            rpc.ServeConn(conn)
            conn.Close()
        }
    }
    ```

    而RPC客户端则需要在一个公共的地址提供一个TCP服务，用 于接受RPC服务器的链接请求:

    ```go
     func main() {
         listener, err: = net.Listen("tcp", ":1234")
         if err != nil {
             log.Fatal("ListenTCP error:", err)
         }
         clientChan: = make(chan * rpc.Client)
         go func() {
             for {
                 conn, err: = listener.Accept()
                 if err != nil {
                     log.Fatal("Accept error:", err)
                 }
                 clientChan < -rpc.NewClient(conn)
             }
         }()
         doClientWork(clientChan)
     }
    ```

19. **RPC是远程函数调用，因此每次调用的函数参数和返回值不能 太大**，否则将严重影响每次调用的响应时间。因此传统的RPC 方法调用对于上传和下载较大数据量场景并不适合。

    **RPC模式也不适用于对时间不确定的订阅和发布模式。**为此， GRPC框架针对服务器端和客户端分别提供了流特性。

    **服务端或客户端的单向流是双向流的特例**。我们在 HelloService增加一个支持双向流的Channel方法:

    ```go
    rpc Channel (stream String) returns (stream String);
    ```

    关键字stream指定启用流特性，参数部分是接收客户端参数的 流，返回值是返回给客户端的流。

20. **GRPC建立在HTTP/2协议之上，对TLS提供了很好的支持。我 们前面章节中GRPC的服务都没有提供证书支持，因此客户端 在链接服务器中通过 grpc.WithInsecure() 选项跳过了对服务 器证书的验证。**没有启用证书的GRPC服务在和客户端进行的 是明文通讯，信息面临被任何第三方监听的风险。为了保障 GRPC通信不被第三方监听篡改或伪造，我们可以对服务器启 动TLS加密特性。

    可以将生成server.key、server.crt、client.key和client.crt四 个文件。其中以.key为后缀名的是私钥文件，需要妥善保管。 以.crt为后缀名是证书文件，也可以简单理解为公钥文件，并 不需要秘密保存。在subj参数中的 /CN=server.grpc.io 表示服 务器的名字为 server.grpc.io ，在验证服务器的证书时需要 用到该信息。

    服务端代码就写成

    ```go
    creds, err := credentials.NewServerTLSFromFile(
      "server.crt" ,  "server.key" )
    ```

​  客户端代码写成

```go
 creds, err := credentials.NewClientTLSFromFile(
           "server.crt" ,  "server.grpc.io" ,
      )
```

其中redentials.NewClientTLSFromFile是构造客户端用的证书对 象，第一个参数是服务器的证书文件，第二个参数是签发证书 的服务器的名字。

21. 如果截取器函数返回了错误，那么该次GRPC方法调用将被视 作失败处理。因此，我们可以在截取器中对输入的参数做一些 简单的验证工作。同样，也可以对handler返回的结果做一些验 证工作。截取器也非常适合前面对Token认证工作。

22. GRPC构建在HTTP/2协议之上，因此我们可以将GRPC服务和 普通的Web服务架设在同一个端口之上。因为目前Go语言版本 的GRPC实现还不够完善，只有启用了TLS协议之后才能将GRPC和Web服务运行在同一个端口。

    因为GRPC服务已经实现了ServeHTTP方法，可以直接作为 Web路由处理对象。如果将GRPC和Web服务放在一起，会导 致GRPC和Web路径的冲突，在处理时我们需要区分两类服 务。

    我们可以通过以下方式生成同时支持Web和GRPC协议的路由 处理函数:

    ```go
      grpcServer.ServeHTTP(w, r)  // GRPC Server
    ```

23. GRPC服务一般用于集群内部通信，如果需要对外暴露服务一 般会提供等价的REST接口。通过REST接口比较方便前端 JavaScript和后端交互.通 过RegisterRestServiceHandlerFromEndpoint函数将RestService服 务相关的REST接口中转到后面的GRPC服务。

24. Protobuf本身具有反射功能，可以在运行时获取对象的Proto文 件。grpc同样也提供了一个名为reflection的反射包，用于为 grpc服务提供查询。GRPC官方提供了一个C++实现的grpc_cli 工具，可以用于查询GRPC列表或调用GRPC方法。
