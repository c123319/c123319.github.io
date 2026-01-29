---
title: Spring集成websocket(1)
date: 2024-12-26 09:24:56
author: chengjt
summary:
img:
cover:
coverImg:
tags:
    - spring
    - WebSocket
categories:
    - spring
---
# spring集成websocket方法列举：
1. 基于原生 WebSocket（通过 @ServerEndpoint 注解或 Spring 的 
WebSocketHandler
2. 基于 Spring WebSocket（基于 STOMP 协议）

## 基于原生 WebSocket 通过 @ServerEndpoint 注解 (javaEE)
```
    //把@ServerEndpoint 修饰的类注册到spring容器，以实现依赖注入
    //ServerEndpointExporter：这个 Bean 的作用是让 Spring 容器自动扫描带有 
    //@ServerEndpoint 注解的 WebSocket 端点类，并将它们注册到 WebSocket 服务中。通过
    //这个 Bean，Spring 可以支持 Java EE 规范中的 @ServerEndpoint 功能。
@Bean
public ServerEndpointExporter serverEndpointExporter() {
    return new ServerEndpointExporter();
}
```
@ServerEndpoint：这是 Java EE (或 Jakarta EE) 中的标准注解，用于标注一个类作为 WebSocket 端点。它允许你将 WebSocket 的 URL 映射到 Java 类的方法，从而简化了 WebSocket 的配置和实现。

这个方式不需要实现 WebSocketConfigurer 或 WebSocketHandler，而是通过使用 @ServerEndpoint 注解来直接标识一个类作为 WebSocket 端点。

WebSocket 端点的生命周期（如连接的建立、消息的发送等）由 WebSocket 容器（例如 Tomcat、Jetty）负责。

```

@ServerEndpoint("/ws")
public class MyWebSocketEndpoint {

    @OnOpen
    public void onOpen(Session session) {
        System.out.println("WebSocket opened: " + session.getId());
    }

    @OnMessage
    public String onMessage(String message, Session session) {
        return "Received: " + message;
    }

    @OnClose
    public void onClose(Session session) {
        System.out.println("WebSocket closed: " + session.getId());
    }
}


```

## 基于原生 WebSocket 通过 Spring 的 WebSocketHandler (Spring)

```
@EnableWebSocket  // 开启WebSocket的自动配置
@Configuration    // 代表当前是一个配置类
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyWebSocketHandler(), "/ws").setAllowedOrigins("*");
    }
}
```

## 基于 Spring WebSocket（基于 STOMP 协议）
Spring WebSocket (基于 STOMP 协议)：适用于需要复杂消息处理的场景，支持消息订阅、广播等功能，通常使用 @MessageMapping 和消息代理。

选择哪种方式取决于应用的需求。如果你需要订阅/发布、广播等功能，可以使用 Spring WebSocket



## 总结
在 Spring 中集成 WebSocket 主要有两种方式：

Spring WebSocket (基于 STOMP 协议)：适用于需要复杂消息处理的场景，支持消息订阅、广播等功能，通常使用 @MessageMapping 和消息代理。

原生 WebSocket (不基于 STOMP)：适用于简单的 WebSocket 通信场景，直接处理 WebSocket 消息，灵活轻量。

选择哪种方式取决于应用的需求。如果你需要订阅/发布、广播等功能，可以使用 Spring WebSocket；如果只是简单的双向通信，直接使用原生 WebSocket 可能会更简单。
<br/>

# 下面描述的是第一种方法的WebSocketHandler这种方式

## 1. 引入依赖

### spring parent

```bash
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.10.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```

### websocket 引入的依赖

```bash
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
 </dependency>
         <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
 </dependency>
```

## 2.websocket配置类

```
import com.github.chengjt.websocket.handlers.SimpleHandshakeHandler;
import com.github.chengjt.websocket.handlers.SimpleWebSocketMessageHandler;
import com.github.chengjt.websocket.interceptors.HttpWebSocketInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@EnableWebSocket    // 开启WebSocket的自动配置
@Configuration      //  代表当前是一个配置类
@RequiredArgsConstructor //用于依赖注入
public class WebSocketConfig implements WebSocketConfigurer {
    //自己实现的websocket 拦截器
    @Autowired
    HttpWebSocketInterceptor httpWebSocketInterceptor; 
    //自己实现的 websocket 握手处理器 
    @Autowired
    SimpleHandshakeHandler simpleHandshakeHandler;
    //实现的 WebSocket处理程序，个人理解类似controller里面的接口一样
    @Autowired
    SimpleWebSocketMessageHandler simpleWebSocketMessageHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(simpleWebSocketMessageHandler, "/websocket/**")  // 注册WebSocket处理程序
                .setAllowedOrigins("*")             // 设置允许跨域访问
                .addInterceptors(httpWebSocketInterceptor);   // 添加WebSocket握手拦截器(后面需要实现)
// 设置WebSocket握手处理程序（后面需要实现）,用默认的就行了，实现复杂没搞定
//                .setHandshakeHandler(simpleHandshakeHandler);
    }
}
```
#### WebSocketHandlerRegistry：
&emsp;&emsp;它是Spring Framework中用于注册并管理WebSocket处理程序的类。它是WebSocketConfigurer接口的一部分，
        它定义了一个方法registerWebSocketHandlers(WebSocketHandlerRegistry registry)，
        该方法用于注册和配置WebSocket处理程序实例。<br/>
        WebSocketHandlerRegistry提供了一些方法，可以用于注册WebSocket处理程序、设置跨域访问规则，设置拦截器等。其主要方法如下：<br/>
&emsp;&emsp;①：addHandler(WebSocketHandler handler, String... paths):<br/>
        &emsp;&emsp;注册WebSocket处理程序，并指定处理程序可访问的路径；<br/>
&emsp;&emsp;②：setAllowedOrigins(String... origins):<br/>
        &emsp;&emsp;设置允许跨域访问的域名列表；<br/>
&emsp;&emsp;③：addInterceptors(HandshakeInterceptor... interceptors):<br/>
        &emsp;&emsp;添加WebSocket握手拦截器；<br/>
&emsp;&emsp;④：setHandshakeHandler(HandshakeHandler handshakeHandler):<br/>
        &emsp;&emsp;设置WebSocket握手处理程序；<br/>
&emsp;&emsp;⑤：setTaskScheduler(TaskScheduler taskScheduler):<br/>
        设置任务调度程序。(注：需要SpringBoot 2.7.11版本以上)<br/>
注：这段文字来自于大佬的参考文章

## 3.websocket拦截器
这一段是大佬文章里面的实现
``` bash
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor;
import java.util.Map;

/**
 * WebSocket握手拦截器，检查握手请求和响应，对WebSocketHandler传递属性，用于区别WebSocket
 *
 * @author Anhui OuYang
 * @version 1.0
 **/
@Slf4j
@Component
public class HttpWebSocketInterceptor extends HttpSessionHandshakeInterceptor {

    /***
     * 握手之前被调用
     * @param request 请求信息
     * @param response 响应信息
     * @param wsHandler 用于处理WebSocket通信过程中的各种事件和消息
     * @param attributes 如果需要，可以使用setAttribute方法添加属性，这些属性可以在后续处理中使用
     * @return 返回true表示继续握手，或者返回false以终止握手
     * @throws Exception 异常信息
     */
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        log.info("请求被拦截器拦截，当前请求地址为：{}", request.getURI().toString());
        //在拦截器解析信息，并设置到 attributes 中，后续程序可以在这里面取数据
        // 示例请求地址：http://localhost:8080/websocket/tom  因为这个restFul风格的地址，所以那个tom我需要拿到
        String path = request.getURI().getPath();
        String name = path.substring(path.lastIndexOf("/") + 1);
        attributes.put("loginName", name);
        log.info("请求握手成功，当前的登录人为：{}", attributes.get("loginName"));
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    /***
     * 握手成功之后或者失败之后被调用；可以利用这个方法去清理任何未完成的状态并记录异常。
     */
    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception ex) {
        log.info("请求被拦截器拦截放行。。。");
        super.afterHandshake(request, response, wsHandler, ex);
    }
}
```
个人理解，这个拦截器和普通http的拦截器一样用，在这里实现类似登录token的校验逻辑<br>
### 以下是大佬文章摘抄：<br>
HandshakeInterceptor接口分别有两个实现类：<br>
HttpSessionHandshakeInterceptor：是基于HTTP会话的WebSocket握手拦截器。<br>
在握手之前，它会基于当前HTTP请求的会话信息来添加 WebSocket 握手的请求头；<br>
在握手之后，它会通过检查握手请求头来确定是否要创建或关闭会话以及报告任何错误。<br>
WebSocketHandshakeInterceptor：是基于WebSocket协议的握手拦截器。<br>
在握手之前，它会检查WebSocket握手请求和响应头，并在需要时添加或删除必要的消息头；<br>
在握手之后，它会通过检查握手请求和响应标头来确保协议交换已成功，如果不成功，它会关闭WebSocket连接并报告任何错误。<br>

## 4.websocket握手处理器
这个东西用默认的就行，不要自己瞎搞，要不然websocket连接会连不上。里面涉及到websocket的升级等，逻辑挺多，目前没明白<br/>
spring默认实现这部分涉及到两个类：AbstractHandshakeHandler 和 DefaultHandshakeHandler ，可以自己看看这里面的逻辑。
``` bash
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.server.HandshakeHandler;
import org.springframework.web.socket.server.support.AbstractHandshakeHandler;

@Component
@Slf4j
public class SimpleHandshakeHandler extends AbstractHandshakeHandler implements HandshakeHandler {
 //AbstractHandshakeHandler 这个类是最重要的，核心之一
}
```
## 5.websocket握手处理器
这个就是你自己写业务逻辑的地方，重写对应的方法。<br>
有两个可以继承的类：TextWebSocketHandler 和 BinaryWebSocketHandler，<br>
TextWebSocketHandler 处理文本消息 ，BinaryWebSocketHandler处理二进制消息，<br>
一般我用TextWebSocketHandler足够。<br>

以下是代码的参考：
```bash
import com.alibaba.fastjson2.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;


/**
 * WebSocket消息处理
 */

@Slf4j
@Component
public class SimpleWebSocketMessageHandler extends TextWebSocketHandler {
    /**
     * 建立websocket连接后触发的方法
     *
     * @param session
     * @throws Exception
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        super.afterConnectionEstablished(session);
    }

    /**
     * 收到消息的处理逻辑
     *
     * @param session
     * @param message
     * @throws Exception
     */
    @Override
    public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) throws Exception {
        super.handleMessage(session, message);
    }

    /**
     * 自动处理Ping/Pong消息，实际上就是心跳消息处理
     * @param session
     * @param message
     * @throws Exception
     */
    @Override
    protected void handlePongMessage(WebSocketSession session, PongMessage message) throws Exception {
        super.handlePongMessage(session, message);
    }

    /**
     * websocket连接异常处理
     *
     * @param session
     * @param exception
     * @throws Exception
     */
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        super.handleTransportError(session, exception);
    }

    /**
     * websocket连接关闭后处理
     *
     * @param session
     * @param status
     * @throws Exception
     */
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        super.afterConnectionClosed(session, status);
    }

    /***
     * supportsPartialMessages 方法的主要作用是判断当前的 TextWebSocketHandler 是否支持处理部分消息（partial messages）。
     * 当 WebSocket 客户端发送一个很大的消息时，可能会分为多个帧发送。如果你的应用程序希望能够处理这种拆分的消息（部分消息），
     * 你可以实现 supportsPartialMessages 方法并返回 true。这表示你的 TextWebSocketHandler 实现支持处理这些部分消息。
     * 默认情况下，supportsPartialMessages 方法返回 false 表示不支持处理部分消息，消息应该一次性完成传输
     */
    @Override
    public boolean supportsPartialMessages() {
        return super.supportsPartialMessages();
    }
}
```
# 参考的文章
https://www.cnblogs.com/antLaddie/p/17366438.html
