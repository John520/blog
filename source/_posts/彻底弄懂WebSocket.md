---
title: 彻底弄懂WebSocket
date: 2022-04-04 22:37:04
tags:
 - 原创
 - WebSocket

categories: WebSocket
---

# 前言

在WebSocket出现之前，前端和后端交互通常使用Ajax进行HTTP API 通讯，然而若有实时性要求的项目，如聊天室或游戏中PVP对战或推送消息等场景，需要前端定时向后端轮询，然而轮询过快可能导致后端服务压力过大，轮询过慢可能导致实时性不高。WebSocket则为浏览器/客户端和服务器和服务端提供了双向通信的能力，保持了客户端和服务端的长连接，支持双向推送消息

# 什么是WebSocket 

WebSocket和HTTP一样属于OSI网络协议中的第七层，支持双向通信，底层连接采用TCP。WebSocket并不是全新的协议，使用时需要由HTTP升级，故它使用的端口是80（或443，有HTTPS升级而来），WebSocket Secure (wss)是WebSocket (ws)的加密版本，下图是WebSocket的建立和通信示意图。

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gycg9vxjx2j206x06dmxo.jpg)

需要特别注意的有

# 快速入门

（注：本文使用golang语言，不过原理都是想通的）

使用Go语言，通常有两种方式，一种是使用Go 语言内置的`net/http` 库编写`WebSocket`服务器，另一种是使用gorilla封装的Go语言的WebSocket语言官方库，Github地址：[**gorilla/websocket**](https://github.com/gorilla/websocket).(注，当然还有其他的Websocket封装库，目前gorilla这个比较常用)，gorilla库提供了一个聊天室的demo,本文将以这个例子上手入门.

## 1. 启动服务

gorilla/websocket 的聊天室的[README.md](https://github.com/gorilla/websocket/tree/master/examples/chat)

```go
$ go get github.com/gorilla/websocket
$ cd `go list -f '{{.Dir}}' github.com/gorilla/websocket/examples/chat`
$ go run *.go
```

## 2. 打开浏览器页面，输入`http://localhost:8080/ `

作为示例，我打开了两个页面，如下图所示

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gyecmutrbpj226c1fg4qp.jpg)

在左边输入`hello,myname is james`,两个窗口同时显示了这句话，F12打开调试窗口，在`network`那个tab下的ws 中左边的data有刚输入的上行和下行消息，而右边只有下行消息，说明消息确实确实通过左边的连接发送到服务端，并进行了广播给所有的客户端。

## 3. 如何建立连接

同样还是上述的F12调试窗口中的network tab 下的ws，打开请求头



<img src="http://tva1.sinaimg.cn/large/8dfd1ceegy1gyecwl7xxrj211o0oigwp.jpg" alt="image.png" style="zoom: 25%;" /><img src="http://tva1.sinaimg.cn/large/8dfd1ceegy1gyeczxc6ukj20we0powna.jpg" alt="image.png" style="zoom:25%;" />

**请求消息**：

```go
GET ws://localhost:8080/ws HTTP/1.1
Host: localhost:8080
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36
Upgrade: websocket
Origin: http://localhost:8080
Sec-WebSocket-Version: 13
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: Goland-cd273d2a=102d1f43-0418-4ea3-9959-2975794fdfe3
Sec-WebSocket-Key: 2e1HXejEZhjvYEEVOEE79g==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

1. 其中GET请求是`ws://`开头，区别于HTTP `/path`
2. `Upgrade: websocket`和`Connection: Upgrade`标识升级HTTP为WebSocket
3. `Sec-WebSocket-Key: 2e1HXejEZhjvYEEVOEE79g==`其中`2e1HXejEZhjvYEEVOEE79g==`为6个随机字节的base64用于标识一个连接，并非用于加密
4. `Sec-WebSocket-Version: 13`指定了WebSocket的协议版本

**应答消息**：

```go
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: WPaVPwi6nk4cFFxS8NJ3BIwAtNE=
```

`101`表示本次连接的HTTP协议即将被更改，更改后的协议就是`Upgrade: websocket`指定的WebSocket协议

其中`Sec-WebSocket-Accept: WPaVPwi6nk4cFFxS8NJ3BIwAtNE=`是服务器获取了Request Header 中的`Sec-WebSocket-Key`base64解码后，并拼接上`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`再用通过 SHA1 计算出摘要，再base64编码，并填写到`Sec-WebSocket-Accept`域.

所以`Sec-WebSocket-Key`和`Sec-WebSocket-Accept`的作用是主要是为了避免客户端不小心升级websocket，也即用来验证WebSocket的handshake，避免接受 non-WebSocket 的client（如HTTP客户端）.具体可参见[RFC6455](https://datatracker.ietf.org/doc/html/rfc6455)或[What is Sec-WebSocket-Key for?](https://stackoverflow.com/questions/18265128/what-is-sec-websocket-key-for)

## 4. gorilla/websocket代码

查看上面chat的demo代码是学习的好资料

```go
var addr = flag.String("addr", ":8080", "http service address")

func serveHome(w http.ResponseWriter, r *http.Request) {
	log.Println(r.URL)
	if r.URL.Path != "/" {
		http.Error(w, "Not found", http.StatusNotFound)
		return
	}
	if r.Method != "GET" {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}
	http.ServeFile(w, r, "home.html")
}

func main() {
	flag.Parse()
	hub := newHub()
	go hub.run()
	http.HandleFunc("/", serveHome)
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWs(hub, w, r)
	})
	err := http.ListenAndServe(*addr, nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```

其中`serveHome`主要返回聊天室的HTML资源，`serveWs`主要接受HTTP client 升级websocket请求，并处理聊天信息的广播（至整个房间的全部ws连接）

```html
<!DOCTYPE html>
<html lang="en">
<head>
<title>Chat Example</title>
<script type="text/javascript">
window.onload = function () {
    var conn;
    var msg = document.getElementById("msg");
    var log = document.getElementById("log");

    function appendLog(item) {
        var doScroll = log.scrollTop > log.scrollHeight - log.clientHeight - 1;
        log.appendChild(item);
        if (doScroll) {
            log.scrollTop = log.scrollHeight - log.clientHeight;
        }
    }

    document.getElementById("form").onsubmit = function () {
        if (!conn) {
            return false;
        }
        if (!msg.value) {
            return false;
        }
        conn.send(msg.value);
        msg.value = "";
        return false;
    };

    if (window["WebSocket"]) {
        conn = new WebSocket("ws://" + document.location.host + "/ws");
        conn.onclose = function (evt) {
            var item = document.createElement("div");
            item.innerHTML = "<b>Connection closed.</b>";
            appendLog(item);
        };
        conn.onmessage = function (evt) {
            var messages = evt.data.split('\n');
            for (var i = 0; i < messages.length; i++) {
                var item = document.createElement("div");
                item.innerText = messages[i];
                appendLog(item);
            }
        };
    } else {
        var item = document.createElement("div");
        item.innerHTML = "<b>Your browser does not support WebSockets.</b>";
        appendLog(item);
    }
};
</script>
<style type="text/css">
html {
    overflow: hidden;
}
...

</style>
</head>
<body>
<div id="log"></div>
<form id="form">
    <input type="submit" value="Send" />
    <input type="text" id="msg" size="64" autofocus />
</form>
</body>
</html>

```

以上是`home.html`这里主要是`conn.onclose`处理连接关闭`conn.onmessage`处理收到消息，`conn.send`处理发送消息，当然还有`conn.onopen`处理连接建立等，这里就不赘述了.

```go
// serveWs handles websocket requests from the peer.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}
	client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
	client.hub.register <- client

	// Allow collection of memory referenced by the caller by doing all work in
	// new goroutines.
	go client.writePump()
	go client.readPump()
}

```

`conn, err := upgrader.Upgrade(w, r, nil)`主要是升级HTTP到WebSocket,底层使用`http.Hijacker`来劫持底层的TCP 连接，后续就可以使用这个连接双端通信了.

```
// Hub maintains the set of active clients and broadcasts messages to the
// clients.
type Hub struct {
   // Registered clients.
   clients map[*Client]bool

   // Inbound messages from the clients.
   broadcast chan []byte

   // Register requests from the clients.
   register chan *Client

   // Unregister requests from clients.
   unregister chan *Client
}
```

`Hub`主要是维护这个房间的所有连接，当用个client 建立添加到`clients`这个map中，每次连接断开就会从`clients`这个map中移除

`go client.readPump()`负责将客户端发送的消息写到`broadcast`，而`go client.writePump()`负责将`broadcast`中的消息广播到`clients`记录的这个房间的全部client ,细节下文在讲完websocket 协议细节会继续来看这个源码。

# WebSocket协议

上面已经对ws进行了快速入门，那么WebSocket的通信格式是怎么样定义？这节就来介绍下

![image.png](http://tva1.sinaimg.cn/large/8dfd1ceegy1gyef3zfy5kj20te0f2q77.jpg)
