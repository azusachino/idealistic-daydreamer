---
title: Combine WebFlux with WebSocket
description: Yet another learning record
date: 2022-02-17
slug: spring-webflux-websocket
image: img/2022/02/Oymyakon.jpg
categories:
  - exp
tags:
  - java
  - spring
  - webflux
  - websocket
---

由于项目中需要用到 `WebSocket`，而之前一直没接触过，于是这是学习 `WebSocket` 相关使用方式的内容。【不涉及协议等底层知识点】

## 项目依赖

`spring-boot-starter-webflux` 中已经集成了 `WebSocket` 的依赖，其内容主要在 `spring-webflux` 包中。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

---

`WebFlux` 本身提供了对 `WebSocket` 协议的支持，处理 `WebSocket` 请求需要对应的 `handler` 实现 `WebSocketHandler` 接口，每一个 `WebSocket` 都有一个关联的 `WebSocketSession`，包含了建立请求时的握手信息 `HandshakeInfo`，以及其它相关的信息。可以通过 session 的 `receive()` 方法来接收客户端的数据，通过 session 的 `send()` 方法向客户端发送数据。

## WebSocketHandler

`WebSocketHandler` 是我们在 `WebFlux` 项目中处理 `WebSocket` 消息需要实现的主要接口，我们的 `WebSocket` 业务代码很大部分都将在 `WebSocketHandler#handle(WebSocketSession)` 方法中实现，方法签名如下：

```java
/**
 * A WebSocketHandler must compose the inbound and outbound streams into a unified flow and return a Mono<Void> that reflects the completion of that flow.
 */
public interface WebSocketHandler {

    /**
     * Return the list of sub-protocols supported by this handler.
     * <p>By default an empty list is returned.
     */
    default List<String> getSubProtocols() {
        return Collections.emptyList();
    }

    /**
     * Invoked when a new WebSocket connection is established, and allows
     * handling of the session.
     *
     * <p>See the class-level doc and the reference manual for more details and
     * examples of how to handle the session.
     * @param session the session to handle
     * @return indicates when application handling of the session is complete,
     * which should reflect the completion of the inbound message stream
     * (i.e. connection closing) and possibly the completion of the outbound
     * message stream and the writing of messages
     */
    Mono<Void> handle(WebSocketSession session);
}
```

## 最简单的 EchoWebSocketHandler

```java
public class EchoWebSocketHandler implements WebSocketHandler {
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // 在收到客户端消息时，给客户端发送 `Echo -> ${msg}`
        return session.send(session.receive().map(msg -> session.textMessage("Echo -> ".concat(msg.getPayloadAsText()))));
    }
}
```

## 注册到 HandlerMapping

写完 `WebSocketHandler` 之后，还要与指定的 `Route` 进行绑定，类似于 React-Router 的玩法。

**代码演示：**

```java
    @Bean
    public HandlerMapping webSocketMapping(EchoWebSocketHandler echoHandler) {
        final Map<String, WebSocketHandler> map = new HashMap<>(4);
        // 配置相应的路由
        map.put("/echo", echoHandler);

        final SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        // 将 WebSocketHandler处理设置为最优先级别
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE);
        mapping.setUrlMap(map);
        return mapping;
    }

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
```

### WebSocketHandlerAdapter

首先 `DispatchHandler` 是 `WebFlux` 中的分发器；然后我们根据 `WebSocketHandlerAdapter` 的类描述，知道了只要实例化该类，应用就会具有分发 `WebSocketHandler` 的能力，并且会通过 `SimpleUrlHandlerMapping` 进行路由；主要描述如下：

```java
/**
 * HandlerAdapter that allows org.springframework.web.reactive.DispatcherHandler
 * to support handlers of type WebSocketHandler with such handlers mapped to
 * URL patterns via org.springframework.web.reactive.handler.SimpleUrlHandlerMapping.
 */
public class WebSocketHandlerAdapter implements HandlerAdapter, Ordered {}
```

### SimpleUrlHandlerMapping

正如类名所描述的那样，`SimpleUrlHandlerMapping` 支持最简单的绑定方式，一个 `url` 对应一个 `WebSocketHandler`；稍微看了一点源码，主要的逻辑处理在 `AbstractUrlHandlerMapping#getHandlerInternal` 方法中。

## 分离数据的接收与发送操作

我们需要分别针对接收和发送操作写业务逻辑，最后通过 `Mono.zip` 进行合并；其中 `Mono<Void>` 用于表明处理是否结束。

```java
    @Override
    public Mono<Void> handle(WebSocketSession session) {

        // print all received msg
        Mono<Void> input = session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .map(msg -> id + ": " + msg)
                .doOnNext(System.out::println).then();

        // send `hello from server`
        Mono<Void> output = session.send(Mono.create(sink -> sink.success(session.textMessage("hello from server"))));

        /**
         * Mono.zip() 会将多个 Mono 合并为一个新的 Mono，
         * 任何一个 Mono 产生 error 或 complete 都会导致合并后的 Mono
         * 也随之产生 error 或 complete，此时其它的 Mono 则会被执行取消操作。
         */
        return Mono.zip(input, output).then();
    }
```

## 在 WebSocketHandler 外面发送消息

### FluxSink

`Flux#create` 方法是以编程方式创建 Flux 的高级形式，它允许每次产生多个数据，并且可以由多个线程产生。`Flux#create` 方法将内部的 FluxSink 暴露出来，FluxSink 提供了 next、error、complete 等方法。通过 `Flux#create` 方法，我们可以将响应式堆栈中的 API 与外侧进行连接。

### 封装

将 WebSocketSession 和 Flux 封装到一个类中，仅向外暴露数据发送接口。

```java
public class WebSocketSender {

    /**
     * WebSocket Session
     */
    private final WebSocketSession session;

    /**
     * Flux Publisher
     */
    private final FluxSink<WebSocketMessage> sink;

    public WebSocketSender(WebSocketSession session, FluxSink<WebSocketMessage> sink) {
        this.session = session;
        this.sink = sink;
    }

    /**
     * Send msg use current session
     */
    public void sendData(String data) {
        this.sink.next(this.session.textMessage(data));
    }
}
```

### 业务实现

```java
    /**
     * 利用SpringBean实现单例
     */
    @Resource
    private ConcurrentHashMap<String, WebSocketSender> sessionMap;

    /**
     * @param session WebSocket Session
     * @return None
     */
    @NonNull
    @Override
    public Mono<Void> handle(@NonNull WebSocketSession session) {
        // 从WS的URI中提取参数
        HandshakeInfo handshakeInfo = session.getHandshakeInfo();
        Map<String, String> queryMap = CommonUtils.getQueryMap(handshakeInfo.getUri().getQuery());
        // 每个客户端都带有id，作为Session的唯一标识
        String id = queryMap.getOrDefault("id", "defaultId");

        // 1. 给客户端消息返回响应
        Mono<Void> input = session.send(
                    session.receive()
                        .map(WebSocketMessage::getPayloadAsText)
                        .map(msg ->  session.textMessage("Echo -> " + msg)));
        // 2. 将 Session 和 FluxSink 保存到 Map 中
        Mono<Void> output = session.send(Flux.create(sink -> this.sessionMap.put(id, new WebSocketSender(session, sink))));

        // 3. Combine input with output
        return Mono.zip(input, output).then();
    }
```

### 外部使用

```java
    @Resource
    private ConcurrentHashMap<String, WebSocketSender> sessionMap;

    public Mono<String> testHandler(String id, String msg) {
        return Mono.create(sink -> {
            WebSocketSender sender = this.sessionMap.get(id);
            if (Objects.nonNull(sender)) {
                sender.sendData(finalMsg);
                sink.success(finalMsg);
            } else {
                sink.error(new RuntimeException("wrong id, no ws connection"));
            }
        });
    }
```

## 总结

我就是 `CTRL C/V` 的机器人。

## Reference

- [WebFlux 定点推送、全推送灵活 websocket 运用](https://cloud.tencent.com/developer/article/1480108)
