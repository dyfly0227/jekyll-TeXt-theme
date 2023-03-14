---
title: 'gin开启websocket'
excerpt: '使用订阅者模式实现多客户端推送'
tags: 'go'
---

### 安装依赖
```bash
go get "github.com/gorilla/websocket"
```
### 代码实现
```go
package Socket

import (
	"encoding/json"
	"flag"
	"fmt"
	"github.com/gorilla/websocket"
	"log"
	"net/http"
)

// ConnClients socket客户
type ConnClients struct {
	conn *websocket.Conn // 连接对象
	send chan []byte // 发送消息通道
}
// ConnServer socket容器
type ConnServer struct {
	broadcast chan []byte // 广播消息通道
	register chan *ConnClients // 注册的客户
	unregister chan *ConnClients // 未注册的客户
	clients map[*ConnClients]*ConnClients // 客户集合
}
// 定义全局控制器
var connServer = &ConnServer{
	broadcast:  make(chan []byte),
	register:   make(chan *ConnClients),
	unregister: make(chan *ConnClients),
	clients: make(map[*ConnClients]*ConnClients),
}

// 升级配置
var socketUpgrade = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func websocketServer(w http.ResponseWriter, r *http.Request) {
	// 将http协议升级为websocket协议
	conn, err := socketUpgrade.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}
	// 新客户
	connClient := &ConnClients{
		conn: conn,
		send: make(chan []byte, 256),
	}
	// 加入注册列表
	connServer.register <- connClient

	// 开启单独线程用于收发消息
	go connClient.receiveMsg()
	go connClient.sendMsg()
}
// StartWebSocket 启动函数
func StartWebSocket() {
	var addr = flag.String("socketAddr", ":4433", "http service address")
	// 开启线程监听socket控制器
	go connServer.run()
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		websocketServer(w, r)
	})
	go http.ListenAndServe(*addr, nil)
	fmt.Println("socket server started")
}
type WebsocketData struct {
	Action string
}
func (c *ConnClients)receiveMsg()  {
	for {
		_, message, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("error: %v", err)
			}
			break
		}
		var msg WebsocketData
		err = json.Unmarshal(message, &msg)
		// 解析socket消息，如果是close，则注销当前客户，如果是心跳信息ping，则返回pong
		if msg.Action == "close" {
			connServer.unregister <- c
			break
		}
		if msg.Action == "ping" {
			j, _ := json.Marshal(&WebsocketData{Action: "pong"})
			c.send <- j
		}
	}
}
func (c *ConnClients)sendMsg()  {
	defer func() {
		c.conn.Close()
	}()
	for {
		select {
		// 循环遍历发送消息通道
		case message, ok := <-c.send:
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			c.conn.WriteMessage(websocket.TextMessage, message)
		}
	}
}

func (m *ConnServer) run()  {
	for {
		select {
		case client := <-m.register: // 新增客户
			m.clients[client] = client
		case client := <-m.unregister: // 注销
			if _, ok := m.clients[client]; ok {
				delete(m.clients, client)
				close(client.send)
			}
		case message := <-m.broadcast: // 广播消息到每个客户
			for client := range m.clients {
				client.send <- message
			}
		}
	}
}
// AddMsg 公用方法 推送消息给客户端
func AddMsg(t []byte)  {
	connServer.broadcast <- t
}
```

**思路**： 作为服务器，必然是一对多的关系，这里使用`订阅者模式`
