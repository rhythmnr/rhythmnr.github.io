---
title: weboscket使用
date: 2022-11-04 10:19:00
tags:
- 原创
categories:
- 网络
---
**Golang 官方标准库实现的 websocket 在功能上有些欠缺，所以可以用**[https://pkg.go.dev/github.com/gorilla/websocket](https://pkg.go.dev/github.com/gorilla/websocket)这个包，这个包提供了完整的针对webosocket协议的实现，遵循了[RFC 6455](https://rfc-editor.org/rfc/rfc6455.html)。

## postman发送websocket请求

**我使用的Postman的版本是10.0.32，点击左侧的New，选择Websocket Request**

![WX20221023-161950@2x](https://tvax3.sinaimg.cn/mw690/006gLprLgy1h7fap98ya6j31sy0l8tga.jpg)

## 包分析

### 接收到请求时，响应

**一个可用的demo**

```
func main() {
var upgrader = websocket.Upgrader{
ReadBufferSize:   1024,
WriteBufferSize:  1024,
HandshakeTimeout: 5 * time.Second,
}
http.HandleFunc("/websocket", func(w http.ResponseWriter, r *http.Request) {
conn, _ := upgrader.Upgrade(w, r, nil)
for {
msgType, msg, err := conn.ReadMessage()
if err != nil {
return
}
fmt.Printf("%s receive: %s\n", conn.RemoteAddr(), string(msg))
if err = conn.WriteMessage(msgType, msg); err != nil {
return
}
if err = conn.WriteMessage(msgType, []byte("..........")); err != nil {
return
}
}
})
http.ListenAndServe(":22345", nil)
time.Sleep(30 * time.Minute)
}
```

**postman对应的请求效果：**

![WX20221023-163149@2x](https://tva1.sinaimg.cn/mw690/006gLprLgy1h7fb12radjj31kw0u4q7e.jpg)

**在http.Handle里，调用** `conn, _ := upgrader.Upgrade(w, r, nil)`，将http Request升级为一个websocket包的Conn对象。

**调用ReadMessage读出消息，WriteMessage写消息。**

**也可以改造成使用io.Reader，主要是使用conn.NextReader()**

```
func main() {
var upgrader = websocket.Upgrader{
ReadBufferSize:   1024,
WriteBufferSize:  1024,
HandshakeTimeout: 5 * time.Second,
}

http.HandleFunc("/websocket", func(w http.ResponseWriter, r *http.Request) {
conn, _ := upgrader.Upgrade(w, r, nil)

for {
messageType, r, err := conn.NextReader()
if err != nil {
return
}
w, err := conn.NextWriter(messageType)
if err != nil {
fmt.Println("......err....", err)
}
if _, err := io.Copy(w, r); err != nil {
fmt.Println("......err..1..", err)
}
if err := w.Close(); err != nil {
fmt.Println("......err..2..", err)
}
}
})

http.ListenAndServe(":22345", nil)
time.Sleep(30 * time.Minute)
}
```

**这是运行结果：**

![WX20221023-164846@2x](https://tva2.sinaimg.cn/mw690/006gLprLgy1h7fbilt65dj31ke0q4tc7.jpg)

### 服务端主动推送消息

```
package module

import (
"log"
"net/http"
"sync"
"time"

"github.com/gin-gonic/gin"
"github.com/gorilla/websocket"
)

var (
// 消息通道
news = make(map[string]chan interface{})
// websocket客户端链接池
client = make(map[string]*websocket.Conn)
// 互斥锁，防止程序对统一资源同时进行读写
mux sync.Mutex
)

// api:/getPushNews接口处理函数
func GetPushNews(context *gin.Context) {
id := context.Query("userId")
log.Println(id + "websocket链接")
// 升级为websocket长链接
WsHandler(context.Writer, context.Request, id)
}

// api:/deleteClient接口处理函数
func DeleteClient(context *gin.Context) {
id := context.Param("id")
// 关闭websocket链接
conn, exist := getClient(id)
if exist {
conn.Close()
deleteClient(id)
} else {
context.JSON(http.StatusOK, gin.H{
"mesg": "未找到该客户端",
})
}
// 关闭其消息通道
_, exist = getNewsChannel(id)
if exist {
deleteNewsChannel(id)
}
}

// websocket Upgrader
var wsupgrader = websocket.Upgrader{
ReadBufferSize:   1024,
WriteBufferSize:  1024,
HandshakeTimeout: 5 * time.Second,
// 取消ws跨域校验
CheckOrigin: func(r *http.Request) bool {
return true
},
}

// WsHandler 处理ws请求
func WsHandler(w http.ResponseWriter, r *http.Request, id string) {
var conn *websocket.Conn
var err error
var exist bool
// 创建一个定时器用于服务端心跳
pingTicker := time.NewTicker(time.Second * 10)
conn, err = wsupgrader.Upgrade(w, r, nil)
if err != nil {
log.Println(err)
return
}
// 把与客户端的链接添加到客户端链接池中
addClient(id, conn)

// 获取该客户端的消息通道
m, exist := getNewsChannel(id)
if !exist {
m = make(chan interface{})
addNewsChannel(id, m)
}

// 设置客户端关闭ws链接回调函数
conn.SetCloseHandler(func(code int, text string) error {
deleteClient(id)
log.Println(code)
return nil
})

for {
select {
case content, _ := <-m:
// 从消息通道接收消息，然后推送给前端
err = conn.WriteJSON(content)
if err != nil {
log.Println(err)
conn.Close()
deleteClient(id)
return
}
case <-pingTicker.C:
// 服务端心跳:每20秒ping一次客户端，查看其是否在线
conn.SetWriteDeadline(time.Now().Add(time.Second * 20))
err = conn.WriteMessage(websocket.PingMessage, []byte{})
if err != nil {
log.Println("send ping err:", err)
conn.Close()
deleteClient(id)
return
}
}
}
}

// 将客户端添加到客户端链接池
func addClient(id string, conn *websocket.Conn) {
mux.Lock()
client[id] = conn
mux.Unlock()
}

// 获取指定客户端链接
func getClient(id string) (conn *websocket.Conn, exist bool) {
mux.Lock()
conn, exist = client[id]
mux.Unlock()
return
}

// 删除客户端链接
func deleteClient(id string) {
mux.Lock()
delete(client, id)
log.Println(id + "websocket退出")
mux.Unlock()
}

// 添加用户消息通道
func addNewsChannel(id string, m chan interface{}) {
mux.Lock()
news[id] = m
mux.Unlock()
}

// 获取指定用户消息通道
func getNewsChannel(id string) (m chan interface{}, exist bool) {
mux.Lock()
m, exist = news[id]
mux.Unlock()
return
}

// 删除指定消息通道
func deleteNewsChannel(id string) {
mux.Lock()
if m, ok := news[id]; ok {
close(m)
delete(news, id)
}
mux.Unlock()
}
```

### 控制信息

**本包定义了3中控制信息：ping pong close，调用WriteControl, WriteMessage 或者 NextWriter可以发送控制信息**

### 请求后每隔一段时间给客户端推送消息

```
func main() {
var upgrader = websocket.Upgrader{
ReadBufferSize:   1024,
WriteBufferSize:  1024,
HandshakeTimeout: 5 * time.Second,
}
http.HandleFunc("/websocket", func(w http.ResponseWriter, r *http.Request) {
conn, _ := upgrader.Upgrade(w, r, nil)
for {
time.Sleep(time.Second * 2)
if err := conn.WriteMessage(1, []byte("..........")); err != nil {
return
}
}
})
http.ListenAndServe(":22345", nil)
time.Sleep(30 * time.Minute)
}
```

**一旦点击了Connect，服务端就会每隔2秒给客户端发..........**
