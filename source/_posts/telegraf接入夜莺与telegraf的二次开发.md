---
title: telegraf接入夜莺与telegraf的二次开发
date: 2023-7-19 11:00:00
categories:
- 运维
---

## 大纲

之前的夜莺监控系统一文中，提到了telegraf可以作为agent向夜莺传送数据，夜莺再将数据转存到数据源如Prometheus。夜莺监控系统一文中的部署部分没有涉及到agent的部署，只部署了夜莺和数据源部分，本文将把telegraf作为agent部署，完成agent->夜莺->数据源的全链路部署。

此外， telegrapf自身提供了对插件的二次开发的良好支持，包括input插件，output插件等等。本文将介绍二次开发可能比较多的input插件的二次开发规范与步骤的等。

## telegraf简介

telegraf是一个用Go实现的指标数据收集、处理、聚合和写入的采集工具。是夜莺官方比较推荐的agent。github地址为https://github.com/influxdata/telegraf

## 部署

笔者clone https://github.com/influxdata/telegraf项目后，执行build构建程序。

然后参考github上面的步骤，安装telegraf

```shell
# influxdata-archive_compat.key GPG fingerprint:
#     9D53 9D90 D332 8DC7 D6C8 D3B9 D8FF 8E1F 7DF8 B07E
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
sudo apt-get update && sudo apt-get install telegraf
```

执行：

```shell
telegraf config > telegraf.conf
```

生成一个telegraf配置文件。

修改生成的telegraf.conf的outputs.opentsdb这块，[参考](https://flashcat.cloud/docs/content/flashcat-monitor/nightingale-v6/agent/telegraf/)，改成：

```yml
[[outputs.opentsdb]]
  host = "http://127.0.0.1"
  port = 17000 # 夜莺默认监听的端口是 17000
  http_batch_size = 50
  http_path = "/opentsdb/put"
  debug = true
  separator = "_"
```

其实就是将输出改成夜莺的n9e-server服务的地址，这样telegraf可以将数据指标输出给夜莺。

然后对于夜莺，因为之前配置的是夜莺读取Prometheus的数据，没有实现夜莺将数据写入Prometheus中，下面修改夜莺的配置文件，实现让夜莺的数据写入Prometheus。修改夜莺的etc/config.yml文件，将Pushgw.Writers部分改成：

```shell
[[Pushgw.Writers]]
# Url = "http://127.0.0.1:8480/insert/0/prometheus/api/v1/write"
Url = "http://127.0.0.1:9090/api/v1/write"
```

这个是Prometheus对外暴露的API，夜莺指定该Url可以将数据输出到Prometheus中。

## 插件开发

telegraf包括以下四种插件：

Input插件：收集指标
Processor插件：转换与过滤指标数据
Aggregator插件：创建聚合指标，如创建平均值、最小值、最大值等聚合指标
Output插件：将指标写入各个输出目的地

指标数据的数据流和上面四个插件的顺序一致，指标数据依次经过收集、转换与过滤、聚合、输出这四个部分。本节将介绍Input插件的开发，其他三个插件的开发同理，可参考官网。

参考[官方文档](https://github.com/influxdata/telegraf/blob/master/docs/INPUTS.md)

在`plugins/inputs`下新建simple目录，该目录下存放3个文件：

```
README.md
sample.conf
simple.go
```

simple.go的内容为：

```go
//go:generate ../../../tools/readme_config_includer/generator
package simple

import (
	_ "embed"

	"github.com/influxdata/telegraf"
	"github.com/influxdata/telegraf/plugins/inputs"
)

//go:embed sample.conf
var sampleConfig string

type Simple struct {
	Ok  bool            `toml:"ok"`
	Log telegraf.Logger `toml:"-"`
}

func (*Simple) SampleConfig() string {
	return sampleConfig
}

// Init is for setup, and validating config.
func (s *Simple) Init() error {
	return nil
}

func (s *Simple) Gather(acc telegraf.Accumulator) error {
	if s.Ok {
		acc.AddFields("state", map[string]interface{}{"value": "pretty good"}, nil)
	} else {
		acc.AddFields("state", map[string]interface{}{"value": "not great"}, nil)
	}

	return nil
}

func init() {
	inputs.Add("simple", func() telegraf.Input { return &Simple{} })
}
```

sample.conf的内容为：

```conf
[[inputs.simple]]
  ok = true
```

README.md内容为空。

在telegraf.conf文件追加

```shell
[[inputs.simple]]
  ok = true
```

即表示开启了自定义的输出插件。

开发完毕后，如果需要测试结果，可以指定使用某个input插件，执行：

```shell
./telegraf --config telegraf.conf --input-filter simple # simple这是是自定义的输入插件名，可以换成自己开发的插件名
```

查看结果是否为自定的输出。

结果正常，如果想查看telegraf是否真的把数据上报给了夜莺，可修改telegraf.conf的outputs.opentsdb部分，将debug改成true，再启动telegraf可以在终端查看是否向夜莺发送了http请求了，请求结果如何。

```shell
[[outputs.opentsdb]]
  host = "http://127.0.0.1"
  port = 17000
  http_batch_size = 50
  http_path = "/opentsdb/put"
  debug = true # 注意
```

