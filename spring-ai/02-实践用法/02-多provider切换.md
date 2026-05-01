# 实践 2：多 Provider 切换

读完这一篇，你的 Spring Boot 工程能同时连接 OpenAI 和 Anthropic（或本地 Ollama），通过 `@Qualifier` 拿到不同的 `ChatClient`，并能在请求里按需选 Provider。**这一篇会重点讲一个坑**：两个 starter 一起加的时候，默认装配会冲突——Spring AI 不像 JDBC 那样允许"同类多实例"，要让两家共存得手动出手。

## 一、核心问题：默认装配只允许一个 Chat Provider

打开 `OpenAiChatAutoConfiguration` 与 `AnthropicChatAutoConfiguration` 的类头：

```java
// OpenAiChatAutoConfiguration.java:50-54
@AutoConfiguration
@ConditionalOnProperty(name = SpringAIModelProperties.CHAT_MODEL,
    havingValue = SpringAIModels.OPENAI, matchIfMissing = true)
public class OpenAiChatAutoConfiguration { ... }
```

```java
// AnthropicChatAutoConfiguration.java:46-49
@AutoConfiguration
@ConditionalOnProperty(name = SpringAIModelProperties.CHAT_MODEL,
    havingValue = SpringAIModels.ANTHROPIC, matchIfMissing = true)
public class AnthropicChatAutoConfiguration { ... }
```

两边都用 `spring.ai.model.chat` 这一个 key 做开关，且都把 `matchIfMissing = true` 设上了。这意味着：

- 只引一个 starter 时，`spring.ai.model.chat` 不设也行，对应的 `ChatModel` bean 自动出现
- 两个 starter 都引、`spring.ai.model.chat` 不设时，**两个 `ChatModel` 都会被创建**
- 然后 `ChatClientAutoConfiguration` 想注入 `ChatModel` 时就会抛 `NoUniqueBeanDefinitionException`

所以多 Provider 共存的第一件事，是把这个全局开关关掉，自己来装配。

## 二、依赖

`pom.xml` 同时引两个 starter：

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
  </dependency>
</dependencies>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>2.0.0-SNAPSHOT</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

如果想加本地 Ollama 当第三家，再加 `spring-ai-starter-model-ollama`，写法对称，下面省略。

## 三、配置：把全局开关关掉，再分别配 key 与默认模型

`application.yml`：

```yaml
spring:
  ai:
    model:
      chat: none   # 关键：禁用默认 ChatModel 装配，自己来
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          temperature: 0.7
```

`spring.ai.model.chat: none` 让 OpenAI / Anthropic 的 ChatModel auto-config 都不触发；`spring.ai.openai.*` 与 `spring.ai.anthropic.*` 的连接信息照样会被各自的 `*ConnectionProperties` 与 `*ChatProperties` 读取——这两个 `@ConfigurationProperties` 不挂在 `@ConditionalOnProperty` 上，关掉装配也不影响它们绑定。

## 四、自己声明两个 ChatModel + 两个 ChatClient

写一个 `@Configuration`：

```java
@Configuration
public class MultiProviderConfig {

    @Bean
    OpenAiChatModel openAiChatModel(OpenAiConnectionProperties conn,
                                    OpenAiChatProperties chat,
                                    ToolCallingManager toolCallingManager) {
        var resolved = OpenAiAutoConfigurationUtil.resolveConnectionProperties(conn, chat);
        OpenAIClient client = OpenAiSetup.setupSyncClient(
            resolved.getBaseUrl(), resolved.getApiKey(), resolved.getCredential(),
            null, null, resolved.getOrganizationId(),
            false, false, resolved.getModel(), resolved.getTimeout(),
            resolved.getMaxRetries(), resolved.getProxy(), resolved.getCustomHeaders());
        OpenAIClientAsync asyncClient = OpenAiSetup.setupAsyncClient(/* 同上参数 */);
        return OpenAiChatModel.builder()
            .openAiClient(client)
            .openAiClientAsync(asyncClient)
            .options(chat.getOptions())
            .toolCallingManager(toolCallingManager)
            .build();
    }

    @Bean
    AnthropicChatModel anthropicChatModel(AnthropicConnectionProperties conn,
                                          AnthropicChatProperties chat,
                                          ToolCallingManager toolCallingManager) {
        AnthropicApi api = AnthropicApi.builder()
            .apiKey(conn.getApiKey())
            .baseUrl(conn.getBaseUrl())
            .build();
        return AnthropicChatModel.builder()
            .anthropicApi(api)
            .defaultOptions(chat.getOptions())
            .toolCallingManager(toolCallingManager)
            .build();
    }

    @Bean
    @Qualifier("openai")
    ChatClient openAiChatClient(OpenAiChatModel model) {
        return ChatClient.builder(model).build();
    }

    @Bean
    @Qualifier("anthropic")
    ChatClient anthropicChatClient(AnthropicChatModel model) {
        return ChatClient.builder(model).build();
    }
}
```

> 这里直接 `ChatClient.builder(model)`，没用 `ChatClient.Builder` 注入。原因：`ChatClientAutoConfiguration` 提供的 `ChatClient.Builder` 是单 `ChatModel` 注入的版本（见 `ChatClientAutoConfiguration.java:91-99`），多 Provider 场景下手动构造更直接。如果你需要 `ChatClientCustomizer` 链上的统一加工，把 `ChatClientBuilderConfigurer` 注进来在 `build()` 前 `configure()` 一次即可。

`Controller` 里按 qualifier 拿：

```java
@RestController
class ChatController {

    @Autowired @Qualifier("openai")     ChatClient openAi;
    @Autowired @Qualifier("anthropic")  ChatClient anthropic;

    @GetMapping("/openai")
    String fromOpenAi(@RequestParam String q) {
        return openAi.prompt(q).call().content();
    }

    @GetMapping("/anthropic")
    String fromAnthropic(@RequestParam String q) {
        return anthropic.prompt(q).call().content();
    }
}
```

## 五、统一抽象层：用 Map 做"按名字切换"

两个 endpoint 区别只在 qualifier，复制粘贴会越写越乱。Spring 的 `Map<String, T>` 注入正好用得上：

```java
@Service
class ChatRouter {

    private final Map<String, ChatClient> clients;

    ChatRouter(Map<String, ChatClient> clients) {
        this.clients = clients;
    }

    String ask(String provider, String prompt) {
        ChatClient client = clients.get(provider + "ChatClient");
        if (client == null) {
            throw new IllegalArgumentException("unknown provider: " + provider);
        }
        return client.prompt(prompt).call().content();
    }
}
```

Spring 把所有 `ChatClient` bean 装成 `Map<beanName, instance>` 注入。bean 名是 `@Bean` 方法名（`openAiChatClient` / `anthropicChatClient`），所以查的时候用 `provider + "ChatClient"` 拼。

如果想要更体面的 key（直接 `"openai"` / `"anthropic"`），把方法名改成 `openai` / `anthropic`，或者显式 `@Bean("openai")`。一般来说显式起名最稳：

```java
@Bean("openai")
ChatClient openAiChatClient(OpenAiChatModel m) { return ChatClient.builder(m).build(); }

@Bean("anthropic")
ChatClient anthropicChatClient(AnthropicChatModel m) { return ChatClient.builder(m).build(); }
```

`Controller` 现在简化成一个 endpoint：

```java
@GetMapping("/chat/{provider}")
String chat(@PathVariable String provider, @RequestParam String q) {
    return router.ask(provider, q);
}
```

## 六、Provider 特有 options 怎么传

`ChatOptions` 通用层只有 `temperature` / `topP` / `maxTokens` / `stopSequences` 等几个跨 Provider 字段。Provider 各家私有的（OpenAI 的 `logitBias`、`responseFormat`，Anthropic 的 `thinking` 配置）需要用具体子类的 builder：

```java
// OpenAI 私有字段：logitBias
String result = openAi.prompt("写一段诗")
    .options(OpenAiChatOptions.builder()
        .model("gpt-4o")
        .temperature(0.9)
        .logitBias(Map.of("50256", -100))
        .build())
    .call().content();

// Anthropic 私有字段：thinking
String thoughts = anthropic.prompt("分析这道数学题：...")
    .options(AnthropicChatOptions.builder()
        .model("claude-opus-4-1-20250805")
        .maxTokens(8000)
        .thinking(ChatCompletionRequest.ThinkingConfig.builder()
            .type("enabled")
            .budgetTokens(2000)
            .build())
        .build())
    .call().content();
```

per-call 选项会和默认选项做 `combineWith` 合并（非 null 优先）—— 默认 `temperature=0.7` + per-call `temperature=0.9` 最终走 0.9。这套合并逻辑在 part 1 [03-chatmodel-options](../01-源码剖析/03-chatmodel-options.md) 第四节有详细推演。

如果你的统一抽象层想让上层"无视 Provider"地传选项，**只能传通用层字段**——任何想用 `logitBias` 的调用都要绕过 router 直接拿 OpenAI client。这是抽象的代价。

## 七、踩坑清单

**1. 不设 `spring.ai.model.chat`，两边都装配，启动炸 `NoUniqueBeanDefinitionException`**

报错点是 `ChatClientAutoConfiguration.chatClientBuilder` 里的 `ChatModel` 参数：找到 `openAiChatModel` 和 `anthropicChatModel` 两个 bean 不知道注哪个。解决就是上面的 `chat: none` + 手工装配。

**2. 设了 `chat: openai`，Anthropic 的 `ChatModel` 也注不出来**

`@ConditionalOnProperty` 是 OR 不了的——值只能是 `openai` 之一。所以单设一边就只剩一边。两边都要的话只能 `none` + 手工。

**3. `@Bean` 名字和 `@Qualifier` 不一致导致 Map 注入查不到**

bean 名字默认是 `@Bean` 方法名；`@Qualifier("openai")` 改的是注入点筛选，不改 bean 名。`Map<String, ChatClient>` 的 key 永远是 bean name。所以"显式起名"最干脆：`@Bean("openai")`。

**4. 默认配置（`application.yml` 的 `spring.ai.openai.chat.options.model`）在手工装配时会"看起来失效"**

实际上 `chatProperties.getOptions()` 还是被读到了；如果你 `OpenAiChatModel.builder().options(chat.getOptions())` 漏掉这一行，默认 model/temperature 就丢了。检查一下你手工 `@Bean` 是否把 `xxxChatProperties` 注进来再喂给 builder。

**5. 多 Provider 都启用 streaming + observability 时 span 命名不冲突，但日志会乱**

每个 ChatModel 自己产 ChatModel span，name 一致（`spring.ai.chat.client`），靠 `tags.gen_ai.system=openai|anthropic` 区分。Grafana 面板写的时候记得按这个 tag 切，不然两家流量混在一起。

## 八、完整最小工程结构

```
src/main/java/com/example/multi/
  Application.java                    // @SpringBootApplication
  config/MultiProviderConfig.java     // 上面那块
  controller/ChatController.java      // /chat/{provider}
  service/ChatRouter.java             // Map 路由
src/main/resources/
  application.yml                     // chat: none + 各家配置
```

把 `OPENAI_API_KEY` 和 `ANTHROPIC_API_KEY` 设到环境变量，`./mvnw spring-boot:run`，然后：

```bash
curl 'http://localhost:8080/chat/openai?q=用一句话介绍 Spring AI'
curl 'http://localhost:8080/chat/anthropic?q=用一句话介绍 Spring AI'
```

两条会从不同 Provider 回来。

## 想知道这背后怎么实现的

- **为什么 `ChatOptions` 拆通用层 + 子类扩展，per-call 怎么覆盖默认**：见 [01-源码剖析/03-chatmodel-options.md](../01-源码剖析/03-chatmodel-options.md)
- **`ChatClient.Builder` 为什么是 prototype scope，单一个 ChatModel 的限制怎么来的**：见 [01-源码剖析/09-boot-观测.md](../01-源码剖析/09-boot-观测.md)

> 基于 spring-ai commit 9cde97c1
