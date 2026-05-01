# 实践 3：ChatMemory 多轮 + 持久化

读完这篇你能跑出：一个支持多用户、多轮、可恢复的 `/chat` 接口——同一个 `conversationId` 的请求能记住上下文，重启后从数据库捞回历史，窗口大小可控不爆 token。

底层用的是 `MessageWindowChatMemory` + `MessageChatMemoryAdvisor`；存储先用内存版起步，再切到 JDBC。

## 前置依赖

`pom.xml`：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>

<!-- 内存版只需要 spring-ai-starter-model-chat-memory（按需）；
     这里直接上 JDBC 版，starter 会自动把 InMemory 替换掉 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
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
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always   # 让 starter 帮你建表
  datasource:
    url: jdbc:postgresql://localhost:5432/chatdemo
    username: chat
    password: chat
```

> 想先纯内存跑？把上面 `repository-jdbc` 那段依赖去掉，并删掉 `spring.datasource` 与 `spring.ai.chat.memory.repository.jdbc` 配置，`ChatMemoryAutoConfiguration` 会自动落到 `InMemoryChatMemoryRepository` 上（见 `auto-configurations/.../ChatMemoryAutoConfiguration.java:36-48`）。

## 第一版：内存起步

最薄的写法是不显式声明 `ChatMemory` bean，直接靠 autoconfig 提供的默认实现，然后在 `ChatClient.Builder` 阶段注入 advisor：

```java
@Configuration
class ChatConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            .defaultSystem("You are a helpful assistant. Be concise.")
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
            .build();
    }
}
```

注入到这里的 `ChatMemory` 就是 autoconfig 给的 `MessageWindowChatMemory`，默认窗口 20 条（见 `MessageWindowChatMemory.java:44`）。`MessageChatMemoryAdvisor` 只是把"取历史 → 拼到消息列表 → 把新一轮也写回去"封装成了一个 around advisor。

## REST 接口：用 conversationId 区分用户

advisor 怎么知道当前请求属于哪个会话？答案在 `BaseChatMemoryAdvisor.getConversationId`（`spring-ai-client-chat/.../advisor/api/BaseChatMemoryAdvisor.java:39-45`）：它从 `ChatClientRequest.context()` 里读 `ChatMemory.CONVERSATION_ID`（常量值 `"chat_memory_conversation_id"`），缺省回退到 `"default"`。

所以接口长这样：

```java
@RestController
@RequiredArgsConstructor
class ChatController {

    private final ChatClient chatClient;

    @PostMapping("/chat")
    String chat(@RequestParam String conversationId,
                @RequestBody String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

`a.param(...)` 把 conversationId 写到 advisor params（最终落到 `ChatClientRequest.context`）。每个用户用不同的 id，互不串台。

调一下：

```bash
curl -XPOST 'localhost:8080/chat?conversationId=alice' -d '我叫 Alice'
curl -XPOST 'localhost:8080/chat?conversationId=alice' -d '我刚才告诉你我叫什么？'
# → 它会答出 Alice

curl -XPOST 'localhost:8080/chat?conversationId=bob'   -d '我刚才告诉你我叫什么？'
# → 它不知道（bob 是新会话）
```

**忘了带 conversationId 会怎样**：所有请求都落到默认会话 `"default"`，多用户互相串扰。生产环境务必校验：

```java
if (!StringUtils.hasText(conversationId)) {
    throw new IllegalArgumentException("conversationId is required");
}
```

## 切到 JDBC：重启不丢历史

加上 `spring-ai-starter-model-chat-memory-repository-jdbc` 之后，`JdbcChatMemoryRepositoryAutoConfiguration` 会用配置的数据源把 `ChatMemoryRepository` bean 替换掉（条件优先级高于 `InMemoryChatMemoryRepository`）。`MessageWindowChatMemory` 因此自动改用 JDBC 后端，对应用代码零改动。

`initialize-schema: always` 让 starter 启动时跑 `schema-postgresql.sql`，建一张表（见 `memory/repository/.../resources/.../schema-postgresql.sql`）：

```sql
CREATE TABLE IF NOT EXISTS SPRING_AI_CHAT_MEMORY (
    conversation_id VARCHAR(36) NOT NULL,
    content TEXT NOT NULL,
    type VARCHAR(10) NOT NULL CHECK (type IN ('USER','ASSISTANT','SYSTEM','TOOL')),
    "timestamp" TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS SPRING_AI_CHAT_MEMORY_CONVERSATION_ID_TIMESTAMP_IDX
ON SPRING_AI_CHAT_MEMORY(conversation_id, "timestamp");
```

每条消息一行：`type` 区分 USER / ASSISTANT / SYSTEM / TOOL，`timestamp` 保留顺序。重启应用后再用 `conversationId=alice` 调，历史还在。

如果你的 DBA 不让应用建表，把 `initialize-schema` 改成 `never`，自己提前执行 SQL 即可。`@@platform@@` 占位符会按数据源类型自动替换（H2/MySQL/Postgres/Oracle/SqlServer/MariaDB/SQLite 都有内置 schema 文件）。

切 Redis、MongoDB、Cassandra 是同样的套路，换一个 `spring-ai-starter-model-chat-memory-repository-{xxx}` 即可，应用代码不用动。

## 控制窗口：别让 prompt 爆炸

默认窗口 20 条够大多数场景；如果对话很长或 token 预算紧，自己声明一个 `ChatMemory` bean 覆盖默认值：

```java
@Bean
ChatMemory chatMemory(ChatMemoryRepository repo) {
    return MessageWindowChatMemory.builder()
        .chatMemoryRepository(repo)
        .maxMessages(10)   // 只保留最近 10 条（USER + ASSISTANT 算 2 条）
        .build();
}
```

`MessageWindowChatMemory.process` 的关键行为（`MessageWindowChatMemory.java:80-112`）：

- 超出 `maxMessages` 时，**优先驱逐非 SystemMessage**。SystemMessage 永远保留，避免每轮都"忘了我是谁"
- 新加入的 SystemMessage 会顶替之前的 SystemMessage（同一会话只留一份系统提示）

如果你想让模型不带"原始消息列表"而是"摘要后的字符串"做记忆，换成 `PromptChatMemoryAdvisor`：

```java
.defaultAdvisors(PromptChatMemoryAdvisor.builder(chatMemory).build())
```

它把历史消息按 `USER:xxx\nASSISTANT:yyy` 格式拼成一段，注入到 system prompt 里（见 `PromptChatMemoryAdvisor.java:59-69` 的模板）。代价是历史不再以 message 形态进 prompt，模型对工具调用历史等结构化片段的感知会变差——一般场景仍推荐 `MessageChatMemoryAdvisor`，只有在 system prompt 集中管控、历史想要被"摘要化"时才换。

## 踩坑提示

1. **窗口卡得太紧**：`maxMessages=4` 看似省 token，但 USER + ASSISTANT 一来一回就是 2 条，第三轮就开始丢首条。建议 `>= 10`，敏感时配上 logger 把每次进 prompt 的消息打出来检查
2. **conversationId 透传断点**：advisor 是从 `ChatClientRequest.context` 读 id 的——如果你用了自定义 advisor 而它在 `mutate()` 时丢掉了 context，memory 就废了。`mutate()` 默认会保留 context，自己实现时检查这一步
3. **JDBC 表没建 / 列名冲突**：`SPRING_AI_CHAT_MEMORY` 这个表名是写死的，多个独立服务共用同一个 schema 时会互踩；要么各自给一套数据源，要么改用 conversationId 前缀做隔离
4. **跨 starter 切换不平滑**：从 InMemory 切到 JDBC 时，老历史不会自动迁移——准备一段一次性脚本，否则用户感觉"AI 失忆了"
5. **observation 里看不到 conversationId**：默认 `DefaultChatClientObservationConvention` 把 conversationId 标成 high-cardinality tag，需要在 Micrometer 里显式启用（见剖析篇 09）

## 关键代码索引

- `spring-ai-model/.../chat/memory/ChatMemory.java:33-38` —— `DEFAULT_CONVERSATION_ID` / `CONVERSATION_ID` 常量
- `spring-ai-model/.../chat/memory/MessageWindowChatMemory.java:42-143` —— 窗口逻辑
- `spring-ai-client-chat/.../advisor/MessageChatMemoryAdvisor.java:79-138` —— 注入历史 + 写回助手消息
- `spring-ai-client-chat/.../advisor/PromptChatMemoryAdvisor.java:59-130` —— 摘要式记忆注入
- `spring-ai-client-chat/.../advisor/api/BaseChatMemoryAdvisor.java:39-45` —— conversationId 解析
- `auto-configurations/.../ChatMemoryAutoConfiguration.java:36-48` —— 默认 bean 装配
- `memory/repository/.../resources/org/springframework/ai/chat/memory/repository/jdbc/schema-*.sql` —— 各数据库 schema

## 思考题

1. 如果你的业务有"管理员可以查看任意用户对话"的需求，`SPRING_AI_CHAT_MEMORY` 的表结构要补什么列？为什么 spring-ai 没有把 `userId` 直接做成一等字段？
2. `MessageWindowChatMemory` 删消息时不删 SystemMessage，但 SystemMessage 也算进 `maxMessages` 总数——极端情况下（连续切换 system prompt 又超窗口）会发生什么？
3. 改成 `PromptChatMemoryAdvisor` 之后，工具调用的 `ToolResponseMessage` 还会被记进上下文吗？为什么模板里只过滤 USER/ASSISTANT？

## 延伸阅读

- 想知道这背后怎么实现：[01-源码剖析/10-mcp-memory.md](../01-源码剖析/10-mcp-memory.md) 的"ChatMemory 也只是 advisor"那一节
- advisor 是怎么挂上链的、为什么用 deque：[01-源码剖析/04-advisor-链.md](../01-源码剖析/04-advisor-链.md)
- conversationId 是怎么穿过 observation 链的：[01-源码剖析/09-boot-观测.md](../01-源码剖析/09-boot-观测.md)

> 基于 spring-ai commit 9cde97c1
