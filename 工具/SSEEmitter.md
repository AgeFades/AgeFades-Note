[TOC]

# SSEEmitter

## 参考资料

[SSE简介&SpringBoot整合使用](https://juejin.cn/post/6844904113226924045)

[Vue SSE使用](https://juejin.cn/post/7009079180482576420)

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
 * @author DuChao
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