---
title: 基于go-zero和websocket的聊天室demo
date: 2022-10-28 10:00:00
tags:
- 原创
categories:
- golang
---
## 操作步骤

**使用go-zero的goctl工具，新建名为chat的api服务：**

> **ps：goctl生成api使用的handler模板使用默认的即可，如果没有修改过tpl文件，可以不用确认tpl文件是否为初始的默认值**

```
mkdir chat
cd chat
touch chat.api
```

**修改chat.api，修改为：**

```
type WsConnectRequest {
ID int `path:"id"`
}

type WsRequestItem {
Message string `json:"message"`
ToID    []int  `json:"to_id,omitempty"`
}

type WsResponseItem {
Message string `json:"message"`
FromID  int    `json:"from_id"`
Time    int64  `json:"time"`
}

service chat-api {
@handler WsHandler
get /:id
}
```

**根据新的api文件重新生成代码：**

```
goctl api go -api chat.api -dir .
go mod tidy
```

**此时chat目录内容如下：**

```
> tree chat                          
chat
├── chat.api
├── chat.go
├── etc
│   └── chat-api.yaml
└── internal
    ├── config
    │   └── config.go
    ├── handler
    │   ├── routes.go
    │   └── wshandler.go
    ├── logic
    │   └── wslogic.go
    ├── svc
    │   └── servicecontext.go
    └── types
        └── types.go

7 directories, 9 files
```

**在chat/internal/handler新建websocket.go文件，其内容如下：**

```
package handler

import (
"encoding/json"
"log"
"time"

"github.com/gorilla/websocket"
"github.com/zeromicro/go-zero/core/logx"

"github.com/nrbackback/zero-chatroom/chat/internal/types"
)

type Hub struct {
clients map[int]*Client

broadcast chan Msg
}

type Msg struct {
Msg    string
FromID int
ToIDs  []int
Time   int64
}

type Client struct {
ID   int
conn *websocket.Conn
send chan Msg
}

var h *Hub

func InitHub() {
h = &Hub{
broadcast: make(chan Msg),
clients:   make(map[int]*Client),
}
}

func RunHub() {
for {
select {
case message := <-h.broadcast:
for _, client := range h.clients {
select {
case client.send <- message:
default:
close(client.send)
delete(h.clients, client.ID)
}
}
}
}
}

func register(c *Client) {
h.clients[c.ID] = c
}

func unregister(c *Client) {
delete(h.clients, c.ID)
close(c.send)
}

var maxMessageSize = int64(512)
var pongWait = 60 * time.Second

func (c *Client) readPump() {
logx.Errorw("test close writer error", logx.Field("error", "fdfsdfds"))
defer func() {
unregister(c)
c.conn.Close()
}()
c.conn.SetReadLimit(maxMessageSize)
c.conn.SetReadDeadline(time.Now().Add(pongWait))
c.conn.SetPongHandler(func(string) error {
c.conn.SetReadDeadline(time.Now().Add(pongWait))
return nil
})

for {
_, message, err := c.conn.ReadMessage()
if err != nil {
if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
log.Printf("error: %v", err)
}
break
}
var req types.WsRequestItem
if err := json.Unmarshal(message, &req); err != nil {
logx.Errorw("unmarshal message when read error", logx.Field("message", string(message)), logx.Field("error", err))
continue
}
h.broadcast <- Msg{
Msg:    req.Message,
FromID: c.ID,
ToIDs:  req.ToID,
Time:   time.Now().Unix(),
}
}
}

var pingPeriod = (pongWait * 9) / 10
var writeWait = 10 * time.Second

func (c *Client) writePump() {
ticker := time.NewTicker(pingPeriod)
defer func() {
ticker.Stop()
c.conn.Close()
}()
for {
select {
case message, ok := <-c.send:
c.conn.SetWriteDeadline(time.Now().Add(writeWait))
if !ok {
c.conn.WriteMessage(websocket.CloseMessage, []byte{})
return
}
w, err := c.conn.NextWriter(websocket.TextMessage)
if err != nil {
logx.Errorw("NextWriterd error", logx.Field("error", err))
continue
}
resp := types.WsResponseItem{
Message: message.Msg,
FromID:  message.FromID,
Time:    message.Time,
}
v, _ := json.Marshal(resp)
w.Write(v)
if err := w.Close(); err != nil {
logx.Errorw("close writer error", logx.Field("error", err))
continue
}
case <-ticker.C:
c.conn.SetWriteDeadline(time.Now().Add(writeWait))
if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
logx.Errorw("write ping message error", logx.Field("error", err))
continue
}
}
}
}
```

**修改chat/internal/handler/wshandler.go，更改为如下内容：**

```
package handler

import (
"log"
"net/http"

"github.com/gorilla/websocket"
"github.com/zeromicro/go-zero/rest/httpx"

"github.com/nrbackback/zero-chatroom/chat/internal/svc"
"github.com/nrbackback/zero-chatroom/chat/internal/types"
)

var upgrader = websocket.Upgrader{
ReadBufferSize:  1024,
WriteBufferSize: 1024,
}

func WsHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
return func(w http.ResponseWriter, r *http.Request) {
var req types.WsConnectRequest
if err := httpx.Parse(r, &req); err != nil {
httpx.Error(w, err)
return
}
conn, err := upgrader.Upgrade(w, r, nil)
if err != nil {
log.Println(err)
return
}
if err := conn.WriteMessage(1, []byte("connected")); err != nil {
return
}
c := &Client{
ID:   req.ID,
conn: conn,
send: make(chan Msg),
}
register(c)

go c.writePump()
go c.readPump()
}
}
```

**修改chat/chat.go，更改为如下内容：**

```
package main

import (
"flag"
"fmt"

"github.com/zeromicro/go-zero/core/conf"
"github.com/zeromicro/go-zero/rest"

"github.com/nrbackback/zero-chatroom/chat/internal/config"
"github.com/nrbackback/zero-chatroom/chat/internal/handler"
"github.com/nrbackback/zero-chatroom/chat/internal/svc"
)

var configFile = flag.String("f", "etc/chat-api.yaml", "the config file")

func main() {
flag.Parse()

var c config.Config
conf.MustLoad(*configFile, &c)

ctx := svc.NewServiceContext(c)
server := rest.MustNewServer(c.RestConf)
defer server.Stop()

handler.InitHub()
go handler.RunHub()

handler.RegisterHandlers(server, ctx)

fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
server.Start()
}
```

## 测试

![WX20221029-151628@2x](https://tva4.sinaimg.cn/mw690/006gLprLgy1h7m6kq0cg2j31l61267b6.jpg)

![WX20221029-151713@2x](https://tvax3.sinaimg.cn/mw690/006gLprLgy1h7m6l7tqe7j31ky0zagro.jpg)

## 项目

github项目地址 https://github.com/nrbackback/zero-chatroom
