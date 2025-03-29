---
title: gohangout项目代码阅读总结
date: 2022-8-17 00:00:00
categories:
- golang
---
[原文](https://studygolang.com/articles/18059)

## 功能

**gohangout类似logstash，从输入（ES，Kafka，Stdin等）读取数据，经过一些过滤和加工，将处理好的数据输出到指定输出中（ES，Stdin等）**

## 代码可学习

### 多个flag的结构化

**如果有多个flag参数，可以把参数的读取定义在init()中，这些参数在main所在文件里定义成一个option**

**注意option是个指针**

```
var options = &struct {
config     string
autoReload bool // 配置文件更新自动重启
pprof      bool
pprofAddr  string
cpuprofile string
memprofile string

prometheus string

exitWhenNil bool
}{}

func init() {
flag.StringVar(&options.config, "config", options.config, "path to configuration file or directory")
flag.BoolVar(&options.autoReload, "reload", options.autoReload, "if auto reload while config file changed")

flag.BoolVar(&options.pprof, "pprof", false, "pprof or not")
flag.StringVar(&options.pprofAddr, "pprof-address", "127.0.0.1:8899", "default: 127.0.0.1:8899")
flag.StringVar(&options.cpuprofile, "cpuprofile", "", "write cpu profile to `file`")
flag.StringVar(&options.memprofile, "memprofile", "", "write mem profile to `file`")

flag.StringVar(&options.prometheus, "prometheus", "", "address to expose prometheus metrics")

flag.BoolVar(&options.exitWhenNil, "exit-when-nil", false, "triger gohangout to exit when receive a nil event")

flag.Parse()
}
```

### 日志

**日志打印小写开头**

### 配置文件转换

**定义了一个Parser interface**

```
type Parser interface {
parse(filename string) (map[string]interface{}, error)
}
```

**虽然只有一个YamlParser实现了：**

```
type YamlParser struct{}

func (yp *YamlParser) parse(filepath string) (map[string]interface{}, error) {
  ...
}
```

**读取配置文件根据文件的大小读取：**

```
configFile, err := os.Open(filepath)
if err != nil {
return nil, err
}
fi, _ := configFile.Stat()

if fi.Size() == 0 {
return nil, fmt.Errorf("config file (%s) is empty", filepath)
}

buffer = make([]byte, fi.Size())
_, err = configFile.Read(buffer)
```

### 小的代码书写注意项

* **多个变量声明**

```
var (
buffer []byte
err    error
)
```

* **这个项目叫gohangout，main函数被定义在gohangout.go里**
* **返回的被包装的错误也是小写字母开头的，错误包装例子** `fmt.Errorf("config file (%s) is empty", filepath)`
* **同一个文件中，被调用的函数定义在调用处前面，靠近调用处**
* **不用公开访问的函数小写字母开头即可**
* **文件名，用下划线分割，http_input.go，tcp_input.go这样**
* **可以加上go build flag，这样在一个文件下可以有同名的函数，比如**

  ```
  // +build linux darwin
  
  func listenSignal() {}
  ```

  ```
  // +build linux darwin
  
  func listenSignal() {}
  ```
* **多个返回值命名** `func getUserPasswordAndHost(url string) (scheme, user, password, host string)`

## 一些工具

**pprof**是golang 官方提供的性能调优分析工具，可以对程序进行性能分析，并可视化数据，看起来相当的直观。 当你的go 程序遇到性能瓶颈时，可以使用这个工具来进行调试并优化程序

**go plugin**：需要 `go build -buildmode=plugin -o=plugin_doctor.so plugin_bad_docter.go`这样先编译成so文件，然后使用处执行 `plugin.Open("xxx.so")`，Open完毕才会执行插件的init函数，然后调用 `plug.Lookup("Doctor")`，再将结果转换为自己需要定义的interface等再使用。总而言之，如何使用还得自己定义，和使用相关的方法等还得自己定义，虽然被编译的go文件里可能已经定义了。so文件就像一个黑盒子。

**用到了**[https://github.com/childe/healer](https://github.com/childe/healer)，这是作者写的一个消费者Group Consumer

**Grok**：解析日志数据时最常见的任务是将原始文本行分解为其他工具可以操作的一组结构化字段。 如果你使用 Elastic Stack，则可以利用 Elasticsearch 的聚合和 Kibana 的可视化，从日志中提取的信息（如 IP 地址，时间戳和特定域的数据）解释业务和操作问题。对于 Logstash，这个解构工作由 [logstash-filter-grok](https://link.segmentfault.com/?enc=qL2t%2BD3iTe4xOINVN8omxg%3D%3D.3vfUTFyLK%2FHOPhM16wcpIivEyOePYxpfZ6aPYXHss7dXpRZqHHChG3womHjDEFQI%2FvBjMXm71DO2Wl8vSTY2k0a3AWlt95iOqD4NkBEYtf8%3D) 来承担，它是一个过滤器插件，可以帮助你描述日志格式的结构。grok 过滤器用于将非结构化数据解析为结构化数据。简而言之，grok是一种将行与正则表达式匹配，将行的特定部分映射到专用字段中以及根据此映射执行操作的方法。简而言之，grok是一种将行与正则表达式匹配，将行的特定部分映射到专用字段中以及根据此映射执行操作的方法。

**map修改是会互相影响的**：

```
func main() {
e := make(map[interface{}]interface{})
e[1] = 1
f(e)
fmt.Println(e)
}

func f(m map[interface{}]interface{}) {
m[2] = 2
fmt.Println(m)
}
```

**运行结果为：**

```
map[1:1 2:2]
map[1:1 2:2]
```

**go的text/template包**

## 每个包里的定义

### codec包

```
type Decoder interface {
Decode([]byte) map[string]interface{}
}
```

### condition_filter包

```
type Condition interface {
Pass(event map[string]interface{}) bool
}
```

### field_deleter包

```
type FieldDeleter interface {
Delete(map[string]interface{})
}
```

### field_setter包

```
type FieldSetter interface {
SetField(event map[string]interface{}, value interface{}, fieldName string, overwrite bool) map[string]interface{}
}
```

### filter包

```
type Converter interface {
convert(v interface{}) (interface{}, error)
}
type DateParser interface {
Parse(interface{}) (time.Time, error)
}
```

### input包

```
type Converter interface {
convert(v interface{}) (interface{}, error)
}
type DateParser interface {
Parse(interface{}) (time.Time, error)
}
```

### output包

```
type Event interface {
Encode() []byte
}

type BulkRequest interface {
add(Event)
bufSizeByte() int
eventCount() int
readBuf() []byte
}
type NewBulkRequestFunc func() BulkRequest

type BulkProcessor interface {
add(Event)
bulk()
awaitclose(time.Duration)
}

type HostSelector interface {
Next() interface{}
ReduceWeight()
AddWeight()
Size() int
}
```

### simplejson包

```
type SimpleJsonDecoder struct {
bytes.Buffer
scratch [64]byte
}

type JSONMarshaler interface {
MarshalJSON() ([]byte, error)
}

func (d *SimpleJsonDecoder) string(s string) int {}
```

### **topology包（拓扑）**

**这个包里定义了：**

```
type Filter interface {
Filter(map[string]interface{}) (map[string]interface{}, bool)
}
type Input interface {
ReadOneEvent() map[string]interface{}
Shutdown()
}
type Output interface {
Emit(map[string]interface{})
Shutdown()
}
type Processor interface {
Process(map[string]interface{}) map[string]interface{}
}
```

### value_render包

```
type ValueRender interface {
Render(map[string]interface{}) interface{}
}
```

## 结构总结

**input包的init函数有**

```
Register("Kafka", newKafkaInput)
Register("Random", newRandomInput)
Register("Stdin", newStdinInput)
Register("TCP", newTCPInput)
Register("UDP", newUDPInput)
```

**topology没有init函数，**

### 调用大纲

**main调用buildPluginLink构建input，**

```
boxes, err := buildPluginLink(config)
```

> **buildPluginLink详情：**
>
> **根据配置input中的类型获得对应的topology.Input**
>
> **根据topology.Input初始化InputBox struct**
>
> **根据配置的****add_fields**，调用field_setter.NewFieldSetter(k)，给InputBox struct增加addFields即b.addFields[fieldSetter] = value_render.GetValueRender(v.(string))
>
> **给 InputBox 的shutdownWhenNil = shutdownWhenNil赋值**

**将boxes转换：**

```
inputs = gohangoutInputs(boxes)
```

**调用** `input.start()`，其具体逻辑就是将input强制转换为 `boxes := ([]*input.InputBox)(inputs)`，依次调用boexes的Beat，如下：

**main中使用了InputBox这个struct，通过每个InputBox的Beat启动，Beat调用beat，beat的内容包括：**

**buildTopology返回了一个Node**

> **buildTopology详情：**
>
> **读取配置中的output，**`box.outputsInAllWorker[workerIdx] = outputs`，根据output获取outputProcessor，
>
> **读取 filter信息，**`filterBoxes := topology.BuildFilterBoxes(box.config, filter.BuildFilter)`，filterBoxes 是 []*FilterBox
>
>> **BuildFilterBoxes详情**
>>
>> **根据filters的各个类型，获得对应的 topology.Filter interface**
>>
>> **NewFilterBox获得FilterBox struct，这个步骤还包括根据配置中的if设置conditionFilter，根据配置的failTag设置failTag，根据remove_fields，****add_fields**设置等
>>
>
> **根据filterBoxes和outputProcessor，构建拓扑，返回拓扑**
>
> ```
>   var firstNode *topology.ProcessorNode // ProcessorNode是个struct
> for _, b := range filterBoxes {
> firstNode = topology.AppendProcessorsToLink(firstNode, b)
> }
> firstNode = topology.AppendProcessorsToLink(firstNode, outputProcessor)
> ```

**对于每个box（一个box对应一个input）。首先获取box.input.ReadOneEvent()，然后对于读取出来的结果，增加addFields**

```
for fs, v := range box.addFields {
event = fs.SetField(event, v.Render(event), "", false)
}
```

**接下来递归调用Process**

```
for fs, v := range box.addFields {
event = fs.SetField(event, v.Render(event), "", false)
}
firstNode.Process(event)
```

**主要是执行** `func (b *FilterBox) Process(event map[string]interface{}) map[string]interface{}`和 `func (p *OutputBox) Process(event map[string]interface{}) map[string]interface{}`或者 `func (p OutputsProcessor) Process(event map[string]interface{}) map[string]interface{}`

**单个****OutputsProcessor**的调用过程（**OutputsProcessor**是在有多个output配置的情况下使用的）：

```
type OutputsProcessor []*OutputBox
```

**挨个调用OutputBox，每个OutputBox都会调用Pass和Emit，和上面的类似。**

**FilterBox**的调用：

**b.conditionFilter.Pass(event) 后调用**

```
event, rst = b.Filter.Filter(event)
if event == nil {
return nil
}
event = b.PostProcess(event, rst)
```

**PostProcess会根据FilterBox的conditionFilter，failTag，removeFields等字段，增加或者删除一些字段。也就是过滤完毕后的字段处理。**

### 函数详情

**topology包的**

```
type Input interface {
ReadOneEvent() map[string]interface{}
Shutdown()
}
```

**实现这些接口的地方在input包内。**

**一些实现Input的struct在返回前会调用Decode，比如**

```
func (p *RandomInput) ReadOneEvent() map[string]interface{} {
return p.decoder.Decode([]byte(strconv.Itoa(n)))
}
```

**关于Decode的定义在codec/decoder.go里：**

```
type Decoder interface {
Decode([]byte) map[string]interface{}
}
```

**PlainDecoder，JsonEncoder等都实现了这个interface**

**Filter的配置读取是根据类型读取到**

```
type Filter interface {
Filter(map[string]interface{}) (map[string]interface{}, bool)
}
```

**上的，然后再根据单个filter的配置读取为FilterBox。根据failTag，if，remove_fields等读取到FilterBox的conditionFilter，failTag，removeFields等字段中，之前读取的Filter为FilterBox的Filter字段。**

`firstNode.Process(event)`里单个FilterBox的调用过程：

**先通过conditionFilter查看event是否满足条件，满足条件的情况下才继续执行。接下来调用Filter的Filter方法（其实是个interface）。至于Filter方法的实现，会涉及到field_setter和value_render，**

```
type FieldSetter interface {
SetField(event map[string]interface{}, value interface{}, fieldName string, overwrite bool) map[string]interface{}
}
type ValueRender interface {
Render(map[string]interface{}) interface{}
}
```

**其中一个Filter的调用处为：**

```
func (plugin *AddFilter) Filter(event map[string]interface{}) (map[string]interface{}, bool) {
for fs, v := range plugin.fields {
event = fs.SetField(event, v.Render(event), "", plugin.overwrite)
}
return event, true
}
```

**会修改传入的event，再把修改完毕的event返回。**

**将处理完毕的event调用** `event = b.PostProcess(event, rst)`。rst是Filter返回的布尔值。

### 配置更新

**main中开启了线程，该线程序一旦从configChannel接受到了变更，将执行：**

```
    inputs.stop()
boxes, err := buildPluginLink(cfg)
if err == nil {
inputs = gohangoutInputs(boxes)
go inputs.start()
} else {
glog.Errorf("build plugin link error: %v", err)
exit()
}
```

**通过三方viper，监视配置，监听到了配置的变更后，将变更后的配置发送到configChannel这个channel中。这个没有开启单独线程。**

### 程序停止

**listenSignal中接收到停止信号，调用每个InputBox的Shutdown，Shutdown做了如下操作：**

**执行box.input.Shutdown()，然后挨个调用Output的o.Output.Shutdown()**

### 零碎点整理

**condition_filter的Condition interface被output/elasticsearch_output.go使用，condition_filter/parse.go使用。**

**output包里的struct里用到了codec.Encoder，intput包里的struct里用到了codec.Decoder，codec.Encoder和codec.Decoder似乎也只在这两个地方用。**

### 一些实现了interface的struct和原生struct

**实现了 topology.Input interface：**

```
type KafkaInput struct {
config         map[interface{}]interface{}
decorateEvents bool

messages chan *healer.FullMessage

decoder codec.Decoder

groupConsumers []*healer.GroupConsumer
consumers      []*healer.Consumer
}
```

```
type InputBox struct {
config             map[string]interface{} // whole config
input              topology.Input
outputsInAllWorker [][]*topology.OutputBox
stop               bool
once               sync.Once
shutdownChan       chan bool

promCounter prometheus.Counter

shutdownWhenNil    bool
mainThreadExitChan chan struct{}

addFields map[field_setter.FieldSetter]value_render.ValueRender
}
```

**实现了topology.Output interface：**

```
type ElasticsearchOutput struct {
config map[interface{}]interface{}

action             string
index              value_render.ValueRender
index_type         value_render.ValueRender
id                 value_render.ValueRender
routing            value_render.ValueRender
source_field       value_render.ValueRender
bytes_source_field value_render.ValueRender
es_version         int
bulkProcessor      BulkProcessor

scheme   string
user     string
password string
hosts    []string
}
```

**实现了 topology.Filter：**

```
type GrokFilter struct {
config    map[interface{}]interface{}
overwrite bool
groks     []*Grok
target    string
src       string
vr        value_render.ValueRender
}
```

**实现了field_setter.FieldSetter：**

```
type MultiLevelFieldSetter struct {
preFields []string
lastField string
}
```

## 信号

### mainThreadExitChan

**加载配置失时，会执行** `mainThreadExitChan <- struct{}{}`退出

**当启动exitWhenNil选项时，beat会在没有event的时候调用** `box.mainThreadExitChan <- struct{}{}`退出
