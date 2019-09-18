# 黑马 - WebSocket

## 什么是 WebSocket？

```shell
# WebSocket 是 HTML5 一种新的协议。
	# 实现了浏览器与服务器的全双工通信<full-duplex>
	# 一开始的握手需要借助 HTTP 请求完成
	# 在单个 TCP 连接上进行全双工通讯协议
	
# 全双工和单工的区别？
	# 全双工
		# A -> B && B -> A
	# 单工
		# A -> B || B -> A
```

## Http 与 WebSocket 的区别

### Http

```shell
# http 是短连接，每次请求之后，都会关闭连接。
```

### WebSocket

```shell
# WebSocket 是长连接，只需要通过一次请求来初始化链接，然后所有的请求和响应都通过这个 TCP 链接进行通讯。
```

## WebSocket 的相关注解说明

```shell
# @ServerEndpoint("/websocket/{uid}")
	# 申明这是一个 WebSocket 服务
	# 指定访问该服务的地址，地址中指定参数用 {} 占位
	
# @OnOpen
	# public void onOpen(Session session,@PathParam("uid) String uid) 
												# throws IOException{}
		# 该方法将在建立连接后执行，会传入 session 对象，即客户端与服务端建立的长连接通道
		# 通过 @PathParam 获取 url 申明的参数
		
# @OnClose
	# public void onClose{}
	# 关闭连接后执行
	
# @OnMessage
	# public void onMessage(String message,Session session) throws IOException{}
	# 接收客户端发来的消息
	# message : 发来的消息数据
	# session : 会话对象(也是通道)
	
# @OnError
	# 捕获错误
	
#  发送消息到客户端
	# 用法 session.getBasicRemote().sendText("你好");
	# 通过 session 进行发送
```

## SpringBoot 整合 WebSocket

### 导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### 编写 WebSocketHandler

```shell
# SpringBoot 中，处理消息的具体业务逻辑需要实现 WebSocketHandler 接口
```

```java
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import java.io.IOException;

public class MyHandler extends TextWebSocketHandler {
    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message)
            throws IOException {
        System.out.println("获取到消息 >> " + message.getPayload());
        session.sendMessage(new TextMessage("消息已收到"));
        if (message.getPayload().equals("10")) {
            for (int i = 0; i < 10; i++) {
                session.sendMessage(new TextMessage("消息 -> " + i));
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws
            Exception {
        session.sendMessage(new TextMessage("欢迎连接到ws服务"));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status)
            throws Exception {
        System.out.println("断开连接！");
    }
}
```

### 编写配置类

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/ws")
            .setAllowedOrigins("*")
            .addInterceptors(this.myHandshakeInterceptor);
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }
}
```

### WebSocket 拦截器

```shell
# SpringBoot 中提供了 WebSocket 拦截器，在建立连接前实现一些业务逻辑。
```

```java
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.Map;

@Component
public class MyHandshakeInterceptor implements HandshakeInterceptor {
    
    /**
     * 握手之前，若返回false，则不建立链接
     */
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse
            response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws
            Exception {
        // 将用户id放入socket处理器的会话(WebSocketSession)中
        attributes.put("uid", 1001);
        System.out.println("开始握手。。。。。。。");
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse
            response, WebSocketHandler wsHandler, Exception exception) {
        System.out.println("握手成功啦。。。。。。");
    }
}
```

