---
layout: post
title: Websocket 技术介绍及应用
tags: ["websock", "springboot"]
---

Web 开发中最常用的协议当属 HTTP，除此之外还有非常多丰富的网络协议来解决各类问题。本文中给大家介绍一种流行的协议 **Websocket** 以及它在一些典型场景的应用。

# 为什么要有 WebSocket？

我们知道 HTTP 协议是基于 TCP 的应用层协议，它是单向通信的，由客户端（一般是浏览器）发起请求，服务端（webserver）处理请求后发送响应。我们用 HTTP 进行通信用对话式是这样的：

```bash
2019-07-05 19:20:01 CLIENT: 在吗？
2019-07-05 19:20:03 SERVER: 在的
2019-07-05 19:20:04 CLIENT: 吃了吗?
2019-07-05 19:20:05 SERVER: 你猜 🌝
```

整个流程是一问一答，在传输过程中是这样的：

<center>
    <img src="https://i.loli.net/2019/07/05/5d1f60cce28be92079.png" width="350"/>
</center>
<br/>

这种 _请求<-->响应_ 的方式也称为短连接，在部分场景下存在一些问题：

1. 服务端状态更新无法直接通知客户端（如消息推送）
2. 在高频低延迟的场景中无法满足（如股票 K 线图）

一般简单粗暴的解决办法是客户端定期请求服务端，达到伪实时的效果，这种方式也就是轮训。它的缺点是大部分请求都是无用的，非常浪费服务器资源和带宽，在小型应用中较适用。

除了轮训的方式也有其他方案，比如长连接：在页面内嵌一个 iframe 设置 src 属性为长连接或 xhr 请求，这样消息就可以即时到达，但是弊端也很明显：服务器需要维护很多长连接，开销不少。

下面我们来了解一下 websocket。

# 什么是 Websocket？

在维基百科里是这么介绍 websocket 的：

> WebSocket是一种通信协议，可在单个 TCP 连接上进行全双工通信。WebSocket 协议在2011年由IETF标准化为RFC 6455，后由RFC 7936补充规范。Web IDL中的WebSocket API由W3C标准化。
> 
> WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。

websocket 支持全双工通信：允许服务器和客户端随时推送消息，是真正的服务器端推送。

它有以下特点：

- 双向协议 - 客户端/服务器可以向另一方发送消息
- 全双工通信 - 客户端和服务器可以同时独立地相互通信
- 单个 TCP 连接 - 在升级 HTTP 连接后，客户端和服务器在 Websocket 连接的整个生命周期内通过同一个 TCP 连接进行通信
- 开销较少 - 连接创建后双方进行数据交换数据包头较小
- 没有同源限制 - 客户端可以和任意服务器通信
- 协议标识为 ws，加密的标识为 wss

它的通信式对话是这样的：

```bash
2019-07-05 19:20:01 CLIENT: 这个需求做好了吗？
2019-07-05 19:20:03 SERVER: 等我先测试一下
2019-07-05 19:20:05 CLIENT: 好的，你先忙
2019-07-05 20:20:05 SERVER: 测试环境通过
2019-07-05 20:40:05 SERVER: 正在发布生产
2019-07-05 20:45:05 SERVER: 生产发布成功
2019-07-05 20:46:00 CLIENT: 好的，我在验收了
```

你可以发现客户端可以给服务器发送消息，服务器也会发送消息到客户端。下面是 websocket 的通信流程：

<center>
    <img src="https://i.loli.net/2019/07/05/5d1f73c22c10664209.png" width="450"/>
    <i>Websocket连接（图片来自 <a href="https://www.pubnub.com/blog/2015-01-05-websockets-vs-rest-api-understanding-the-difference/">PubNub.com</a> ）</i>
</center>
<br/>

## 通信过程

### 握手过程（Handshake）

Websocket 的握手是复用了 HTTP 的，所以很多请求头都是类似的，不过会做一次协议升级，请求的数据报结构是这样：

```bash
GET / HTTP/1.1
Host: 127.0.0.1:8080
Origin: http://127.0.0.1:8080
Connection: keep-alive, Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: gi4r4pxnRiVVv991VRCJ/g==
```

这里需要注意的是这几个 header 的含义：

- `Connection`: keep-alive 是 HTTP/1.1 中保持连接的，Upgrade 表示要升级协议
- `Upgrade`: websocket 表示要升级到 websocket 协议
- `Sec-WebSocket-Version`: 13 表示 websocket 的版本
- `Sec-WebSocket-Key`: 主要用于和服务器通信时保持会话

当 ws 客户端发起请求后，服务端会响应如下报文：

```bash
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Date: Sat, 05 Jul 2019 19:30:01 GMT
Sec-WebSocket-Accept: dIQIfGgvv50y9o2IHFQIH8cuS4M=
Upgrade: websocket
```

- `HTTP 101`: 表示服务器收到了切换协议请求，同意切换协议
- `Sec-WebSocket-Accept`: 根据客户端发送的 Sec-WebSocket-Key 计算而来

Sec-WebSocket-Accept 计算公式：

1. 将 Sec-WebSocket-Key 和 258EAFA5-E914-47DA-95CA-C5AB0DC85B11 拼接
2. 对拼接结果做 SHA1 计算，然后转为 Base64 字符串。

### 数据帧格式

ws 协议中通信的格式是数据帧(frame)，一次消息可以由一个或多个帧组成。

```bash
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
```

帧的格式是上面所述，每一项代表的意思本文不在详细介绍，在参考资料里可以看。

## STOMP

Websocket 是一套完整的应用层协议，有标准的 API。它定义了两种传输信息的类型：二进制和文本信息。如果你直接使用 websocket 编程就类似与用 TCP 套接字编写 web 应用，是比较麻烦的。

STOMP（Simple (or Streaming) Text Orientated Messaging Protocol）是一种简单的文本消息协议，它允许 STOMP 客户端和任意的 STOMP 消息代理（Broker）进行交互。STOMP 协议可以建立在 Websocket 之上，也可以建立在其他应用层协议之上，它是 websocket 的上层协议。当然 STOMP 也可以应用在 RabbitMQ、ActiveMQ 等中间件中。

它的格式为：

```bash
SEND
destination: /app/hello
content-length: 17

{\"name\":\"你好!\"}
```

- SEND：STOMP 命令，表明会发送一些内容
- destination：头信息，用来表示消息发送到哪里，类似于 MQ 里的 topic
- content-length：头信息，用来表示负载内容的长度
- 空行：协议规定，和 http 相同
- 帧内容（负载）内容

# 案例分享

下面我们通过几个例子来分享一下它的应用场景

## 集成 Spring Boot

Spring Boot 框架对 websocket 的支持比较好，我们可以加入相关依赖：

```xml
<!-- 服务端 websocket 支持 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

编写服务端代码

```java
@Configuration
// 表示开启使用STOMP协议来传输基于代理的消息，Broker就是代理的意思
@EnableWebSocketMessageBroker 
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	
    // 配置消息代理
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 配置服务端推送消息的前缀
        config.enableSimpleBroker("/topic");
        // 配置客户端订阅消息的前缀
        config.setApplicationDestinationPrefixes("/app");
    }

    // 注册STOMP协议的节点，并指定映射的URL
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // 注册STOMP协议节点，同时指定使用SockJS协议。
        registry.addEndpoint("/websocket-example")
            .setAllowedOrigins("*") //解决跨域问题
            .withSockJS();
    }

}
```

```java
@MessageMapping("/hello")
@SendTo("/topic/hello")
public Greeting greeting(HelloMessage message) throws Exception {
    log.info("收到消息: {}", message);
    Thread.sleep(1000); // simulated delay
    return new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!");
}
```

编写客户端代码

```js
var socket = new SockJS('/websocket-example');
stompClient = Stomp.over(socket);
stompClient.connect({}, function (frame) {
    setConnected(true);
    console.log('Connected: ' + frame);
    stompClient.subscribe('/topic/hello', function (data) {
        console.log(JSON.parse(data.body).content);
    });
});
```

上面的代码是简化版的，客户端步骤：

1. 开启 SockJS: websocket 协议的实现，增加了对浏览器不支持 websocket 的时候的兼容支持
2. 使用 STOMP: 消息统一使用 STOMP 协议格式进行交互

我将完整的代码放在 [Github]() 上，有 3 个示例你可以参考：

1. [聊天应用]()
2. [消息推送]()
3. [订单状态回调]()

# 常见问题

## 跨域问题

## 生产部署


# 总结

## 参考资料

- [WebSocket协议：5分钟从入门到精通](https://www.cnblogs.com/chyingp/p/websocket-deep-in.html)
- [rfc6455](https://tools.ietf.org/html/rfc6455)
- [刨根问底HTTP和WebSocket协议](https://www.jianshu.com/p/0e5b946880b4)