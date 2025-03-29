---
title: GO专家编程阅读记录
date: 2023-12-12 11:00:00
categories:
- golang
---

> 下面的记录是关于我在阅读《GO专家编程》这本书的笔记，笔记的内容主要是关于一些我不太熟悉的需要记忆的知识点。
>
> 因为这本书比较短，而且内容也不是特别深，所以记录的内容也没有特别多。记录的内容关注点在自己不熟悉的知识点。也没有完全按照每章的目录结构记录。

## 第一章 常见数据结构实现原理

channel是Golang在语言级别提供的goroutine之间的通信方式，比Unix管道更易用也更轻便。

### channel的数据结构

```go
type hchan struct{
	qcount uint // 当前队列中剩余元素个数
	dataqsiz uint // 环形队列长度，即可以存放的元素个数
	buf unsafe.Pointer // 环形队列指针
	elemsize uint16 // 每个元素的大小
	closed uint32 // 标识关闭状态
	elemtype *_type // 元素类型
	sendx uint // 队列下标，指示元素写入时存放到队列中的位置
	recvx uint // 队列下标，指示元素从队列的该位置读出
	recvq waitq // 等待读消息的goroutine队列
	sendq waitq // 等待写消息的goroutine队列
	lock mutex // 互斥锁，chan不允许并发读写
}
```

从数据结构的定义就可以看出来，chan内部实现了一个环形队列作为其缓冲区，队列的长度是在创建chan时指定的。

> **读/写数据的goroutine阻塞的场景：**
>
> 从channel中读取数据时，如果缓冲区为空或者没有缓冲区，那么读取数据的goroutine会阻塞。
>
> 向channel中写数据时，如果缓冲区满了或者没有缓冲区，会导致向channel写数据的goroutine阻塞。
>
> 所以只要没有缓冲区，以上两种操作都会阻塞。

### 单向channel

单向channel顾名思义，就是只能读或者只能写的channel。单向channel是对使用行为的一种限制，在实际使用过程中可以通过参数的定义来实现单项channel。

```go
func read(chName <- ch int) // 通过参数限制只能从channel里读数据，传入的channel在read函数里只能读
func write(chName  ch <- int) // 通过参数限制只能从channel里写数据，传入的channel在write函数里只能写
func noLimit(chName ch int) // 没有只读或者只写的限制
```

注意这里参数定义的写法，channel的ch关键字的位置和<-的位置决定了是可读的还是可以写的，而channel的类型int总是放在最后。

### slice

示例程序：

```go
func main(){
	var array [10]int
	var slice = array[5:6]
	fmt.Println("length of slice: ", len(slice))
	fmt.Println("capacity of slice: ", cap(slice))
	fmt.Println(&slice[0]==&array[0])
}
```

运行结果为：

```shell
length of slice:  1
capacity of slice:  5
false
```

解释：slice和array共享内存空间，slice在数组的初试位置是array[5]，其容量一直到array的结尾也就是5，slice取array[5:6]是左闭右开的，slice的长度是1。slice[0]和array[5]的地址相同。这里slice不会用到数组头，除非slice指定从array[0]开始切片，**默认slice占用的空间是到数组结束位置的。**

关于slice的长度增长：对slice进行append操作时，可能会使得slice占用的空间增长。append时如果slice检测到空间不足，会申请新空间，新存储空间的大小是原来的2倍或者1.25倍。如果append时空间充足，则slice不会扩容。一般扩容操作是如果原slice容量小于1024，则新slice的容量为原来的2倍。如果原slice的容量大于等于1024，则新slice的容量是原来的1.25倍。

切片时指定容量：比如`order[low:high:max]`表示新切片范围时[low:high)，新切片的容量是max-low。比如

```go
func main(){
	var array [10]int
	var slice = array[5:6:7]
	fmt.Println("length of slice: ", len(slice))
	fmt.Println("capacity of slice: ", cap(slice))
}
```

运行结果是：

```shell
length of slice:  1
capacity of slice:  2
```

slice依据数组实现，底层数组对用户屏蔽，如果底层数组容量不足，则会实现自动重新分配并生成新的slice。所以数组和切片操作时可以作用于同一块内存，这个是需要格外注意的。

### slice copy

使用copy操作两个切片时，会将源切片的数据逐一拷贝到目标切片指向的数组中，**拷贝数量取两个切片长度的最小值。**比如长度为10的切片拷贝到长度为5的切片时，只会拷贝5个元素，因为拷贝数量是两个切片长度的最小值，所以拷贝过程中不会发生扩容。

### 总结

创建slice可以预先分配理想的长度，避免切片扩容，这样可以提升性能。

**通过函数传递切片时，不会拷贝整个切片，因为切片只是一个struct而已。但是函数修改切片会影响到原始切片的值。**比如：

```go
func main(){
	var array [10]int
	var slice = array[5:6:7]
	set(slice)
	fmt.Println(array)
	fmt.Println(slice)
}

func set(a []int){
	a[0]=1
}
```

运行结果是：

```shell
[0 0 0 0 0 1 0 0 0 0]
[1]
```

### map 

map底层是使用哈希表实现的。

哈希表可能会出现一个问题就是哈希冲突，负载因子就是一个衡量哈希冲突情况的指标，公式为：

```text
负载因子 = 键数量/bucket数量
```

> bucket相当于key进行某个操作后得到的哈希值，bucket里存储了key和key对应的value。如果map里的两个key的哈希值相同，那么这个哈希值对应的bucket里就存储了两条数据。查找时先找到key对应的bucket，再到bucket里找到这两对key value，看看哪个key才是要查找的那个key。某个bucket里存储的键值对越多，则存取效率越低。

哈希表需要将负载因子控制在合理范围内，超过阈值需要重新进行rehash，也就是键值对重新组织：

如果哈希因子过小，则表示空间利用率比较小。如果哈希因子过大，则说明冲突严重，存取效率低。

> go的struct的字段经常携带Tag，该Tag主要用于反射场景。在reflect包里使用较多。

## 第二章 常见控制结构实现原理

### iota

iota代表了const声明块的行索引。

### string

Go标准库builtin给出了所有内置类型的定义，源代码位于src/builtin/builtin.go。

字符串拼接可以使用`str="aa"+"bbb"`这样的方式拼接，即使有非常多的字符串需要拼接，性能上也会有比较好的保障，因为新字符串的内存空间是一次性分配完成的，所以性能主要在拷贝数据上。

问题：[]byte转换为string一定会拷贝内存吗？回答：不一定，编译器会识别场景，有时候会拷贝有时候不会，有时候只是需要临时字符串的场景下，byte切片转换为string不需要拷贝内存，而是直接返回一个string，这个string的指针（string.str）指向切片的内存。

### defer

问题：下面的代码会输出什么？

```go
func deferFuncParameter(){
	var aInt = 1
	defer fmt.Println(aInt)

	aaInt = 2
	return
}
```

输出是1，因为fmt.Println(aInt)的参数在defer语句出现时就出现了，一直都是1，这个参数是一个值也不是引用。后面的修改不会影响defer时的值。

每次申请到一个需要释放的资源时，后面立刻追加释放资源的defer语句是一个好习惯。

关键词return不是一个原子操作，实际上执行了汇编指令ret，即执行跳转程序。比如return i，实际上是分两步，先将i值存入栈中作为返回值，然后执行跳转。

return是先保存返回值，执行defer（如果有的话），执行ret跳转

```go
func foo() int{
	var i int
	defer func(){
		i++
  }()
	return i
} // 运行结果是0
```

这里foo函数的返回值是一个匿名返回值，对于匿名返回值，可以假定有一个变量存储了返回值，假设变量名叫annoy，上面的返回语句可以拆分成以下三个过程：

```go
annoy = i
i++
return
```

所以defer中修改i值，对函数返回值不会影响。

那么对于非匿名的返回值，主函数声明一个带名字的返回值，会被初始化成一个局部变量，函数内部可以像使用局部变量一样使用该返回值。上面的函数，改成非匿名返回值运行结果是1，如下：

```go
func foo() (i int) {
	defer func() {
		i++
	}()
	return
}
```

每个goroutine数据结构中实际上也有一个defer指针，该指针指向一个defer的单链表。每次声明一个defer就将defer插入单链表，每次执行defer时就从单链表表头取出一个defer执行。

**已经关闭的channel也是可读的。**

### select

执行下面的函数会有什么结果？

```go
package main

func main(){
	select {
	}
}
```

对于空的select，程序会阻塞，准确说是当前协程被阻塞。但是go自带死锁检测机制，就发现当前协程再也没有机会被唤醒，程序会Panic。

Golang实现select时，定义了一个数据结构存储每个case，default实际上是一种特殊的case，select执行可以类比成一个函数，输入多个case，选出要执行的case输出，然后程序流转到选中的case上。

select语句除了default外，各个case的执行顺序是随机的。

select除了default外，每个case操作一个channel，要么读要么写。

### range

问题：下面的函数能正常结束吗？

```go
func main(){
	v:=[]int{1,2,3}
	for i:=range v{
		v=append(v, i)
	}
}
```

能正常结束，因为v的长度在循环前就确定了，循环内改变切片的长度也不会影响之前的结果。遍历slice前会先获取slice的长度作为循环次数。

对于map的遍历，map底层使用的是hash表实现，hash插入顺序是随机的，所以在循环过程中map中新增元素不一定能遍历到。

编程Tips：遍历过程中放弃接收index和value，可以一定程度提升性能。

**什么是自旋？**自旋对应CPU的“PAUSE”命令，也就是CPU对该指令什么操作也不做，相当于CPU空转。对程序来说是sleep了一小段时间，时间非常短，当前实现是30个时钟周期。协程调度机制中的Process数量必须大于1，比如GOMAXPROCS()将处理器值设置为1就benign启用自旋。自旋的条件是很苛刻的，总而言之就是不忙的时候才会启用自旋。如果自旋过程中获得锁，那么之前被阻塞的协程则无法获取锁，如果加锁的协程特别多，每次都通过自旋获得锁，那么之前被阻塞的进程则很难获得锁，进而进入饥饿状态。

处于饥饿状态下，不会启用自旋。也就是一旦有协程释放了锁，那么一定会唤醒协程，被唤醒的协程将获得锁，同时把等待计数器减1。

重复加锁解锁会Panic，应该避免这样的操作。

读锁不能阻塞读锁：一个协程拥有读锁时，其他协程也可以拥有读锁。

**读写锁是如何阻止读操作的？**

当写锁定进行时，会先把readerCount减去2^30，从而readerCount变成了负数，此时再有读锁定到来时，会检测到readerCount为负数，便知道此时有写操作在执行，只好阻塞等待。而真正的读操作个数并不会丢失，只需要将readCount加上2^30即可获得。所以，写操作时通过将readerCount变成负数来实现的。

## 第三章 协程

在高并发的场景下频繁创建线程会造成不必要的开销，所以有了线程池。线程池预留一定数量的线程，而新任务将不会以创建线程的方式去执行。

Goroutine调度器

Go提供了一种调度机制，可以在线程中自己实现调度，上下文切换更少，从而达到了线程数量更少，而并发数不会少的效果。而线程中调度的就是Goroutine。

Goroutine的主要概念如下：

G（Goroutine）：即Go协程，每个go关键字都会创建一个协程。

M（Machine）：工作线程，在Go中被称为Machine。

P（Processor）：处理器（Go中定义的一个概念，不是指CPU），包含运行Go代码的必要资源，也有调度goroutine的能力。

M必须拥有P才能执行G中的代码，P中包含多个G的队列，**P可以提交G交给M执行。**因为M必须拥有P才能运行Go代码，所以同时运行的M个数，也即线程数一般等于CPU的个数。一般情况下M的个数会略大于P的个数，多出来的M将会在G 产生系统调用的时候发挥作用。空闲的P也会把其他P中空闲的G投过来，每次偷去一半，来实现全局的执行加速。

## 第四章 内存管理

### 自动垃圾回收机制

C语言可以通过malloc()方法动态申请内存，其中内存分配器使用的是glibc提供的ptmalloc2。

span是用于管理arena页的关键因素。

所谓的垃圾就是不再需要的内存块，这些垃圾如果不被清理就没办法再次被分配利用，在不支持垃圾回收的编程语言里，这些垃圾内存就是泄露的内存。

为了防止内存分配过快，在GC的过程中，如果goroutine需要分配内存，那么这个goroutine会参与一部分GC的工作，即帮GC做一部分工作，这个机制叫Mutator Assist。

程序代码可以手动执行runtime.GC()来手动触发GC。

### 逃逸分析

所谓逃逸分析（Escape  analysis）是指由编译器决定内存分配的位置，不需要程序员指定。函数中申请一个新对象：

- 如果分配在栈中，则函数执行结束后可自动将内存回收。
- 如果分配在堆中，则函数执行结束后可交给GC（垃圾回收）处理。

对于函数外部没有引用的对象，也有可能放到堆中，比如内存过大超过栈的存储能力。

下面是一个指针逃逸分析的场景：

Go可以返回局部变量地址，这是一个典型的变量逃逸的例子。示例代码如下：

```go
package main

type Student struct{
	Name string
	Age string
}

func StudentRegister(name string, age int) *Student{
	s := new(Student) // 局部变量s逃逸到堆
	s.Name =name
	s.Age = age
	return s
}

func main(){
	StudentRegister("Jim", 19)
}
```

StudentRegister里的s是局部变量，其值通过函数返回值返回，s本身是一个指针，其指针指向的内存地址不会是栈而是堆，这就是典型的逃逸案例。

通过编译参数-gcflag=-m可以查编译过程中的逃逸分析。

总结：逃逸分析在编译阶段完成。

传递指针可以减少底层值的拷贝，可以提高效率，如果拷贝的数据量小，由于指针可能会产生逃逸，然后可能就会使用堆，增加GC负担，所以传递指针也不一定是高效的。

## 第五章 并发控制

### context

Golang context是Golang常用的并发控制技术，它与WaitGroup的最大不同就是context对于派生的goroutine有更强的控制能力，它可以控制多级的goroutine。context翻译成中文就是上下文，它可以控制一组呈树状的goroutine，每个goroutine都有相同的上下文。比如某个goroutine派生来子goroutine，子goroutine派生出了新的子goroutine，此时使用WaitGroup就不方便，因为子goroutine的数量容易确定，此时使用context就很容易实现。

context实际上只定义了接口，只要实现了这个接口的类都可以是一种context，context的接口定义如下：

 ```go
type Context interface{
	Deadline() (deadling time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
}
 ```

当context关闭时，Done()返回了一个被关闭的管道，关闭的管道依然是可读的，据此goroutine可以收到关闭请求；当context还没有关闭时，Done()返回nil。

当context关闭时，Err()返回context的关闭原因。当context还没有关闭时，Err()返回nil。

context提供了4种方法创建不同类型的context，使用这4种方法如果没有父context，都需要传入background，即background作为其父节点：

- WithCancel()
- WithDeadline()
- WithTimeout()
- WithValue()

下面介绍一些实现了context的struct，比如valueCtx:

```go
type valueCtx struct{
	Context
	key, val interface{}
}
```

valueCtx只是在Context的基础上加了一个key-val对，用于在各级协程之间传递一些数据。由于valueCtx不需要cancel，也不需要deadline，那么只需要实现Value()接口即可。

下面是一个使用valueCt的案例：

```go
func HandleRequest(ctx context.Context){
	for{
		select {
			case <-ctx.Done():
				fmt.Println("HandleRequest Done.")
				return
			default:
				fmt.Println("HandleRequest running, parameter: ", ctx.Value("parameter"))
				time.Sleep(2*time.Second)
		}
	}
}

func main(){
	ctx:=context.WithValue(context.Background(), "parameter", "1")
	go HandleRequest(ctx)
	
	time.Sleep(10*time.Second)
}
```

本例中子协程无法结束，因为context是不可以cancel的，也就是<-ctx.Done()永远无法返回。如果需要返回，则需要在创建context的时候指定一个可以cancel的context的作为父节点，使用父节点的cancel()在适当时机结束整个context。

## 第六章 反射

Go是静态类型语言，也就是每个变量都有一个静态类型，而且是在编译时就确定好了的。

interface代表了一种特殊的类型，它代表方法集合，它可以存放任何实现了该方法的值。空interface类型的方法集合是空，也就是说所有类型都可以认为是实现了该接口。

至于interface类型是如何表示的，可以查看下面的示例代码：

```go
type r io.Reader
tty, err:=os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err!=nil{
	return nil, err
}
r = tty
```

r的类型始终都是io.Reader interface类型，无论其存储的是什么值。

那么File类型体现在哪里呢？答案是r保存了一对(value, type)对来表示其存储的信息，value即r所持有的元素的值，type即其所持有的元素的底层值。

**interface类型是有个(value, type)对的，而反射就是检查interface的这个(valuel type)对的。**

下面的程序可以输出interface存储的类型和值：

```go
func main() {
	var x float64 = 3.4
	t:=reflect.TypeOf(x) // t is reflect.Type
	fmt.Println("type:", t)
	
	v:=reflect.ValueOf(x) // x is reflect.Value
	fmt.Println("value:", v)
}
```

反射是针对interface类型变量的，TypeOf()和ValueOf()接受的参数都是interface类型的，也就是x值是被转换成interface类型然后传入的。

再看下面的：

```go
func main(){
	var x float64 = 3.4
	v:=reflect.ValueOf(x) // v is reflect.Value
	var y float64 = v.Interface().(float64)
	fmt.Println("value:", y)
}
```

对象x转换成反射对象v，v又通过Interface()接口转换成interface对象。

reflect.ValueOf函数传入的其实是x的值，而非x本身。所以通过修改v是无法影响到x的。如果reflect.Value传入的是x的地址，那么之后修改v就会影响原始的x了。

reflect.Value提供了Elem()方法，可以获取指针指向的value，如下：

```go
func main(){
	var x float64=3.4
	v:=reflect.ValueOf(&x)
	v.Elem().SetFloat(7.1)
	fmt.Println("x:", v.Elem().Interface())
}
```

输出结果是：`x: 7.1`

## 第七章 go test

测试文件可以和源文件在同一个包，但是常见的做法是创建一个包专门用于测试，这样可以使得测试文件和源文件隔离。GO源代码以及其他知名开源框架通常会创建测试包，测试的包的名字是原包的名字加上"_test"。

测试函数的命名规则是"Test__"，其中Test是单元测试的固定开头。

除了单元测试，还有种测试叫例子测试，例子测试函数名需要以"Example"开头。

此外还有TestMain。如下：

```go
func TestMain(m *testing.M){
	println("TestMain setup.")
	retCode := m.Run() // 执行测试，包括单元测试，性能测试和示例测试
	println("TestMain tear-down.")
	os.Rxit(retCode)
}
```

日志打印的两行分别对应Setup和Tear-Down代码。m.Run()即为执行所有的测试，m.Run()的返回结果通过os.Exit()返回。

testing.T和testing.B属于testing包中的两个数据类型，该类型提供一系列的方法用于控制函数执行流程，考虑到二者有一定的相似性，所以Go实现时抽象出了一个testing.common作为一个基础类型。而testing.T和testing.B则属于testing.common的扩展。

common有很多方法，比如common.hasSub标记当前测试是否包含子测试，当测试使用t.Run()方法启动子测试时，t.hasSub则置为1。

common还有一个方法叫common.Log()用于记录简单日志，通过fmt.Sprintln()方法生成日志字符串后记录。

性能测试在代码里定义的结构体名字叫B，其有一个停止计时的函数：B.StopTimer()，StopTimer()负责停止计时，并累加响应的统计值。

```go
func (b *B) StopTimer(){
	if b.timderOn{
		b.duration = time.Since(b.Start) // 累加测试耗时
		runtime.ReadMemStats(&memStats) // 读取当前栈内存分配信息
		b.netAllocs +=memStats.Mallocs = b.startAllocs // 累加堆内存分配的字节数
		b.timerOn=false // 标记计时标志
	}
}
```

StopTimer()并不一定是测试结束，一个测试中可能有多个统计阶段，所以其统计值是累加的。

性能测试的测试结果以最后一次迭代的数据为准，当然最后一次迭代则表示b.N更大，测试准确性更高。

关于测试结果的比较：输出有序的情况下，比较很简单，就是比较两个String的内容是否一致，无序的情况就是把String排序后再比较。

源代码里定义了一个testing.M的数据结构，单元测试、性能测试和示例测试在经过编译后都会被存放到一个testing.M的数据结构中，在执行测试时，该数据结构将传递给TestMain()，真正执行测试的是testimg.M的Run()方法。

**TestMain()函数通常有一个m.Run()函数，该方法会执行单元测试、性能测试和示例测试。**如果用户实现了TestMain()但是没有调用m.Run()的话，则什么测试都不会被执行。

m.Run()不仅会执行测试，还会做一些初始化的工作，比如解析参数、启动定时器、根据工具参数指示创建一系列的文件等。

m.Run()会使用三个独立的方法来执行三种测试：

- 单元测试：runTest(m.deps.MatchString, m.tests)
- 性能测试：runExample(m.deps.MatchString, m.examples)
- 示例测试：runBenchmarks(m.deps.ImportPath(), m.deps.MatchString, m.benchmarks)其中m.deps存放了测试匹配相关的内容。

包列表模式是从Go1.10版本才引入的，它会把每个包的测试结果写入到本地临时文件作为缓存，下次执行时直接从缓存读取结果，从而节省测试时间。

go test有一个参数叫-cpu，比如-cpu 1,2,4，-cpu提供了一个CPU个数的参数，提供此列表后，那么测试将按照这个列表指定的CPU个数设置GOMAXPROCS并分别测试。

```
func AfterFuncDemo(){
	log.Println("AfterFuncDemo start:", time.Now())
	time.After(1*time.Second, func(){
		log.Println("AfterFuncDemo end:", time.Now())
	})
	time.Sleep(2*time.Second) // 等待协程退出
}
```

time.AfterFunc()是异步执行的，所以需要在函数最后sleep等待指定的协程推出，否则可能函数结束后协程还没有执行。

## 第八章 定时器

每个Go应用程序都有一个协程专门负责管理所有的Timer。

Timer的数据结构定义如下：

```go
type Timer struct{
	C <- chan Time
	r runtimeTimer
}
```

这里的C是面向Timer用户的，Timer.r是面向底层的定时器实现。

每创建一个Timer意味着创建一个runtimeTimer变量，然后把它交给系统进行监控。我们通过设置runtimeTimer过期后的行为达到定时的目的。

启动Timer的函数实现如下：

```go
func NewTimer(d Duration) *Timer{
	c:=make(chan Time, 1) // 创建一个管道
	t:=&Timer{
		when: when(d), // 触发事件
		f: sendTime, // 触发后执行函数sendTime
		arg: c, // 触发后执行函数sendTime时附带的参数
	}
  startTimer(&t.r) // 此时启动定时器，只是runtimeTimer放到系统协程的堆中，由系统协程维护
}
```

Ticker的数据结构和Timer完全一致：

```go
type Ticker struct{
	C <-chan Time
	r runtimTimer
}
```

Ticker对外仅暴露一个channel，指定的时候到来时就往该channel中写入系统时间，也就是一个事件。

定时器Timer和周期性定时器Ticker，这两种定时器的内部实现机制完全相同。创建定时器的协程并不负责计时，而是把任务交给系统协程，系统协程统一处理所有的定时器。系统协程在首次创建定时器时创建，定时器存储在切片中，系统协程负责计时并维护这个切片。

创建定时器的运作机制：

创建Timer或者Ticker实际上分为两步：

1. 创建一个管道
2. 创建一个timer并启动（这里是timer而不是Timer，这里的timer时系统所管理的timer）

每个timer都必须归属到某个timersBucket，第一步就是选择一个timersBucket，选择的算法很简单：将当前协程所属的Processor ID和timersBucket数组长度求模，结果就是timersBucket数组的下标。

每个timer都必须要加入到timersBucket中，timerBucket中切片保存着timer的指针，新加入的timer并不是按照加入时间顺序存储的，**而是把timer按照触发的时间排序成一个小头堆。**在小头堆里，每个圆圈代表一个timer，圆圈里的数字代表距离触发事件的秒数，圆圈外的我数字代表其在切片中的下标。新加入timer会触发小头堆的排序。

不管什么情况，创建一个Ticker后，在后面追加defer语句来关闭Ticker是一个好习惯。

## 第九章 语法糖

语法糖（Syntactic sugar）的概念是英国计算机科学家提出的，表示编程语言中某种类型的语法，这类语法不会影响功能，但是使用起来很方便。

语法糖的其中一个用法就是简短变量声明，比如`field, offset := nextField(str, 0)`。

比如：

```go
field1, offset := nextField(str, 0)
field2, offset := nextField(str, 1)
```

:= 实现的效果是重新声明，这里的offset被重新声明，重新声明并没有问题，没有引入新变量，只是把变量的值给改掉了。

在`:=`左侧没有新变量是不允许的。

可以理解`:=`实际上会被拆分成两个部分，即声明和赋值，赋值语句不能出现在函数外部。

关于可变参函数：

```go
func Greeting(prefix string, who ...string){
	if who == nil{
		fmt.Println("nobody to say hi")
		return
	}
	for _,people :=range who{
		fmt.Prinf("%s %s\n", prefix, people)
	}
}
```

这里的who就是可变参数，可变参数在函数内部是作为切片来解析的。

切片传入时不会生成新的切片，也就是函数内部使用的切片和传入的切片共享相同的存储空间。也就是函数内部修改了切片的值，可能影响外部调用的函数。

