---
title: 'websocket封装'
excerpt: 'vue项目中对websocket封装，包含心跳机制'
tags: 'vue'
---

```javascript

import { socket_url } from "../config/common"; // socket的请求地址写在配置文件中

export default (onMessage: Function) => {
    let socketUrl = socket_url.replace("https", "ws").replace("http", "ws");
    let socket: WebSocket;
    let lockReconnect = false;
    let timer: ReturnType<typeof setTimeout>;
    const createSocket = () => {
        try {
            socket = new WebSocket(socketUrl);
            init();
        } catch (e) {
            console.log("catch" + e)
            reconnect()
        }
    }
    const reconnect = () => {
        if (lockReconnect) return;
        lockReconnect = true;
        clearTimeout(timer);
        timer = setTimeout(() => {
            createSocket();
        }, 4000);
    }

    const init = () => {
        socket.onopen = function (event) {
            console.log("WebSocket:已连接");
            //心跳检测重置
            heartCheck.reset().start();
        };

        //接收到消息的回调方法
        socket.onmessage = function (event) {
            console.log("WebSocket:收到一条消息", event.data);
            const isHeart = /pong/.test(event.data)
            if (onMessage && !isHeart) { // 触发自定义onMessage
                onMessage.call(null, event)
            }
            heartCheck.reset().start();
        };

        //连接发生错误的回调方法
        socket.onerror = function (event) {
            console.log("WebSocket:发生错误");
            reconnect();
        };

        //连接关闭的回调方法
        socket.onclose = function (event) {
            console.log("WebSocket:已关闭");
            heartCheck.reset();//心跳检测
            reconnect();
        };

        //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
        window.onbeforeunload = function () {
            socket.close();
        };
    }

    var heartCheck = {
        timeout: 5000,
        timeoutObj: setTimeout(() => { }),
        serverTimeoutObj: setInterval(() => { }),
        reset: function () {
            clearTimeout(this.timeoutObj);
            clearTimeout(this.serverTimeoutObj);
            return this;
        },
        start: function () {
            var self = this;
            clearTimeout(this.timeoutObj);
            clearTimeout(this.serverTimeoutObj);
            this.timeoutObj = setTimeout(function () {
                //这里发送一个心跳，后端收到后，返回一个心跳消息，
                //onmessage拿到返回的心跳就说明连接正常
                socket.send(JSON.stringify({
                    "action": "ping"
                }));
                console.log('ping');
                self.serverTimeoutObj = setTimeout(function () { // 如果超过一定时间还没重置，说明后端主动断开了
                    console.log('关闭服务');
                    socket.close();//如果onclose会执行reconnect，我们执行 websocket.close()就行了.如果直接执行 reconnect 会触发onclose导致重连两次
                }, self.timeout)
            }, this.timeout)
        }
    };
    createSocket();
}
```
### vue的全局注册
```javascript
// main.ts
import Websocket from "./utils/socket"
const onMessageList: Array<Function> = []; // 声明一个全局变量来收集onMessage列表，因为在不同的页面可能会有不同的处理
app.provide("onMessageList", onMessageList); // 全局注入
const onMessage = (event: any) => {
	// 遍历onMessage集合并触发
    onMessageList.forEach(f => {
        f.call(null, event);
    })
}
Websocket(onMessage); // 启动websocket
```
### 页面的自定义
```javascript
import { defineComponent, inject } from "vue";
const onMessageList = inject("onMessageList"); // 接收注入
const socketMessage = (res) => {
    const data = JSON.parse(res.data);
    // do something
};
onMessageList.push(socketMessage); // 将自定义处理事件插入onMessge集合
```