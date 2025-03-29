---
title: Go语言高级编程-第5章go和web
date: 2021-12-4 00:00:00
tags:
- 读书
categories:
- golang
---

> Go语言高级编程系列是我读《Go语言高级编程》时的一些要点总结。

1. 因为Go的 net/http 包提供了基础的路由函数组合与丰富的功 能函数。所以在社区里流行一种用Go编写api不需要框架的观点;在我们看来，如果你的项目的路由在个位数、URI 固定且 不通过 URI 来传递参数，那么确实使用官方库也就足够。

2. 只要你的路由带有参数，并且 这个项目的 api 数目超过了 10，就尽量不要使用 net/http 中默 认的路由。

3. 在常见的 web 框架中，router 是必备的组件。golang 圈子里 router 也时常被称为 http 的 multiplexer。在上一节中我们通过 对 Burrow 代码的简单学习，已经知道如何用 http 标准库中内 置的 mux 来完成简单的路由功能了。

4. restful 中常见的请求路径:

   ````
   GET /repos/:owner/:repo/comments/:id/reactions
   POST /projects/:project_id/columns
   PUT /user/starred/:owner/:repo
   DELETE /user/starred/:owner/:repo
   ````

   restful 风格的 API 重度依赖请求路径。会将很 多参数放在请求 URI 中。除此之外还会使用很多并不那么常 见的 HTTP 状态码，

5. 较流行的开源 golang web 框架大多使用 httprouter，或是基于 httprouter 的变种对路由进行支持。前面提到的 github 的参数 式路由在 httprouter 中都是可以支持的。

   因为 httprouter 中使用的是显式匹配，所以在设计路由的时候 需要规避一些会导致路由冲突的情况，例如:

   ```go
   conflict:
     GET /user/info/:name
     GET /user/:id
     no conflict:
     GET /user/info/:name
     POST /user/:id
   ```

   简单来讲的话，如果两个路由拥有一致的 http method (指 GET/POST/PUT/DELETE) 和请求路径前缀，且在某个位置出 现了 A 路由是 wildcard (指 :id 这种形式) 参数，B 路由则是普 通字符串，那么就会发生路由冲突。

   因为 httprouter 考虑到字典树的深度，在 初始化时会对参数的数量进行限制，所以在路由中的参数数目 不能超过 255，否则会导致 httprouter 无法识别后续的参数。

   除支持路径中的wildcard参数之外，httprouter还可以支持 *号来进行通配，不过* 号开头的参数只能放在路由的结尾， 例如下面这样:

   ```
   Pattern: /src/*filepath
   ```

6. 除了正常情况下的路由支持，httprouter 也支持对一些特殊情 况下的回调函数进行定制，例如 404 的时候

   目前开源界最为流行(star 数最多)的 web 框架 gin 使用的就是 httprouter 的变种。

7. httprouter 和众多衍生 router 使用的数据结构被称为 radix tree， 压缩字典树。读者可能没有接触过压缩字典树，但对字典树 trie tree 应该有所耳闻。

8. 我们犯的最大的错误是把业务代码和非业务代码揉在 了一起。对于大多数的场景来讲，非业务的需求都是在 http 请 求处理前做一些事情，或者/并且在响应完成之后做一些事 情。一个middleware可以类似这样写，注意`next.ServeHTTP(wr, r)`

   ```go
   func  hello(wr http.ResponseWriter, r *http.Request) {
         wr.Write([] byte ( "hello" ))
   }
   func  timeMiddleware(next http.Handler) http.Handler {
          return                    func
             timeStart := time.Now()
              // next handler
             next.ServeHTTP(wr, r)
             timeElapsed := time.Since(timeStart)
             logger.Println(timeElapsed)
         })
   }
   func main() {
    http.Handle(
   "/"
    , timeMiddleware(http.HandlerFunc(hello)))
   err := http.ListenAndServe( ":8080" ,  nil )
   ...
   }
   ```

9. 一般为了验证请求是否正确，我们得为每一个 http 请求都去写这么一套差不多的 validate 函数，有没有更好的办法来帮助我们解除这项体力劳 动?答案就是 validator。

   这里我们引入一个新的 validator 库:

   ```go
   import   "gopkg.in/go-playground/validator.v9"
   type RegisterReq struct {
   // 字符串的 gt=0 表示长度必须 > 0，gt = greater than
   Username string `validate:"gt=0"` // 同上
   PasswordNew string `validate:"gt=0"` // eqfield 跨字段相等校验
         PasswordRepeat   string
     `validate:"eqfield=PasswordNew"`
   // 合法 email 格式校验
   Email   string `validate:"email"`
   }
   
   
     func  validate(req RegisterReq) error {
         err := validate.Struct(mystruct)
          if  err !=  nil  {
             doSomething()
   }
   ...
   }
   ```

10. 注意下面函数写法，对于reflect

    ```go
    func validate(v interface {})(bool, string) {
        validateResult: = true
        errmsg: = "success"
        vt: = reflect.TypeOf(v)
        vv: = reflect.ValueOf(v)
        for i: = 0;i < vv.NumField();i++{
            fieldVal: = vv.Field(i)
            tagContent: = vt.Field(i).Tag.Get("validate")
            k: = fieldVal.Kind()
            switch k {
                case reflect.String:
                    val: = fieldVal.String() tagValStr: = tagContent
                    switch tagValStr {
    
                        case "email":
                            nestedResult: = validateEmail(val)
                            if nestedResult == false {
                                errmsg =
                                    "validate mail failed, field val is: " + val
                                validateResult = false
                            }
                    }
                case reflect.Struct:
                    // 如果有内嵌的 struct，那么深度优先遍历 // 就是一个递归过程
                    valInter: = fieldVal.Interface() nestedResult, msg: = validate(valInter) if nestedResult == false {
                        validateResult = false
                        errmsg = msg
                    }
            }
        }
        return validateResult,
        errmsg
    }
    ```

11. 我们对 struct 进行 validate 时大量使用了 reflect，而 go 的 reflect 在性 能上不太出众，有时甚至会影响到我们程序的性能。这样的考 虑确实有一些道理，但需要对 struct 进行大量校验的场景往往 出现在 web 服务，这里并不一定是程序的性能瓶颈所在，实 际的效果还是要从 pprof 中做更精确的判断。

12. Go官方提供了 database/sql 包来给用户进行和数据库打交道 的工作，实际上 database/sql 库就只是提供了一套操作数据 库的接口和规范，例如抽象好的 sql 预处理(prepare)，连接池 管理，数据绑定，事务，错误处理等等。官方并没有提供具体 某种数据库实现的协议支持。

    **和具体的数据库，例如 MySQL 打交道，还需要再引入 MySQL 的驱动，像下面这样**

    ```go
     import   "database/sql"
     import  _  "github.com/go-sql-driver/mysql"
    ```

    这一句 import，实际上是调用了 mysql 包的 init 函数，做的事 情也很简单:

    ```go
      func  init() {
          sql.Register( "mysql" , &MySQLDriver{})
      }
    ```

13. 注意读取出来内容后，必须要把 rows 里的内容读完，否则**连接**永远不会释放

    ```go
     rows, err := db.Query(
    "select id, name from users where id = ?" ,  1 )
         if  err !=  nil  {
            log.Fatal(err)
        }
         defer  rows.Close()
    // 必须要把 rows 里的内容读完，否则连接永远不会释放 
    for rows.Next() {
    ```

14. 在 web 开发领域常常提到的 ORM 是什么?我们先看看万能的 维基百科:

    ```
    对象关系映射(英语:Object Relational Mapping，简称ORM，或O/RM，或 O/R mapping)，
    是一种程序设计技术，用于实现面向对象编程语言里不同类型系统的数据之间的转换。
    从效果上说，它其实是创建了一个可在编程语言里使用的“虚拟对象数据库”。
    ```

    最为常见的 ORM 实际上做的是从 db -> 程序的 class / struct 这 样的映射。所以你手边的程序可能是从 mysql 的表 -> 你的程 序内 class。

     ORM 的目的就是屏蔽掉DB 层，实际上很多语言的 ORM 只要把你的 class/struct 定义好， 再用特定的语法将结构体之间的一对一或者一对多关系表达出 来。那么任务就完成了。然后你就可以对这些映射好了数据库 表的对象进行各种操作，例如 save，create，retrieve，delete。 至于 orm在背地里做了什么阴险的勾当，你是不一定清楚的。

    ORM 一类的工具在出发点上就是屏蔽 sql，让我们对数据 库的操作更接近于人类的思维方式。

15. 有些 orm 背后隐藏了非常难以察觉的细节， 那就是生成的 sql 语句会自动 limit 1000。喜欢强类型语言的人一般都不喜欢语 言隐式地去做什么事情，例如各种语言在赋值操作时进行的隐 式类型转换然后又在转换中丢失了精度的勾当，一定让你非常 的头疼。所以一个程序库背地里做的事情还是越少越好，如果 一定要做，那也一定要在显眼的地方做。比如上面的例子，去 掉这种默认的自作聪明的行为，或者要求用户强制传入 limit 参数都是更好的选择。orm 想从设计上隐去太多的细 节。而方便的代价是其背后的运行完全失控。

16. 相比 ORM 来说，sql builder 在 sql 和项目可维护性之间取得了 比较好的平衡。首先 sql builer 不像 ORM 那样屏蔽了过多的细 节，其次从开发的角度来讲，sql builder 简单进行封装后也可 以非常高效地完成开发，举个例子:

    ```go
     where :=  map [ string ] interface {} {
           "order_id > ?"  :  0 ,
           "customer_id != ?"  :  0 ,
      }
      limit := [] int { 0 , 100 }
      orderBy := [] string { "id asc" ,  "create_time desc" }
      orders := orderModel.GetList(where, limit, orderBy)
    ```

    说白了 sql builder 是 sql 在代码里的一种特殊方言，如果你们 没有DBA但研发有自己分析和优化 sql 的能力，或者你们公司 的 dba 对于学习这样一些 sql 的方言没有异议。那么使用 sql builder 是一个比较好的选择，不会导致什么问题。

17. 所以现如今，大型的互联网公司核心线上业务都会在代码中把 sql 放在显眼的位置提供给 DBA review，以此来控制系统在数 据层的风险。

18. web 系统打交道最多的是网络，无论是接收，解析用户请求， 访问存储，还是把响应数据返回给用户，都是要走网络的。在 没有 epoll/kqueue 之类的系统提供的 IO 多路复用接口之前， 多个核心的现代计算机最头痛的是 C10k 问题，C10k 问题会导 致计算机没有办法充分利用 CPU 来处理更多的用户连接，进 而没有办法通过优化程序提升 CPU 利用率来处理更多的请 求。

    自从 linux 实现了 epoll，freebsd 实现了 kqueue，这个问题基 本解决了，我们可以借助内核提供的 API 轻松解决当年的 C10k 问题，**也就是说如今如果你的程序主要是和网络打交 道，那么瓶颈一定在用户程序而不在操作系统内核。**

19. Go 的 net 库针对不同平台封装了不同的 syscall API，http 库又是构建 在 net 库之上，所以在 Go 我们可以借助标准库，很轻松地写 出高性能的 http 服务，

20. 如果 碰到业务逻辑复杂代码量巨大的模块，其瓶颈并不是三下五除 二可以推测出来的，还是需要从压力测试中得到更为精确的结 论。

21. 对于 IO/Network bound 类的程序，其表现是网卡/磁盘 IO 会先 于 CPU 打满，这种情况即使优化 CPU 的使用也不能提高整个 系统的吞吐量，**只能提高磁盘的读写速度，增加内存大小**，提 升网卡的带宽来提升整体性能。而 CPU bound 类的程序，则 是在存储和网卡未打满之前 CPU 占用率提前到达 100%，CPU 忙于各种计算任务，IO 设备相对则较闲。

    无论哪种类型的服务，在资源使用到极限的时候都会导致请求 堆积，超时，系统 hang 死，最终伤害到终端用户。对于分布 式的 web 服务来说，瓶颈还不一定总在系统内部，也有可能 在外部。非计算密集型的系统往往会在关系型数据库环节失 守，而这时候 web 模块本身还远远未达到瓶颈。

    不管我们的服务瓶颈在哪里，最终要做的事情都是一样的，那就是流量限制。

22. 流量限制的手段有很多，最常见的:漏桶、令牌桶两种

23. QoS 全称是 Quality of Service，顾名思义是服务质 量。QoS 包含有可用性、吞吐量、时延、时延变化和丢失等指 标。一般来讲我们可以通过优化系统，来提高 web 服务的 CPU 利用率，从而提高整个系统的吞吐量。

24. 流行的 web 框架大多数是 MVC 框架，MVC 这个概念最早由 Trygve Reenskaug 在 1978 年提出，为了能够对 GUI 类型的应 用进行方便扩展，将程序划分为:

    1. 控制器(Controller)- 负责转发请求，对请求进行处理。
    2. 视图(View)-界面设计人员进行图形界面设计。
    3. 模型(Model)-程序员编写程序应有的功能(实现算法等等)、数据库专家进行数据管理和数据库设计(可以实现 具体的功能)。

    随着时代的发展，前端也变成了越来越复杂的工程，为了更好 地工程化，现在更为流行的一般是前后分离的架构。前后端之间通过 ajax 来交 互，有时候要解决跨域的问题

25. 业务流程也算是一种“模型”，是对 真实世界用户行为或者既有流程的一种建模，并非只有按格式 组织的数据才能叫模型

26. 现在比较流行的 纯后端 api 模块一般采用下述划分方法:

    1. Controller，与上述类似，服务入口，负责处理路由，参数 校验，请求转发。

    2. Logic/Service，逻辑(服务)层，一般是业务逻辑的入口，可 以认为从这里开始，所有的请求参数一定是合法的。业务 逻辑和业务流程也都在这一层中。常见的设计中会将该层 称为 Business Rules。

    3. DAO/Repository，这一层主要负责和数据、存储打交道。 将下层存储以更简单的函数、接口形式暴露给 Logic 层来 使用。负责数据的持久化工作。

    可以认为，我 们的代码运行到 controller 层之后，就没有任何与“协议”相关 的代码了。在这里你找不到 http.Request，也找不到 http.ResponseWriter，也找不到任何与 thrift 或者 gRPC 相关的 字眼。

27. 我们的外部依赖总是为了自己爽而不断地做升级，且不想做向前兼容

28. 非实时的统计类系统，那么就没有必 要在主流程里为每一套系统做一套 RPC 流程。我们只要将下 游需要的数据打包成一条消息，传入消息队列，之后的事情与 主流程一概无关

29. 一件事情本身变的复杂的话，这时候拆解和异步化就不灵了。我们还是要对事情本身进行一定程度的封装抽象。

30. 引入 interface 对 我们的系统本身是否有意义，这是要按照场景去进行分析的。 假如我们的系统只服务一条产品线，并且内部的代码只是针对 很具体的场景进行定制化开发，那么实际上引入 interface 是不 会带来任何收益的。

31. 当我们接手了一个 几十万行的系统时，如果看到定义了很多 interface，例如订单 流程的 interface，**我们希望能直接找到这些 interface 都被哪些 对象实现了。但直到现在，这个简单的需求也就只有 goland 实现了，**并且体验尚可。

32. 熟悉开源 lint 工具的同学应该见到过圈复杂度的说法，**在函数 中如果有 if 和 switch 的话，会使函数的圈复杂度上升。**所以 有强迫症的同学即使在入口一个函数中有 switch，还是想要干 掉这个 switch，有没有什么办法呢?当然有，用表驱动的方式 来存储我们需要实例:

    ```go
     func entry() {
         var bi BusinessInstance
         switch businessType {
             case TravelBusiness:
                 bi = travelorder.New()
             case MarketBusiness:
                 bi = marketorder.New()
             default:
                 return errors.New("not supported business")
         }
     }
    ```

    可以修改为:

    ```go
     var businessInstanceMap = map[int] BusinessInstance {
         TravelBusiness: travelorder.New(),
         MarketBusiness: marketorder.New(),
     }
     func entry() {
         bi: = businessInstanceMap[businessType]
     }
    ```

    table driven 的设计方式，很多设计模式相关的书籍并没有把它 作为一种设计模式来讲，但我认为这依然是一种非常重要的帮 助我们来简化代码的手段。在日常的开发工作中可以多多思 考，**不必要的 switch case 可以用一个字典和一行代码就可 以轻松搞定。**

33. 灰度非常重要，灰度发布也称为金丝雀 发布。互联网系统的灰度发布一般通过两种方式实现:

    1. 通过分批次部署实现灰度发布 2. 通过业务规则进行灰度发布

    在对系统的旧功能进行升级迭代时，第一种方式用的比较多。新功能上线时，第二种方式用的比较多。当然，对比较重要的老功能进行较大幅度的修改时，一般也会选择按业务规则来进行发布，因为直接全量开放给所有用户风险实在太大。

34. 假如服务部署在 15 个实例(可能是物理机，也可能是容器) 上，我们把这 7 个实例分为三组，按照先后顺序，分别有 1-2- 4-8 台机器，保证每次扩展时大概都是二倍的关系。为什么要用 2 倍?这样能够保证我们不管有多少台机器，都不 会把组划分得太多。例如 1024 台机器，实际上也就只需要 1- 2-4-8-16-32-64-128-256-512 部署十次就可以全部部署完毕。

35. 在上线时，最有效的观察手法是查看程序的错误日志，如果较 明显的逻辑错误，一般错误日志的滚动速度都会有肉眼可见的 增加。

36. **map 的查询比数组稍慢，但扩展会灵活一 些**

37. 求哈希可用的算法非常多，比如 md5，crc32，sha1 等等，但 我们这里的目的只是为了给这些数据做个映射，并不想要因为 计算哈希消耗过多的 cpu，所以现在业界使用较多的算法是 murmurhash，下面是我们对这些常见的 hash 算法的简单benchmark

    hash.go:

    ```go
    package main
    import
    import
    import "crypto/md5"
    "crypto/sha1"
    "github.com/spaolacci/murmur3"
    var str = "hello world"
    func md5Hash()[16] byte {
        return md5.Sum([] byte(str))
    }
    func sha1Hash()[20] byte {
        return sha1.Sum([] byte(str))
    }
    func murmur32() uint32 {
        return murmur3.Sum32([] byte(str))
    }
    func murmur64() uint64 {
        return murmur3.Sum64([] byte(str))
    }
    ```

    hash_test.go

    ```go
    package main
    import "testing"
    func BenchmarkMD5(b * testing.B) {
        for i: = 0;
        i < b.N;
        i++{
            md5Hash()
        }
    }
    func BenchmarkSHA1(b * testing.B) {
        for i: = 0;
        i < b.N;
        i++{
            sha1Hash()
        }
    }
    func BenchmarkMurmurHash32(b * testing.B) {
        for i: = 0;
        i < b.N;
        i++{
            murmur32()
        }
    }
    func BenchmarkMurmurHash64(b * testing.B) {
        for i: = 0;
        i < b.N;
        i++{
            murmur64()
        }
    }
    ```

    > ~/t/g / hash_bench git: master❯❯❯ go test - bench = .goos: darwin
    > goarch: amd64
    > BenchmarkMD5 - 4
    > BenchmarkSHA1 - 4
    > BenchmarkMurmurHash32 - 4
    > BenchmarkMurmurHash64 - 4
    > PASS
    >
    > 10000000
    > 10000000
    > 50000000
    > 20000000
    > 180 ns / op
    > 211 ns / op
    > 25.7 ns / op
    > 66.2 ns / op
    > 7.050 s
    > ok _ / Users / caochunhui / test / go / hash_bench

​  可见 murmurhash 相比其它的算法有三倍以上的性能提升。

38. 对于哈希算法来说，性能是一方面的问题，另一方面还要考虑哈希后的值是否分布均匀。
39. web 甚至可以 不用非得基于 http 协议。**只要是 CS 或者 BS 架构，都可以认 为是 web 系统。**
