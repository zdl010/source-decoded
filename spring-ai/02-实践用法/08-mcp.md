# 实践 8：MCP 实战 —— 让模型用上外部 MCP Server 的工具

读完这一篇，你能做到：

- 用 `spring-ai-starter-mcp-client` 连一个外部 MCP Server（这里以官方 `@modelcontextprotocol/server-filesystem` 为例）
- 让 MCP Server 提供的工具自动注入 `ChatClient`，模型主动调它们
- 同时连多个 MCP Server，并用 `McpToolNamePrefixGenerator` 给工具加前缀避免命名冲突
- 用 `McpToolFilter` 屏蔽某些不想暴露的工具

整篇就是一个最小可运行的 Spring Boot 工程，跑起来之后让模型读你硬盘里的一个文件。

## 一、前置依赖

仓库里 MCP 客户端有两个 starter：

- `spring-ai-starter-mcp-client` —— 同步、基于 JDK `HttpClient`，最常用
- `spring-ai-starter-mcp-client-webflux` —— 基于 WebFlux，需要响应式栈

本文用同步版。`pom.xml`：

```xml
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

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
  </dependency>
</dependencies>
```

`mcp-client` starter 通过 `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common` 引入了核心 autoconfig，无须手工写任何配置类。它做的事情可以在 `McpToolCallbackAutoConfiguration.java:48-86` 看到：把所有可见的 `McpSyncClient` 收进来，构造一个 `SyncMcpToolCallbackProvider` 注册成 bean。

## 二、启动一个外部 MCP Server

最快的姿势是用 npx 起官方的 filesystem server。这个 server 提供 `read_file` / `write_file` / `list_directory` 等工具，让模型能读写本地文件。

```bash
# 开两个终端：一个跑 server，一个跑 Spring Boot 应用
mkdir -p /tmp/mcp-playground
echo "Spring AI 是一个 Java 生态的 AI 应用框架。" > /tmp/mcp-playground/notes.txt

# 直接跑会读 stdio，所以这里只是验证依赖是否能装
npx -y @modelcontextprotocol/server-filesystem /tmp/mcp-playground
```

实际上 **stdio 传输模式下不需要你提前手动起 server**，Spring AI 会作为父进程把上面这条命令拉起，stdin/stdout 直接接管成 MCP 通道。下一节 yml 里直接写命令就行。

## 三、配置 MCP 客户端：stdio 连接

`application.yml`：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
    mcp:
      client:
        enabled: true
        name: spring-ai-mcp-client
        type: SYNC
        request-timeout: 30s
        stdio:
          connections:
            filesystem:
              command: npx
              args:
                - -y
                - "@modelcontextprotocol/server-filesystem"
                - /tmp/mcp-playground
```

几个关键 key 来自 `McpClientCommonProperties.java:34`（`spring.ai.mcp.client`）和 `McpStdioClientProperties.java:48`（`spring.ai.mcp.client.stdio`）：

- `type: SYNC` —— 默认就是同步，对应注入 `McpSyncClient`
- `stdio.connections.<名字>` —— 一个 server 一个名字；同名只能一个，名字是后面前缀生成器会用到的标识

如果 server 是远程 SSE 模式，把 `stdio` 换成 `sse` 即可：

```yaml
spring:
  ai:
    mcp:
      client:
        sse:
          connections:
            mcp-hub:
              url: http://localhost:3000
              sse-endpoint: /mcp/sse
```

`McpSseClientProperties.java:34-55` 注释里特别强调一点：URL 和 endpoint 要拆开写，**不能把整个 URL 塞进 `url`**——很多人第一次踩这个坑。

## 四、自动注入 tool 到 ChatClient

`McpToolCallbackAutoConfiguration` 已经把所有 MCP 工具打包成一个 `SyncMcpToolCallbackProvider` bean。控制器里把它注进 `ChatClient`：

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder,
                          SyncMcpToolCallbackProvider mcpTools) {
        this.chatClient = builder
            .defaultSystem("你是一个帮忙读取本地文件的助手。")
            .defaultToolCallbacks(mcpTools)
            .build();
    }

    @GetMapping
    public String ask(@RequestParam String q) {
        return chatClient.prompt()
            .user(q)
            .call()
            .content();
    }
}
```

启动应用，发请求：

```bash
curl 'http://localhost:8080/chat?q=请读取 /tmp/mcp-playground/notes.txt 并总结一句话'
```

会看到模型先调用 `read_file`，拿到文件内容后再回话。如果想确认模型确实走了 tool 路径，把日志开到 `DEBUG`：

```yaml
logging:
  level:
    org.springframework.ai.mcp: DEBUG
    io.modelcontextprotocol: DEBUG
```

## 五、连多个 MCP Server：用前缀避免命名冲突

把 yml 加一个 git server：

```yaml
spring:
  ai:
    mcp:
      client:
        stdio:
          connections:
            filesystem:
              command: npx
              args: [-y, "@modelcontextprotocol/server-filesystem", /tmp/mcp-playground]
            git:
              command: npx
              args: [-y, "@modelcontextprotocol/server-git",
                     "--repository", /tmp/mcp-playground]
```

两个 server 都可能叫 `read_file` 之类的同名 tool。框架默认装的是 `DefaultMcpToolNamePrefixGenerator`（`McpToolCallbackAutoConfiguration.java:52-54`）——它**只在重名时**给后来者加 `alt_<n>_` 前缀（见 `DefaultMcpToolNamePrefixGenerator.java:62-77`）。这个策略对调试不友好：你不知道哪个工具来自哪个 server。

更可读的姿势：用 server 名做静态前缀。自定义一个 bean 直接覆盖默认：

```java
@Configuration
class McpConfig {

    @Bean
    McpToolNamePrefixGenerator serverScopedPrefix() {
        return (info, tool) -> {
            String serverName = info.initializeResult() != null
                ? info.initializeResult().serverInfo().name()
                : "unknown";
            return serverName.replaceAll("[^a-zA-Z0-9_]", "_") + "_" + tool.name();
        };
    }
}
```

工具名变成 `filesystem_read_file` / `git_read_file`，模型在 prompt 里看见就知道选哪个。注意接口签名是 `String prefixedToolName(McpConnectionInfo, Tool)`，参数都来自 `McpToolNamePrefixGenerator.java:39-49`。

## 六、屏蔽不想暴露的 tool

`McpToolFilter` 是一个 `BiPredicate<McpConnectionInfo, McpSchema.Tool>`（`McpToolFilter.java:30-32`），返回 `false` 就把这个工具从最终的 `ToolCallback` 列表里剔除。比如不想让模型动 `write_file`：

```java
@Bean
McpToolFilter readOnlyFilter() {
    Set<String> blocked = Set.of("write_file", "edit_file", "create_directory");
    return (info, tool) -> !blocked.contains(tool.name());
}
```

`McpToolCallbackAutoConfiguration.java:71-72` 里通过 `ObjectProvider<McpToolFilter>` 注入这个 bean，整个 provider 在构造工具列表时就会调用它——你不需要重启 server，**重启应用即可**生效。

如果连 prefix + filter 一起精细控制，直接覆盖整个 provider bean：

```java
@Bean
@Primary
SyncMcpToolCallbackProvider customMcpProvider(List<McpSyncClient> clients) {
    return SyncMcpToolCallbackProvider.builder()
        .mcpClients(clients)
        .toolNamePrefixGenerator(serverScopedPrefix())
        .toolFilter(readOnlyFilter())
        .build();
}
```

builder 的入口在 `SyncMcpToolCallbackProvider.java:210-212`，所有定制都从这里走。

## 七、跑通的样子

最终调用一次：

```bash
curl 'http://localhost:8080/chat?q=列出 /tmp/mcp-playground 下的所有文件，并把 notes.txt 的内容读出来总结'
```

观察日志会看到大致流程：

1. `McpClientAutoConfiguration` 启动时把 `npx ... server-filesystem` 拉起，建立 stdio 通道
2. `SyncMcpToolCallbackProvider` 调 `mcpClient.listTools()` 拿到工具清单，加前缀、过滤后缓存
3. `ChatClient` 把 `ToolCallback` 转成 OpenAI 的 `tools` 参数发给模型
4. 模型返回 `tool_calls`，`ToolCallingManager` 反向调用 `SyncMcpToolCallback.call`，再走回 `mcpClient.callTool`
5. 拿到工具结果作为新的 `ToolResponseMessage` 拼回去再请求一次
6. 模型给出最终回答

第 4–5 步的循环就是 part 1 第 5 篇讲的"双路径"中，被 advisor 链/或 ChatModel 内部接管的那段。MCP 工具走的就是普通 `ToolCallback` 通道——MCP 不是新概念。

## 八、踩坑清单

- **`npx` 找不到**：CI/容器环境里 PATH 可能没有 npx。要么换成 `command: /usr/local/bin/npx`，要么在镜像里预装 `@modelcontextprotocol/server-filesystem` 然后用 `node` 直接跑入口脚本
- **`stdio` 用错版本**：MCP server 协议有版本，`type: SYNC` 默认对接较旧的同步语义。如果 server 强制要异步推送（比如长任务），客户端 `type` 要切到 `ASYNC` 并改用 `spring-ai-starter-mcp-client-webflux`
- **SSE 把整 URL 塞进 url**：`url` 只填到 host:port，路径要拆到 `sse-endpoint`。塞错了会连接 404，错误信息又不够明显
- **prefix 不生效**：默认 `DefaultMcpToolNamePrefixGenerator` 只在**重名时**才加 `alt_n_`。想要"每个 server 一律加前缀"必须自己提供 `McpToolNamePrefixGenerator` bean
- **filter 没生效**：检查 bean 唯一性，`ObjectProvider#getIfUnique` 在容器里有多个 `McpToolFilter` 时会回退到默认（全通过）。需要时用 `@Primary` 或 `BiPredicate` 组合：`f1.and(f2)`
- **request-timeout 太短**：默认 20s，对 npm 启动 server 这种冷启动场景很可能超时（首次 `npx -y` 要拉包）。`request-timeout: 60s` 或更长比较稳
- **Tool 数量爆炸打满 context**：一个 MCP server 可能注册十几个 tool，模型 prompt 体积会被工具描述撑大。用 `McpToolFilter` 把不需要的 tool 关掉是性价比最高的办法
- **关闭 tool callback 自动装配**：如果你想自己管 provider，关掉自动那份：`spring.ai.mcp.client.toolcallback.enabled=false`（开关位置在 `McpToolCallbackAutoConfiguration.java:116-117`）

## 关键代码索引

- `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/.../McpClientAutoConfiguration.java` —— sync/async client 装配入口
- `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/.../McpToolCallbackAutoConfiguration.java:48-86` —— `SyncMcpToolCallbackProvider` 默认装配
- `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/.../properties/McpClientCommonProperties.java:34` —— `spring.ai.mcp.client.*` 配置项
- `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/.../properties/McpStdioClientProperties.java:48` —— stdio 连接配置
- `auto-configurations/mcp/spring-ai-autoconfigure-mcp-client-common/.../properties/McpSseClientProperties.java:64` —— sse 连接配置
- `mcp/common/.../SyncMcpToolCallbackProvider.java:210-212` —— provider builder 入口
- `mcp/common/.../McpToolNamePrefixGenerator.java:39-49` —— 前缀生成器接口
- `mcp/common/.../DefaultMcpToolNamePrefixGenerator.java:62-77` —— 默认实现的去重策略
- `mcp/common/.../McpToolFilter.java:30-32` —— 工具过滤器接口

## 思考题

1. 默认前缀生成器只在"重名时"才加前缀，这种"惰性"策略对调试和审计有什么不利？如果让你重新设计默认行为，你会选哪种策略，理由是什么？
2. 同一个 MCP server 在重启后 tool 列表可能变化（server 端动态加载工具），框架靠 `McpToolsChangedEvent`（`SyncMcpToolCallbackProvider.java:165-167`）失效缓存。这个事件如果在请求处理过程中到达，会发生什么？要不要做幂等保护？
3. 如果你要把企业内部一组现有 REST API 包成 MCP server 暴露给模型，你倾向直接写 MCP server（`spring-ai-starter-mcp-server`），还是直接用 `@Tool` 注解（part 1 第 5 篇）？两种方案在维护成本、安全边界、跨语言可用性上差在哪里？

## 延伸阅读

- 想知道"MCP tool 为什么能直接当成 ToolCallback 用"：[01-源码剖析/10-mcp-memory.md](../01-源码剖析/10-mcp-memory.md)
- 想知道 ChatModel 拿到 tool 后是怎么循环调度的：[01-源码剖析/05-tool-calling.md](../01-源码剖析/05-tool-calling.md)
- 想知道整个 ChatClient 调用如何被 advisor 链织起来：[01-源码剖析/04-advisor-链.md](../01-源码剖析/04-advisor-链.md)
- 官方 MCP server 仓库：`github.com/modelcontextprotocol/servers`，里面除了 filesystem、git，还有 sqlite、puppeteer、fetch 等

> 基于 spring-ai commit 9cde97c1
