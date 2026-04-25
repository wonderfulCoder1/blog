---
title: SSE（Server-Sent Events）后端推送消息到前端
date: 2026-04-25 18:18:42
categories:
  - 技术
tags:
  - SSE
  - websocket
  - java
---

# SSE（Server-Sent Events）后端推送消息到前端

> 该方式仅适用于后端单向向前端推流，例如实时数据推送，
>
> 如果需要双向请使websocket

### 使用方法

#### 依赖

使用的是`springmvc`的功能，所以只需要导入`springmvc`就可以了

```xml
<!-- SpringBoot Web容器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



#### controller

> 与正常`controller`接口方法类似，但是返回值是`SseEmitter`，不同前端访问该接口都会生成一个`SseEmitter`,我们可以用`Map`保存它，在数据发生变化后，使用`SseEmitter`对象的`send`方法



```java
@ApiOperation(value = "sse登录连接")
@GetMapping("/web/sseOpen")
public SseEmitter handleSse() {
    // 设置默认的超时时间60秒，超时之后服务端主动关闭连接。
    SseEmitter emitter = new SseEmitter(sseTimeout * 1000L);
    Long userId = SecurityUtils.getUserId();
    SseMap.sseEmitters.put(userId,emitter);

    emitter.onCompletion(() -> SseMap.sseEmitters.remove(userId));
    emitter.onTimeout(() -> SseMap.sseEmitters.remove(userId));
    AtomicLong counter = new AtomicLong();
    new Thread(() -> {
        try {
            emitter.send(SseEmitter.event()
                         .id(String.valueOf(counter.incrementAndGet()))
                         .name("message")
                         .data("连接成功"));
            for (int i = 0; i < 5000; i++) {
                emitter.send(SseEmitter.event()
                             .id(String.valueOf(counter.incrementAndGet()))
                             .name("message")
                             .data("This is message " + i));
                Thread.sleep(10000);
            }
            // emitter.complete();
        } catch (IOException | InterruptedException e) {
            emitter.completeWithError(e);
        }
    }).start();

    return emitter;
}
```

#### SseEmitter容器

> `emitter.send`方法需要的对象参数中，
>
> - `id`可以按需填写，`uuid`应该也可以
> - `name`需要注意，前端需要监听这一事件，`eventSource.addEventListener`中就需要填写对应`name`
> - `data` 就是你传输的数据，可以传`json`字符串，方便解析

```java

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

@Slf4j
public class SseMap {
    public static Map<Long, SseEmitter> sseEmitters = new ConcurrentHashMap<>();
    
    
    public static void notifyExit(Long userId,String msg){
        new Thread(() -> {
            //     sseMap退出通知
            SseEmitter sseEmitter = SseMap.sseEmitters.get(userId);
            if(sseEmitter==null){
                return;
            }
            
            try {
                sseEmitter.send(SseEmitter.event().name("finish").id(userId.toString()).data(msg));
                sseEmitter.complete();
            } catch (IOException e) {
                sseEmitter.completeWithError(e);
                // throw new RuntimeException(e);
                log.info("sse异常",e);
            }
        }).start();
    }
    
}
```

#### 前端调试

可以使用`window.EventSource`或者`event-source-polyfill`(支持携带header，鉴权)

```html
<!DOCTYPE html>
<html>

<head>
    <title>SSE Example</title>
</head>

<body>
    <h1>SSE Example</h1>
    <div id="sse-data"></div>

    <button onclick="aaa()">aaa</button>
    <button onclick="bbb()">bbb</button>
    <button onclick="ccc()">ccc</button>
    <script src="eventsource.js"></script>
    <script>
        // import { EventSourcePolyfill } from 'event-source-polyfill';

        const sseDataElement = document.getElementById('sse-data');

        function aaa() {
            const eventSource = new EventSource('http://127.0.0.1:8080/sse');

            eventSource.onmessage = function (event) {
                // const data = JSON.parse(event.data);
                const data = event.data;
                sseDataElement.innerHTML += data + '<br>';
            };

            eventSource.onopen = function (event) {
                sseDataElement.innerHTML = 'Connection opened<br>';
            };

            eventSource.onerror = function (event) {
                console.log(event)
                if (event.target.readyState === EventSource.CLOSED) {
                    sseDataElement.innerHTML += 'Connection closed<br>';
                } else {
                    sseDataElement.innerHTML += 'Error occurred<br>';
                }
                eventSource.close()
            };
        }

        function bbb() {

            const eventSource = new EventSourcePolyfill('http://127.0.0.1:8080/sse', {
                headers: {
                    'X-Custom-Header': 'value'
                }
            });
            /*
              * open：订阅成功（和后端连接成功）
              */
            eventSource.addEventListener("open", function (e) {
                console.log('open successfully')
            })
            /*
            * message：后端返回信息，格式可以和后端协商
            */
            eventSource.addEventListener("message", function (e) {
                console.log(e.data)
            })

            // eventSource.onmessage = function (event) {
            //     console.log(event);
            //     // const data = JSON.parse(event.data);
            //     const data = event.data;
            //     sseDataElement.innerHTML += data + '<br>';
            // };

            // eventSource.onopen = function (event) {
            //     console.log(event);
            //     sseDataElement.innerHTML = 'Connection opened<br>';
            // };
            /*
            * error：错误（可能是断开，可能是后端返回的信息）
            */
            eventSource.addEventListener("error", function (err) {
                console.log(err)
                // 类似的返回信息验证，这里是实例
                err && err.status === 401 && console.log('not authorized')
                eventSource.close()
            })

            //自定义finish事件，主动关闭EventSource
            eventSource.addEventListener('finish', function (e) {
                console.log(e)
                // source.close();
                let a = "<br>" + "哈哈" + "<br/>";
                sseDataElement.innerHTML = a;
            }, false);
        }

        function ccc() {

            const eventSource = new EventSourcePolyfill(    
                'http://127.0.0.1:8080/api/web/sseOpen'
            {
                headers: {
                    'X-Custom-Header': 'value',
                    'Authorization': 'xxxxxxxx'
                },
                heartbeatTimeout: 15000,
            });
            /*
              * open：订阅成功（和后端连接成功）
              */
            eventSource.addEventListener("open", function (e) {
                console.log(e);
                console.log('open successfully')
            })
            /*
            * message：后端返回信息，格式可以和后端协商
            */
            eventSource.addEventListener("message", function (e) {
                console.log(e.data)
            })

            // eventSource.onmessage = function (event) {
            //     console.log(event);
            //     // const data = JSON.parse(event.data);
            //     const data = event.data;
            //     sseDataElement.innerHTML += data + '<br>';
            // };

            // eventSource.onopen = function (event) {
            //     console.log(event);
            //     sseDataElement.innerHTML = 'Connection opened<br>';
            // };
            /*
            * error：错误（可能是断开，可能是后端返回的信息）
            */
            eventSource.addEventListener("error", function (err) {
                console.log(err)
                // 类似的返回信息验证，这里是实例
                // err && err.status === 401 && console.log('not authorized')
                // eventSource.close()
            })

            //自定义finish事件，主动关闭EventSource
            eventSource.addEventListener('finish', function (e) {
                console.log(e)
                // source.close();
                let a = "<br>" + "哈哈" + "<br/>";
                sseDataElement.innerHTML = a;
                eventSource.close()
            }, false);
        }

    </script>
</body>

</html>
```

#### 注意事项

- 电脑使用**kx上网**的情况，有概率前后端连接失败
- `window.EventSource`在浏览器中`f12`能够看到请求报文， `event-source-polyfill`不可以，建议使用`apifox`进行测试
- `EventSource`在断连后，会自动重连，需要使用`close`才能关闭



**参考：** 

-  [spring sse](https://docs.spring.io/spring-framework/docs/5.2.25.RELEASE/spring-framework-reference/web.html#mvc-ann-async-sse)
-  [EventSourcePolyfill](https://github.com/Yaffle/EventSource/)
-  [EventSource](https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource)

