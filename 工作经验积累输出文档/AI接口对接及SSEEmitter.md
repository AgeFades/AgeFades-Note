[TOC]

# AI接口对接及SSEEmitter

## 参考资料

[SSE简介&SpringBoot整合使用](https://juejin.cn/post/6844904113226924045)

[Vue SSE使用](https://juejin.cn/post/7009079180482576420)

[对接ChatGPT使用SDK](https://github.com/PlexPt/chatgpt-java)

## 使用案例

### SSE工具类

```java
package com.agefades.log.common.core.util;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

/**
 * SSE工具类
 *
 * @author AgeFades
 * @since 2023/2/16 14:43
 */
@Slf4j
public class SSEEmitterUtil {

    /**
     * 已连接接入记录
     */
    private static final Map<String, SseEmitter> sseEmitterMap = new ConcurrentHashMap<>();

    /**
     * 创建连接并返回 SseEmitter
     *
     * @param key     key
     * @param timeout 超时时间,单位毫秒
     * @return SseEmitter
     */
    public static SseEmitter connect(String key, Long timeout) {
        log.info("SSE连接[{}]建立", key);
        if (sseEmitterMap.containsKey(key)) {
            return sseEmitterMap.get(key);
        }
        SseEmitter sseEmitter = new SseEmitter(timeout);
        sseEmitter.onCompletion(completionCallBack(key));
        sseEmitter.onError(errorCallBack(key));
        sseEmitter.onTimeout(timeoutCallBack(key));
        sseEmitterMap.put(key, sseEmitter);
        return sseEmitter;
    }

    /**
     * 向前端同步数据
     */
    public static void send(String key, String message) {
        if (sseEmitterMap.containsKey(key)) {
            try {
                sseEmitterMap.get(key).send(message);
                log.info("SSE连接[{}]发送消息, {}", key, message);
            } catch (IOException e) {
                log.error("connect sync key [{}] exception:{}", key, e.getMessage());
                remove(key);
            }
        }
    }

    /**
     * 移除连接
     */
    public static void remove(String key) {
        sseEmitterMap.remove(key);
        log.info("SSE连接[{}]移除", key);
    }

    private static Runnable completionCallBack(String key) {
        return () -> {
            log.info("connect close：{}", key);
            remove(key);
        };
    }

    private static Runnable timeoutCallBack(String key) {
        return () -> {
            log.info("connect timeout：{}", key);
            remove(key);
        };
    }

    private static Consumer<Throwable> errorCallBack(String key) {
        return throwable -> {
            log.info("connect exception：{}", key);
            remove(key);
        };
    }

}
```

### SseHelper

```java
import lombok.experimental.UtilityClass;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

@Slf4j
@UtilityClass
public class SseHelper {

    public void complete(SseEmitter sseEmitter) {
        if (sseEmitter != null) {
            try {
                sseEmitter.complete();
            } catch (Exception e) {
                log.error("sse连接关闭异常: {}", e.getMessage());
            }
        }
    }

    public void send(SseEmitter sseEmitter, Object data) {
        try {
            sseEmitter.send(SseEmitter.event()
                    .name("message")
                    .data(data));
        } catch (Exception e) {
            log.error("sse发送消息{}, 异常: {},", data, e.getMessage());
        }
    }

    public void sendError(SseEmitter sseEmitter, String msg) {
        try {
            sseEmitter.send(SseEmitter.event()
                    .name("error")
                    .data(msg));
        } catch (Exception e) {
            log.error("sse发送消息异常: {}", e.getMessage());
        } finally {
            SseHelper.complete(sseEmitter);
        }
    }

}
```

### 测试接口

```java
@GetMapping(value = "sse/{key}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
@ApiOperation(value = "测试SSE连接链接")
public SseEmitter testSSEConnect(@PathVariable String key) {
  return SSEEmitterUtil.connect(key, 600 * 1000L);
}

@GetMapping(value = "sse/message/{key}")
@ApiOperation(value = "测试SSE连接接收消息")
public Result<Void> testSSEMsg(@PathVariable String key) {
  try {
    Thread.sleep(3);
    SSEEmitterUtil.send(key, "AgeFades");
    Thread.sleep(1);
    SSEEmitterUtil.send(key, "Hello");
    Thread.sleep(1);
    SSEEmitterUtil.send(key, "World");
    SSEEmitterUtil.remove(key);
  } catch (InterruptedException e) {
    e.printStackTrace();
  }
  return Result.success();
}
```

### 测试效果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1676531840055.png)

1. 先调用建立连接的接口，获取SSE长连接
2. 无论何种触发方式，后端会往第一步创建的SSE连接中推送消息（比如支付状态：1.发起支付、2.支付中、3.支付成功(支付失败)...）不用前端在当前页面轮训调用接口获取最新状态
3. 消息推完或者发生异常或者连接超时，前后端均关闭该连接

## 对接ChatGPT

[参照这个接入示例](https://github.com/PlexPt/chatgpt-online-springboot)

### 服务器访问chatgpt api

1. 在服务器挂梯子做代理
   1. [如何在linux使用clash代理](https://github.com/wanhebin/clash-for-linux)
   2. [梯子购买](https://portal.shadowsocks.au/aff.php?aff=42440)
   3. [ClashX在Mac的使用](https://portal.shadowsocks.nz/knowledgebase/190/macOSClashX-%E8%AE%BE%E7%BD%AE%E6%96%B9%E6%B3%95.html)

2. 购买国外服务器，比如AWS

   1. 国外服务器公网ip，国内服务器可访问，比如在腾讯云服务器上curl国外服务器ip地址（安装了nginx）
   2. 国外服务器安装nginx配置反向代理，路由流量至chatgpt api域名上
      1. 修改nginx.conf配置如下

   ```bash
   # For more information on configuration, see:
   #   * Official English Documentation: http://nginx.org/en/docs/
   #   * Official Russian Documentation: http://nginx.org/ru/docs/
   
   user nginx;
   worker_processes auto;
   error_log /var/log/nginx/error.log notice;
   pid /run/nginx.pid;
   
   # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
   include /usr/share/nginx/modules/*.conf;
   
   events {
       worker_connections 1024;
   }
   
   http {
       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
   
       access_log  /var/log/nginx/access.log  main;
   
       sendfile            on;
       tcp_nopush          on;
       keepalive_timeout   65;
       types_hash_max_size 4096;
   
       include             /etc/nginx/mime.types;
       default_type        application/octet-stream;
   
       # Load modular configuration files from the /etc/nginx/conf.d directory.
       # See http://nginx.org/en/docs/ngx_core_module.html#include
       # for more information.
       include /etc/nginx/conf.d/*.conf;
   
       server {
           listen       80;
           listen       [::]:80;
           server_name  _;
          # root         /usr/share/nginx/html;
   
           # Load configuration files for the default server block.
           include /etc/nginx/default.d/*.conf;
   
           #error_page 404 /404.html;
           #location = /404.html {
           #}
   
           #error_page 500 502 503 504 /50x.html;
           #location = /50x.html {
           #}
   	location / {
   
   	proxy_pass https://api.openai.com;
   
   	proxy_set_header Host api.openai.com;
   
   	proxy_set_header Connection '';
   
   	proxy_http_version 1.1;
   
   	chunked_transfer_encoding off;
   
   	proxy_buffering off;
   
   	proxy_ssl_server_name on;
   
   	proxy_cache off;
   
   	proxy_set_header X-Forwarded-For $remote_addr;
   
   	proxy_set_header X-Forwarded-Proto $scheme;
   
   	}
       }
   
   # Settings for a TLS enabled server.
   #
   #    server {
   #        listen       443 ssl http2;
   #        listen       [::]:443 ssl http2;
   #        server_name  _;
   #        root         /usr/share/nginx/html;
   #
   #        ssl_certificate "/etc/pki/nginx/server.crt";
   #        ssl_certificate_key "/etc/pki/nginx/private/server.key";
   #        ssl_session_cache shared:SSL:1m;
   #        ssl_session_timeout  10m;
   #        ssl_ciphers PROFILE=SYSTEM;
   #        ssl_prefer_server_ciphers on;
   #
   #        # Load configuration files for the default server block.
   #        include /etc/nginx/default.d/*.conf;
   #
   #        error_page 404 /404.html;
   #        location = /404.html {
   #        }
   #
   #        error_page 500 502 503 504 /50x.html;
   #        location = /50x.html {
   #        }
   #    }
   
   }
   ```

## Nginx支持SSE协议

- 对接了ChatGPT API的服务部署到服务器后，访问AI聊天接口，SSE连接要等到所有数据推送完成后，才会给客户端一次性响应，这是因为nginx没有配置对sse协议的支持
- 需要添加配置如下

```apl
proxy_set_header Connection '';
proxy_http_version 1.1;
chunked_transfer_encoding off;
proxy_buffering off;
proxy_cache off;
```

- 完整配置如下

```apl
server {
    listen 80;
    server_name 你的域名;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
   listen       443;
   server_name  你的域名;
   ssl on;
   ssl_certificate 你的证书;
   ssl_certificate_key 你的证书;
   index index.html index.htm index.php index.jsp;
   charset utf-8;

   location / {
      proxy_pass          你的服务端口ip端口;
      proxy_redirect      default;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    Range $http_range;
      proxy_set_header X-Forwarded-Proto  $scheme;
      client_max_body_size 100m;
      client_body_buffer_size 128k;
      proxy_connect_timeout 600;
      proxy_send_timeout 600;
      proxy_read_timeout 600;
      proxy_buffer_size 16k;
      proxy_buffers 4 32k;
      proxy_busy_buffers_size 64k;
      proxy_temp_file_write_size 64k;
      # 支持sse的配置项
			proxy_set_header Connection '';
   		proxy_http_version 1.1;
    	chunked_transfer_encoding off;
    	proxy_buffering off;
    	proxy_cache off;
	}

}
```

## 对接科大讯飞星火

### XfRsq

```java

import cn.hutool.core.annotation.Alias;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@NoArgsConstructor
@AllArgsConstructor
@Builder
@Data
public class XfReq {
    private Header header;
    private Parameter parameter;
    private Payload payload;

    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @Data
    public static class Header {
        @Alias("app_id")
        private String appId;
        private String uid;
    }

    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @Data
    public static class Parameter {
        private Chat chat;

        @NoArgsConstructor
        @AllArgsConstructor
        @Builder
        @Data
        public static class Chat {
            private String domain = "general";
            private Double temperature = 0.5d;
            @Alias("max_tokens")
            private Integer maxTokens = 1024;
        }
    }

    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @Data
    public static class Payload {
        private Message message;

        @NoArgsConstructor
        @AllArgsConstructor
        @Builder
        @Data
        public static class Message {
            private List<Text> text;

            @NoArgsConstructor
            @AllArgsConstructor
            @Builder
            @Data
            public static class Text {
                private String role;
                private String content;
            }
        }
    }
}
```

### XfService

```java
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.date.TimeInterval;
import cn.hutool.json.JSONUtil;
import com.google.gson.Gson;
import com.plexpt.chatgpt.entity.chat.Message;
import lombok.*;
import lombok.extern.slf4j.Slf4j;
import okhttp3.*;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.function.Consumer;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class XfService {

    // 地址与鉴权信息
    @Value("${xf.host_url:https://spark-api.xf-yun.com/v1.1/chat}")
    String hostUrl;

    @Value("${xf.app_id:0bce6c72}")
    String appId;

    @Value("${xf.api_secret:NGZhNzY2ZTBmMjRlMTgzNDQ4OGVlOGQw}")
    String apiSecret;

    @Value("${xf.api_key:9af7a9353ade89d8bec8419b9247d7fb}")
    String apiKey;

//    public SseEmitter sseConversation(SseEmitter sseEmitter, String content, String messages) {
//        handleSseMsg(sseEmitter, UserThreadLocalUtil.getUserId(), content, messages, 3);
//        return sseEmitter;
//    }

    private void handleSseMsg(SseEmitter sseEmitter, String userId, String messageId, String content, List<XfReq.Payload.Message.Text> messages, int retry, Consumer<String> onComplete) {
        try {
            String authUrl = getAuthUrl(hostUrl, apiKey, apiSecret);
            OkHttpClient client = new OkHttpClient.Builder().build();
            String url = authUrl.replace("http://", "ws://").replace("https://", "wss://");
            Request request = new Request.Builder().url(url).build();
            MyWebSocketListener listener = new MyWebSocketListener(sseEmitter, userId, messageId, content, messages, retry, false);
            listener.setOnComplete(onComplete);
            client.newWebSocket(request, listener);
        } catch (Exception e) {
            SseHelper.sendError(sseEmitter, e.getMessage());
        }
    }

    /**
     * 鉴权方法
     */
    public static String getAuthUrl(String hostUrl, String apiKey, String apiSecret) throws Exception {
        URL url = new URL(hostUrl);
        // 时间
        SimpleDateFormat format = new SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss z", Locale.US);
        format.setTimeZone(TimeZone.getTimeZone("GMT"));
        String date = format.format(new Date());
        // 拼接
        String preStr = "host: " + url.getHost() + "\n" +
                "date: " + date + "\n" +
                "GET " + url.getPath() + " HTTP/1.1";
        // SHA256加密
        Mac mac = Mac.getInstance("hmacsha256");
        SecretKeySpec spec = new SecretKeySpec(apiSecret.getBytes(StandardCharsets.UTF_8), "hmacsha256");
        mac.init(spec);

        byte[] hexDigits = mac.doFinal(preStr.getBytes(StandardCharsets.UTF_8));
        // Base64加密
        String sha = Base64.getEncoder().encodeToString(hexDigits);
        // 拼接
        String authorization = String.format("api_key=\"%s\", algorithm=\"%s\", headers=\"%s\", signature=\"%s\"", apiKey, "hmac-sha256", "host date request-line", sha);
        // 拼接地址
        HttpUrl httpUrl = Objects.requireNonNull(HttpUrl.parse("https://" + url.getHost() + url.getPath())).newBuilder().//
                addQueryParameter("authorization", Base64.getEncoder().encodeToString(authorization.getBytes(StandardCharsets.UTF_8))).//
                addQueryParameter("date", date).
                addQueryParameter("host", url.getHost()).
                build();

        return httpUrl.toString();
    }

    public void sseConversation(SseEmitter sseEmitter, User user, String messageId, String prompt, List<Message> messages, Consumer<String> onComplete) {
        handleSseMsg(sseEmitter, user.getId(), messageId, prompt, convert(messages), 3, onComplete);
    }

    private List<XfReq.Payload.Message.Text> convert(List<Message> messages) {
        return messages.stream().map(v -> XfReq.Payload.Message.Text.builder()
                .content(v.getContent())
                .role(v.getRole())
                .build()).collect(Collectors.toList());
    }

    @EqualsAndHashCode(callSuper = true)
    @RequiredArgsConstructor
    @Data
    class MyWebSocketListener extends WebSocketListener {

        private SseEmitter sseEmitter;
        private String userId;
        private String messageId;
        private String content;
        private List<XfReq.Payload.Message.Text> messages;
        private Integer retry;
        private Boolean wsCloseFlag;
        StringBuffer last = new StringBuffer();
        TimeInterval timer;
        @Setter
        Consumer<String> onComplete = s -> {

        };

        public final Gson gson = new Gson();

        public MyWebSocketListener(SseEmitter sseEmitter, String userId, String messageId, String content, List<XfReq.Payload.Message.Text> messages, Integer retry, Boolean wsCloseFlag) {
            this.sseEmitter = sseEmitter;
            this.userId = userId;
            this.messageId = messageId;
            this.content = content;
            this.messages = messages;
            this.retry = retry;
            this.wsCloseFlag = wsCloseFlag;
        }

        @Override
        public void onOpen(WebSocket webSocket, Response response) {
            timer = DateUtil.timer();
            super.onOpen(webSocket, response);
            new Thread(() -> {
                try {
                    // 请求参数json串
//                    List<XfReq.Payload.Message.Text> text = new ArrayList<>();
//                    if (StringUtils.isNotEmpty(messages)) {
//                        text.addAll(JSONArray.parseArray(messages, XfReq.Payload.Message.Text.class));
//                    }
//                    text.add(XfReq.Payload.Message.Text.builder().role("user").content(content).build());
                    XfReq req = XfReq.builder()
                            .header(XfReq.Header.builder()
                                    .appId(appId)
                                    .uid(userId)
                                    .build())
                            .parameter(XfReq.Parameter.builder()
                                    .chat(XfReq.Parameter.Chat.builder()
                                            .domain("general")
                                            .temperature(0.5d)
                                            .maxTokens(1024)
                                            .build())
                                    .build())
                            .payload(XfReq.Payload.builder()
                                    .message(XfReq.Payload.Message.builder()
                                            .text(messages).build())
                                    .build())
                            .build();
                    String jsonStr = JSONUtil.toJsonStr(req);
                    log.info("星火请求对象: {}", jsonStr);
                    webSocket.send(jsonStr);
                    // 等待服务端返回完毕后关闭
                    while (true) {
                        Thread.sleep(200);
                        if (wsCloseFlag) {
                            break;
                        }
                    }
                    webSocket.close(1000, "");
                    SseHelper.complete(sseEmitter);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        @Override
        public void onMessage(WebSocket webSocket, String text) {
            JsonParse myJsonParse = gson.fromJson(text, JsonParse.class);
//            log.info("星火响应内容: {}", text);
            if (myJsonParse.header.code != 0) {
                // 授权错误：秒级流控超限。秒级并发超过授权路数限制
                if (myJsonParse.header.code == 11202 && retry != 0) {
                    try {
                        Thread.sleep(30000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    int retryCount = retry - 1;
                    handleSseMsg(sseEmitter, userId, messageId, content, messages, retryCount, onComplete);
                    log.info("星火重试: {}", retryCount);
                } else {
                    webSocket.close(1000, "");
                    SseHelper.sendError(sseEmitter, "发生错误，错误码为：" + myJsonParse.header.code + ", 本次请求的sid为：" + myJsonParse.header.sid);
                }
            }
            List<Text> textList = myJsonParse.payload.choices.text;
            for (Text temp : textList) {
                last.append(temp.content);
                SseHelper.send(sseEmitter, temp);
            }
            if (myJsonParse.header.status == 2) {
                // 可以关闭连接，释放资源
                wsCloseFlag = true;
            }
        }

        @Override
        public void onClosed(@NotNull WebSocket webSocket, int code, @NotNull String reason) {
            super.onClosed(webSocket, code, reason);
            log.info("回答完成,耗时: {}, code: {}, reason: {}, last: {}", timer.interval(), code, reason, last);
            onComplete.accept(last.toString());
        }

        @Override
        public void onFailure(@NotNull WebSocket webSocket, @NotNull Throwable t, @Nullable Response response) {
            super.onFailure(webSocket, t, response);
            webSocket.close(1000, "");
            SseHelper.sendError(sseEmitter, "发生错误, 异常信息: " + t.getMessage());
        }

        //返回的json结果拆解
        class JsonParse {
            Header header;
            Payload payload;
        }

        class Header {
            int code;
            int status;
            String sid;
        }

        class Payload {
            Choices choices;
        }

        class Choices {
            List<Text> text;
        }

        @NoArgsConstructor
        @AllArgsConstructor
        @Data
        class Text {
            String role;
            String content;
        }

    }

}
```

### AIDomin

```java
public void processSse(String token, String messageId, String prompt, String model, Integer textType, String translate, SseEmitter sseEmitter) {
        String userId = JwtUtil.parseJwt(token);
        User user = userService.getById(userId);
        if (user == null) {
            SseHelper.sendError(sseEmitter, "非法用户");
            return;
        }
        if (StrUtil.isBlank(user.getOpenId()) && StrUtil.isBlank(user.getEmail())) {
            SseHelper.sendError(sseEmitter, "当前操作需要登陆");
            return;
        }

        // 文本处理工具提示词
        if (ObjectUtil.isNotNull(textType)) {
            String callWord = CALL_WORD_MAP.get(textType);
            if (ObjectUtil.equal(textType, 2)) {
                if (StrUtil.isBlank(translate)) {
                    translate = "英文";
                }
                callWord = StrUtil.format(callWord, translate);
            }
            prompt = callWord + prompt;
        }

        Boolean judgeSensitivityWord = SensitiveWordUtil.judgeSensitivityWord(prompt);
        if (judgeSensitivityWord) {
            SseHelper.sendError(sseEmitter, "作为一个人工智能语言模型，我无法提供此类信息。 这种类型的信息可能会违反法律法规，并对用户造成严重的心理和社交伤害。 建议遵守相关的法律法规和社会道德规范，并寻找其他有益和健康的娱乐方式。");
            return;
        }

        // 用户剩余次数扣除
        String cacheKey = RedisConstant.CHAT3_PREFIX + userId + ":" + LocalDate.now();
        boolean isGpt4 = StrUtil.equals(model, "2");
        if (isGpt4) {
            cacheKey = RedisConstant.CHAT4_PREFIX + userId + ":" + LocalDate.now();
        }
        String countStr = redisTemplate.opsForValue().get(cacheKey);
        Integer count = Convert.toInt(countStr, 0);

        ChatQuotaSetUp quotaSetUp = chatQuotaSetUpService.list().get(0);
        Integer everyDayCount = quotaSetUp.getEveryDay35();
        if (isGpt4) {
            everyDayCount = quotaSetUp.getEveryDay4();
        }

        Integer inviteCount = user.getQuota35();
        if (isGpt4) {
            inviteCount = user.getQuota4();
        }
        if (inviteCount == null) {
            inviteCount = 0;
        }
        if (count >= everyDayCount + inviteCount) {
            SseHelper.sendError(sseEmitter, "您的次数已使用完，可通过邀请新用户获得次数");
            return;
        }

        Message message = Message.of(prompt);
        List<ChatMessageHistory> list = chatMessageHistoryService.lambdaQuery().eq(ChatMessageHistory::getMessageId, messageId).list();
        List<Message> messages = list.stream().map(v -> Message.builder()
                .role(v.getRole())
                .content(v.getContent())
                .build()).collect(Collectors.toList());
        messages.add(message);

        String title = StrUtil.sub(prompt, 0, 255);
        if (CollUtil.isNotEmpty(list)) {
            title = list.get(0).getTitle();
        }

        String finalTitle = title;
        AtomicInteger atomicCount = new AtomicInteger(count);
        AtomicInteger atomicEveryDayCount = new AtomicInteger(everyDayCount);
        AtomicInteger atomicInviteCount = new AtomicInteger(inviteCount);
        String finalCacheKey = cacheKey;
        Consumer<String> onComplete = getOnComplete(messageId, textType, userId, user, isGpt4, message, finalTitle, atomicCount, atomicEveryDayCount, atomicInviteCount, finalCacheKey);

        if (StrUtil.equals(model, "3")) {
            xfService.sseConversation(sseEmitter, user, messageId, prompt, messages, onComplete);
            return;
        }

        ChatGPTStream chatGPTStream = ChatGPTStream.builder()
                .timeout(50)
                .apiKey(keyManager.getKey())
//                .proxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("127.0.0.1", 7890)))
                .apiHost(OPENAI_URL)
                .build()
                .init();

        GPTEventSourceListener listener = new GPTEventSourceListener(sseEmitter);

        listener.setOnComplete(onComplete);

        if (isGpt4) {
            ChatCompletion chatCompletion = ChatCompletion.builder()
                    .model(ChatCompletion.Model.GPT_4.getName())
                    .messages(messages)
                    .maxTokens(8000)
                    .temperature(0.9)
                    .build();
            chatGPTStream.streamChatCompletion(chatCompletion, listener);
        } else {
            chatGPTStream.streamChatCompletion(messages, listener);
        }
    }

    @NotNull
    private Consumer<String> getOnComplete(String messageId, Integer textType, String userId, User user, boolean isGpt4, Message message, String finalTitle, AtomicInteger atomicCount, AtomicInteger atomicEveryDayCount, AtomicInteger atomicInviteCount, String finalCacheKey) {
        return msg -> {
            if (ObjectUtil.isNull(textType)) {
                // 关闭连接时存聊天对话数据
                ChatMessageHistory userMsg = ChatMessageHistory.builder()
                        .userId(userId)
                        .messageId(messageId)
                        .title(finalTitle)
                        .role(message.getRole())
                        .content(message.getContent())
                        .build();

                Message aiMsg = Message.ofAssistant(msg);
                ChatMessageHistory aiMsgHistory = ChatMessageHistory.builder()
                        .userId(userId)
                        .messageId(messageId)
                        .title(finalTitle)
                        .role(aiMsg.getRole())
                        .content(aiMsg.getContent())
                        .build();
                chatMessageHistoryService.saveBatch(Arrays.asList(userMsg, aiMsgHistory));
            }
            // 处理次数
            int incremented = atomicCount.incrementAndGet();
            redisTemplate.opsForValue().set(finalCacheKey, Convert.toStr(incremented), 1, TimeUnit.DAYS);
            if (incremented > atomicEveryDayCount.get()) {
                int decremented = atomicInviteCount.decrementAndGet();
                if (isGpt4) {
                    user.setQuota4(decremented);
                } else {
                    user.setQuota35(decremented);
                }
                userService.updateById(user);
            }
        };
    }
```

