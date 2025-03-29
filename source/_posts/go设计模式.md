---
title: go设计模式
date: 2022-8-15 00:00:00
categories:
- golang
---
[原文](https://studygolang.com/articles/18059)

Go语言并非是像C++和Java一样的面向对象语言，但是设计模式同样适用

## 创建型Creational Design Patterns

### 单例Singleton

主要用于**保证一个类仅有一个实例，并提供一个访问它的全局访问点**。

*如何判断一个对象是否应该被建模成单例？*

通常，被建模成单例的对象都有“**中心点**”的含义，比如线程池就是管理所有线程的中心。所以，在判断一个对象是否适合单例模式时，先思考下，这个对象是一个中心点吗？

> 需要初始化一次
>
> mulock   *sync.Mutex 初始化的时候可以通过mulock:   new(sync.Mutex) 赋值
>
> sync.Once只会执行一次

### 建造者Builder

如果某个结构体初始化的时候**有很多字段需要赋值**，一个个赋值字段太多，可以先初始化空的结构体，然后写多个方法，挨个给各字段赋值，类似慢慢建造这个结构体的模式。

```go
 type Message struct {
 Header *Header
 Body   *Body
 }
 type Header struct {
 SrcAddr  string
 SrcPort  uint64
 DestAddr string
 DestPort uint64
 Items    map[string]string
 }
 type Body struct {
 Items []string
 }
 // 多层的嵌套实例化
 message := msg.Message{
 Header: &msg.Header{
 SrcAddr:  "192.168.0.1",
 SrcPort:  1234,
 DestAddr: "192.168.0.2",
 DestPort: 8080,
 Items:    make(map[string]string),
 },
 Body:   &msg.Body{
 Items: make([]string, 0),
 },
 }
 // 需要知道对象的实现细节
 message.Header.Items["contents"] = "application/json"
 message.Body.Items = append(message.Body.Items, "record1")
```

改成：

```go
// Message对象的Builder对象
 type builder struct {
 once *sync.Once
 msg *Message
 }
 // 返回Builder对象
 func Builder() *builder {
 return &builder{
 once: &sync.Once{},
 msg: &Message{Header: &Header{}, Body: &Body{}},
 }
 }
 // 以下是对Message成员对构建方法
 func (b *builder) WithSrcAddr(srcAddr string) *builder {
 b.msg.Header.SrcAddr = srcAddr
 return b
 }
 func (b *builder) WithSrcPort(srcPort uint64) *builder {
 b.msg.Header.SrcPort = srcPort
 return b
 }
...
func main(){
   message := msg.Builder().
 WithSrcAddr("192.168.0.1").
 WithSrcPort(1234).
 WithDestAddr("192.168.0.2").
 WithDestPort(8080).
 WithHeaderItem("contents", "application/json").
 WithBodyItem("record1").
 WithBodyItem("record2").
 Build()
 if message.Header.SrcAddr != "192.168.0.1" {
 t.Errorf("expect src address 192.168.0.1, but actual %s.", message.Header.SrcAddr)
 }
}
```

### 工厂Factory Design

都是**将对象创建的逻辑封装起来，为使用者提供一个简单易用的对象创建接口**。与建造者模式类似，但是建造者模式适用于更多参数的情况。

```go
// 这个是创建处，iGun是一个interface
func getGun(gunType string) (iGun, error) {
    if gunType == "ak47" {
        return newAk47(), nil
    }
    if gunType == "maverick" {
        return newMaverick(), nil
    }
    return nil, fmt.Errorf("Wrong gun type passed")
}
```

### 抽象工厂 Abstract Factory

在工厂方法模式中，我们通过一个工厂对象来创建一个产品族，具体创建哪个产品，则通过`swtich-case`的方式去判断。这也意味着该产品组上，每新增一类产品对象，都必须修改原来工厂对象的代码；而且随着产品的不断增多，工厂对象的职责也越来越重，违反了**单一职责原则**。

抽象工厂模式通过给工厂类新增一个抽象层解决了该问题，

该模式让您创建一系列相关对象。

应该就是实现了某个interface的两个struct都嵌套了一个共同的struct？？？

````go
type adidasShort struct {
    short
}
type nikeShort struct {
    short
}
type iShort interface {
    setLogo(logo string)
    setSize(size int)
    getLogo() string
    getSize() int
}
type short struct {
    logo string
    size int
}

func (s *short) setLogo(logo string) {
    s.logo = logo
}
// short实现了iShort的所有接口，adidasShort和nikeShort不需要再实现任何接口了
````

### 原型Prototype

主要解决对象复制的问题，它的核心就是`clone()`方法，clone返回interface然后对interface强制转换

```go
 // 原型复制抽象接口
 type Prototype interface {
 clone() Prototype
 }
 type Message struct {
 Header *Header
 Body   *Body
 }
 func (m *Message) clone() Prototype {
 msg := *m
 return &msg
 }
func main(){
  newMessage := message.Clone().(*msg.Message)
}
```

## **结构模式**（Creational Pattern）

### 组合 Composite

适用于文件-目录这种树状结构，比如搜索操作，目录会包括多个文件，会调用文件的search操作。

### 适配器模式 Adapter

适配器模式所做的就是**将一个接口`Adaptee`，通过适配器`Adapter`转换成Client所期望的另一个接口`Target`来使用**，实现原理也很简单，就是`Adapter`通过实现`Target`接口，并在对应的方法中调用`Adaptee`的接口实现。

不适配的struct命名为xxxAdapter

````go
type computer interface {
    insertInSquarePort()
}

type mac struct {
}

func (m *mac) insertInSquarePort() {
    fmt.Println("Insert square port into mac machine")
}

type windowsAdapter struct {
	windowMachine *windows
}

func (w *windowsAdapter) insertInSquarePort() {
	w.windowMachine.insertInCirclePort()
}

type windowsAdapter struct {
	windowMachine *windows
}
// 增加了windowsAdapter，可以调用insertInSquarePort用于mac和windows的调用
func (w *windowsAdapter) insertInSquarePort() {
	w.windowMachine.insertInCirclePort()
}
````

### 桥接Bridge

m*n优化为m+n

举一个例子，一个产品有形状和颜色两个特征（变化方向），其中形状分为方形和圆形，颜色分为红色和蓝色。如果采用继承的设计方案，那么就需要新增4个产品子类：方形红色、圆形红色、方形蓝色、圆形红色。如果形状总共有m种变化，颜色有n种变化，那么就需要新增m*n个产品子类！现在我们使用桥接模式进行优化，将形状和颜色分别设计为一个抽象接口独立出来，这样需要新增2个形状子类：方形和圆形，以及2个颜色子类：红色和蓝色。同样，如果形状总共有m种变化，颜色有n种变化，总共只需要新增m+n个子类！

类似积木，可以自由的组合和拼接

### 装饰器Decorator

它允许您提供附加功能或装饰对象而不更改该对象

装饰者也就是附加的部分嵌套了原来的东西 

这里的例子是披萨的增加调料的实现

```go
type pizza interface {
	getPrice() int
}

type cheeseTopping struct {
	pizza pizza
}

func (c *cheeseTopping) getPrice() int {
	pizzaPrice := c.pizza.getPrice()
	return pizzaPrice + 10
}
```

### 正面Facade

将复杂逻辑封装在被调用端，调用端可以无脑调，两边都不用定义interface就定义方法即可

### 蝇量级Flyweight

当需要创建大量可能导致内存问题的对象时使用

### 代理proxy

为对主要对象的受控和智能访问提供额外的间接层。

在这种模式中，创建了一个新的代理类，它实现了与主对象相同的接口。 这使您可以在主对象的实际逻辑之前执行一些行为。 比如nginx服务器会在实际处理前做一些处理（比如统计API调用次数等）。

下面的例子中，实际的nginx是被使用的，它嵌套了业务处理application，也实现了handleRequest，在handleRequest中做了自定义的前置处理checkRateLimiting然后调用application的handleRequest。适用方直接用最外层的nginx的handleRequest即可。

````go
type server interface {
    handleRequest(string, string) (int, string)
}

type application struct {
}

func (a *application) handleRequest(url, method string) (int, string) {
    ....
}

type nginx struct {
    application       *application
    maxAllowedRequest int
    rateLimiter       map[string]int
}
func newNginxServer() *nginx {
    return &nginx{
        application:       &application{},
    }
}
func (n *nginx) handleRequest(url, method string) (int, string) {
    allowed := n.checkRateLimiting(url)
    if !allowed {
        return 403, "Not Allowed"
    }
    return n.application.handleRequest(url, method)
}
func (n *nginx) checkRateLimiting(url string) bool {
}

func main() {
    nginxServer := newNginxServer()
    appStatusURL := "/app/status"
    httpCode, body := nginxServer.handleRequest(appStatusURL, "GET")
}
````

##行为设计模式Behavioural Design Patterns

### 责任链Chain of Responsiblity

责任链设计模式是一种行为设计模式。 它允许您创建一系列请求处理程序。 对于每个传入的请求，它都通过链和每个处理程序传递：

处理请求或跳过处理。
决定是否将请求传递给链中的下一个处理程序

```go
// 以病人去医院看病为例子
type department interface {
    execute(*patient)
    setNext(department)
}

type reception struct {
    next department
}
func (r *reception) execute(p *patient) {
	m.next.execute(p)
}
func (r *reception) setNext(next department) {
    r.next = next
}
type cashier struct {
	next department
}
func (c *cashier) execute(p *patient) {
	fmt.Println("Cashier getting money from patient patient")
}
func (c *cashier) setNext(next department) {
	c.next = next
}

type patient struct {
}
func main() {
    cashier := &cashier{}
    medical := &medical{}
    medical.setNext(cashier)
    doctor := &doctor{}
    doctor.setNext(medical)
    patient := &patient{name: "abc"}
    //Patient visiting
    reception.execute(patient)
}
```

### 命令Command



### 迭代器Iterator

迭代器设计模式是一种行为设计模式。 在这种模式中，集合结构提供了一个迭代器，它允许它按顺序遍历集合结构中的每个元素，而不会暴露其底层实现。

### 调解员Mediator

中介者设计模式是一种行为设计模式。 这种模式建议创建一个中介对象来防止对象之间的直接通信，从而避免它们之间的直接依赖关系。

### 备忘录Memento

Memento 设计模式是一种行为设计模式。 它允许我们保存对象的检查点，从而允许对象恢复到之前的状态。 基本上它有助于对对象进行撤消重做操作。

### 空对象Null Object

空对象设计模式是一种行为设计模式。 当客户端代码依赖于某些可以为空的依赖项时，它很有用。 使用这种设计模式可以防止客户端对这些依赖项的结果进行空检查。 话虽如此，还应该注意的是，客户端行为也适用于此类空依赖项。

### 订阅者Observer

适用于一个订阅者，多个发布者。订阅者订阅某个主题（subject）。

### 状态State

作用于有限状态机

###策略Strategy

改变对象的行为，比如根据实际情况的变化动态的调整缓存策略，所谓的调整缓存策略还是得需要代码里手动set

### 模板方法Template Method

定义多个有顺序要求的接口，调用处按顺序调用接口

### 访问者Visitor

给struct增加一些行为，而不用改变struct

比如给interface多加一些方法，不用修改实现的struct，通过visitor修改。

