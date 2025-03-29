---
title: prometheus exporter的总结
date: 2022-5-26 17:38:00
tags:
- 原创
categories:
- 运维
---

## Go Application

例子来源于<https://prometheus.io/docs/guides/go-application/，用到了官方client：https://github.com/prometheus/client_golang>

prometheus有一个官方Go客户端库，可以用它来检测Go程序。下面这个例子中，会创建一个go应用，该应用将指标数据通过HTTP传送给prometheus。

```go
package main

import (
        "net/http"

        "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
        http.Handle("/metrics", promhttp.Handler())
        http.ListenAndServe(":2112", nil)
}
```

增加自定义指标myapp_processed_ops_total：

```go
package main

import (
        "net/http"
        "time"

        "github.com/prometheus/client_golang/prometheus"
        "github.com/prometheus/client_golang/prometheus/promauto"
        "github.com/prometheus/client_golang/prometheus/promhttp"
)

func recordMetrics() {
        go func() {
                for {
                        opsProcessed.Inc()
                        time.Sleep(2 * time.Second)
                }
        }()
}

var (
        opsProcessed = promauto.NewCounter(prometheus.CounterOpts{
                Name: "myapp_processed_ops_total",
                Help: "The total number of processed events",
               // ConstLabels: prometheus.Labels(map[string]string{"date": "Thursday"}),
        })
)

func main() {
        recordMetrics()

        http.Handle("/metrics", promhttp.Handler())
        http.ListenAndServe(":2112", nil)
}
```

## 我写的exporter

```go
package main

import (
 "fmt"
 "net/http"
)

func main() {
 http.HandleFunc("/metrics", HelloServer)
 http.ListenAndServe(":7777", nil)
}

var t = `# HELP myapp_processed_ops_total1 The total number of processed events
# TYPE myapp_processed_ops_total1 counter
myapp_processed_ops_total1{date="Thursday"} 1622`

func HelloServer(w http.ResponseWriter, r *http.Request) {
 fmt.Fprintf(w, t)
}
```

这个demo的metrics数据可以被prometheus成功获取。

似乎只要实现metrics，返回结构化的数据就可以了。

PS：返回数据

![WX20220526-173011@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h2lxrjsnwwj31iu0oadmi.jpg)

![WX20220526-172920@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h2lxqws4xxj31ko0q6dll.jpg)

## exporter规范

一个 Prometheus exporter 应该提供以下 API：

1. Metrics API：返回需要被暴露的指标(metrics)数据，格式可以是纯文本格式或者是 protobuf。
2. Health Check API：检查 exporter 的健康状态，应该返回 HTTP 200 表示正常，非 200 表示异常。
3. Metadata API（可选）：返回关于 exporter 的元数据，例如 exporter 版本、数据源等信息。

其中，Metrics API 是一个必须提供的 API，它应该以 HTTP 接口的形式暴露指标数据，供 Prometheus 采集器进行数据采集和存储。Health Check API 是一个可选的 API，用于检测 exporter 的健康状况，可以让用户知道 exporter 是否正常工作。Metadata API 也是一个可选的 API，提供关于 exporter 的元数据信息，便于用户了解 exporter 的基本情况。

**除了上述三个标准API外，Prometheus exporter并没有其他固定的API要求。但是，Exporter可以提供其他自定义的API，以便用户可以通过这些API获取更多的监控指标数据。**
