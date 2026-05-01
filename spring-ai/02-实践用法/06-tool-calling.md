# Tool Calling 实战：让 AI 调用你的 Spring Bean

读完这一篇，你能：

- 用 `@Tool` 把任意 Spring Bean 的方法暴露成模型可调用的"工具"
- 在一个调用里同时挂多个工具，让 ChatClient 自动跑完 tool call loop
- 通过 `ToolContext` 把"当前用户 ID"这种业务上下文带进 tool 方法
- 用 `FunctionToolCallback` 注册一个 lambda 做工具，不必走 Bean
- 知道 tool 抛异常时框架默认怎么处理、怎么改

下面用一个"订单查询 + 计算 + 天气"的小例子从零搭起来。

## 前置依赖

需要 JDK 17+、Spring Boot 3.4+（仓库当前 main 已经在准备 Boot 4，这里用 3.4 演示，两边 API 一致）。Maven 加两条依赖：

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`application.yml`：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
```

`@Tool` 是注解，不用额外 starter；它定义在 `spring-ai-model` 里，由 chat starter 间接引入。注解原型见 `spring-ai-model/src/main/java/org/springframework/ai/tool/annotation/Tool.java:37-71`：四个属性 `name / description / returnDirect / resultConverter`，其中只有 `description` 几乎每次都会写。

## 最简示例：一个 Bean，一个 `@Tool` 方法

写一个查订单的 Bean。注意 `@Tool` 加在方法上，参数用 `@ToolParam` 描述：

```java
package com.example.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    @Tool(description = "查询指定订单号的订单详情，返回 JSON 字符串")
    public String getOrder(
            @ToolParam(description = "订单号，形如 ORD-2024-0001") String orderId) {
        // 真实场景这里会查 DB；演示用就返回一个固定结果
        return """
               {"orderId":"%s","status":"PAID","amount":128.50,"items":2}
               """.formatted(orderId);
    }
}
```

然后写 controller，把 `OrderService` 实例直接传给 `tools(...)`：

```java
package com.example.web;

import com.example.tools.OrderService;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
public class ChatController {

    private final ChatClient chat;
    private final OrderService orders;

    public ChatController(ChatClient.Builder builder, OrderService orders) {
        this.chat = builder.build();
        this.orders = orders;
    }

    @GetMapping("/ask")
    public String ask(@RequestParam String q) {
        return chat.prompt()
                .user(q)
                .tools(orders)              // 关键：把整个 bean 丢进去
                .call()
                .content();
    }
}
```

`tools(Object... toolObjects)` 的语义在 `spring-ai-client-chat/.../ChatClient.java:215`：传入的对象会被反射扫描所有 `@Tool` 方法，包成 `MethodToolCallback` 注册进这一次调用。

跑起来 `curl 'localhost:8080/ask?q=帮我看看 ORD-2024-0001 的状态'`，模型会看到工具描述、自己决定调 `getOrder`，框架把结果回喂给模型，最终输出一段中文回复。整个 tool call loop 由框架管，controller 里没有任何"循环"代码——背后两条路径详见 [01-源码剖析/05-tool-calling](../01-源码剖析/05-tool-calling.md)。

## 多 tool 同时注册

`tools(...)` 是变长参数。再写两个 Bean：

```java
@Service
public class WeatherService {
    @Tool(description = "查询某城市当前天气，返回简短文本")
    public String currentWeather(
            @ToolParam(description = "城市名（中文），例如：上海") String city) {
        return "%s 当前晴，气温 22 度".formatted(city);
    }
}

@Service
public class CalculatorService {
    @Tool(description = "计算两数之和")
    public double add(@ToolParam(description = "加数") double a,
                      @ToolParam(description = "被加数") double b) {
        return a + b;
    }
}
```

controller 改成：

```java
return chat.prompt()
        .user(q)
        .tools(orders, weather, calculator)   // 同时挂三个
        .call()
        .content();
```

模型每轮会看到三个工具的 schema，自己决定调哪一个，甚至同一轮里平行调多个。问"帮我把订单 ORD-2024-0001 的金额加上 5"，它会先 `getOrder`，再 `add`，再合成最终答复——用户端只看到一次 HTTP 调用。

如果想让所有 controller 共享同一组工具，把它们设成 `defaultTools`：

```java
@Bean
ChatClient defaultChatClient(ChatClient.Builder builder,
                             OrderService o, WeatherService w, CalculatorService c) {
    return builder.defaultTools(o, w, c).build();
}
```

## 用 `ToolContext` 把业务上下文带进 tool

模型看不到也不该看到"当前是谁登录"。框架给了一个旁路：`ToolContext`。它是一个不可变 Map，由调用方塞进去、tool 方法以参数形式接收。源码 `spring-ai-model/.../chat/model/ToolContext.java:43-63`，本质就是 `Map<String,Object>` 的不可变包装。

修改 `OrderService`：

```java
@Tool(description = "查询当前登录用户的指定订单详情")
public String getOrderForCurrentUser(
        @ToolParam(description = "订单号") String orderId,
        ToolContext ctx) {                  // 框架自动注入，不会进入 schema
    String userId = (String) ctx.getContext().get("userId");
    if (userId == null) {
        return "{\"error\":\"未登录\"}";
    }
    // 用 userId + orderId 查 DB
    return """
           {"userId":"%s","orderId":"%s","status":"PAID"}
           """.formatted(userId, orderId);
}
```

调用端把 userId 写进 `toolContext`：

```java
@GetMapping("/ask")
public String ask(@RequestParam String q,
                  @RequestHeader("X-User-Id") String userId) {
    return chat.prompt()
            .user(q)
            .tools(orders)
            .toolContext(Map.of("userId", userId))   // 不会发给模型
            .call()
            .content();
}
```

关键事实：`ToolContext` 类型的参数会被框架在 schema 推导时跳过，模型不知道它的存在。这条路径让"业务身份"和"对话上下文"彻底解耦——模型只看业务参数，权限校验在 tool 方法里做。

## 函数式 tool：不写 Bean，直接挂 lambda

有些工具简单到不值得开一个 `@Service`。`FunctionToolCallback` 让你直接挂 lambda：

```java
import org.springframework.ai.tool.function.FunctionToolCallback;

record AddIn(double a, double b) {}

ToolCallback addCb = FunctionToolCallback
        .builder("add", (AddIn in, ToolContext ctx) -> in.a() + in.b())
        .description("计算两数之和")
        .inputType(AddIn.class)
        .build();

return chat.prompt()
        .user(q)
        .toolCallbacks(addCb)
        .call()
        .content();
```

`FunctionToolCallback.Builder.build()` 在 `spring-ai-model/.../tool/function/FunctionToolCallback.java:221-232` 自动用 `JsonSchemaGenerator.generateForType(inputType)` 推导 schema。`BiFunction<I, ToolContext, O>` 的二元签名意味着即便你不需要 ToolContext，也得在 lambda 形参里写出来（用 `_` 或忽略即可）。

什么时候用函数式：一次性脚本、动态拼出来的工具集、外部数据驱动的 tool 列表。它和 `@Tool` 混着用没问题——`tools(orders)` 配 `toolCallbacks(addCb)`，框架在 `MergeUtils` 里合并成一份注册表。

## tool 抛异常时会发生什么

默认情况下，tool 抛 `RuntimeException` 不会让整次 chat 调用失败：异常被框架捕获、转成字符串塞回模型，让模型决定下一步（重试、改换工具、向用户解释）。逻辑在 `spring-ai-model/.../tool/execution/DefaultToolExecutionExceptionProcessor.java:57-83`：

```java
if (this.alwaysThrow) {
    throw exception;
}
String message = exception.getMessage();
if (message == null || message.isBlank()) {
    message = "Exception occurred in tool: " + exception.getToolDefinition().name() + " ("
            + cause.getClass().getSimpleName() + ")";
}
return message;          // 这串字符串会作为 tool 结果回灌给模型
```

如果业务里希望某些异常直接冒泡（比如鉴权失败要打断对话），可以注册一个自己的处理器：

```java
@Bean
ToolExecutionExceptionProcessor toolExceptionProcessor() {
    return DefaultToolExecutionExceptionProcessor.builder()
            .alwaysThrow(false)
            .rethrowExceptions(List.of(SecurityException.class))
            .build();
}
```

`rethrowExceptions` 里的类会原样抛出（见同文件 60-65 行的 allowlist 判断），其他还是走"转字符串回喂模型"的路径。生产环境通常会把这个处理器和 `@RestControllerAdvice` 串起来，让鉴权类异常变成 401。

## 踩坑

**description 写太短，模型不会调。** Schema 决定模型是否选这把工具——`@Tool(description = "查订单")` 不如 `@Tool(description = "根据订单号查询订单详情，返回包含状态、金额、商品数的 JSON")`。把"输入是什么"和"输出是什么"都写进去，胜过写 prompt。

**参数 schema 推导失败。** `@ToolParam` 的 Java 形参用基本类型 + 简单 record/POJO。如果用 `Map<String, Object>` 或 `?`，`JsonSchemaGenerator` 推不出有用 schema，模型会拿到空对象。把入参定义成显式 record 或 POJO。

**循环不收敛。** 模型可能反复调同一个 tool。两类常见原因：tool 返回值不稳定（每次返回的随机 ID 让模型以为没拿对）、description 描述了"幂等"但实际有副作用。设个保护：在 ChatOptions 上限制 `maxIterations`（具体字段名见 `ToolCallingChatOptions`），或者用 `@Tool(returnDirect = true)` 让某些 tool 的返回值直接作为最终答复，跳过下一轮模型调用。

**`ToolContext` 数据被序列化进 prompt？** 不会。框架在 schema 推导时显式跳过 `ToolContext` 类型的形参，模型既看不到这个参数也不知道它存在。如果你不放心，把敏感字段命名带 `_internal` 之类前缀做防御。

**函数式 tool 必须加 `inputType()`。** `FunctionToolCallback.Builder.build()` 里 `Assert.notNull(this.inputType, "inputType cannot be null")`——光给名字和 lambda 是不够的，必须告诉框架"参数是哪个 record"，schema 才能推导出来。

## 关键代码索引

- 注解：`spring-ai-model/src/main/java/org/springframework/ai/tool/annotation/Tool.java:37-71`、`ToolParam.java:34-46`
- ChatClient 入口：`spring-ai-client-chat/.../ChatClient.java:213-223`（`tools / toolCallbacks / toolContext / toolNames`）
- 反射扫 `@Tool` 方法：`spring-ai-model/.../tool/method/MethodToolCallback.java`、`MethodToolCallbackProvider.java`
- 函数式 tool：`spring-ai-model/.../tool/function/FunctionToolCallback.java:51-232`
- ToolContext：`spring-ai-model/.../chat/model/ToolContext.java:43-63`
- 异常处理：`spring-ai-model/.../tool/execution/DefaultToolExecutionExceptionProcessor.java:57-83`

## 思考题

1. `tools(Object...)` 接的是任意对象。如果一个 Bean 里既有 `@Tool` 方法也有 `@Transactional` 方法，事务还会生效吗？为什么（提示：方法是直接被反射调用，还是通过 Spring 代理）？
2. 同一次调用里挂了同名工具（`@Tool(name = "search")` 出现两次），框架会怎么处理？查 `MethodToolCallbackProvider` 的实现。
3. 想限制每次对话最多调用 3 次工具，避免恶意 prompt 把模型卡进死循环——应该在 ChatOptions 还是 advisor 层做？两条路径下表现一样吗？

## 延伸阅读

- 想知道两条 tool 循环路径的差异：[01-源码剖析/05-tool-calling](../01-源码剖析/05-tool-calling.md)
- 想用 MCP server 提供的工具替代手写 `@Tool`：[02-实践用法/08-mcp](./08-mcp.md)
- 想给 tool 调用加日志/限流/审计：[01-源码剖析/04-advisor-链](../01-源码剖析/04-advisor-链.md)

> 基于 spring-ai commit 9cde97c1
