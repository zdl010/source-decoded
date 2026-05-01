# Spring AI 源码解读

基于 spring-ai commit `9cde97c1`（`2.0.0-SNAPSHOT`）。

分两部分：

- **[01-源码剖析](./01-源码剖析/)** —— 拆模块、拆抽象、拆调用链，回答"为什么这样设计"
- **[02-实践用法](./02-实践用法/)** —— 拿来就能跑的最小可运行示例，回答"怎么用"

两部分互相引用：实践篇遇到的设计选择会指回剖析篇对应章节，剖析篇的"怎么用得上"会指向实践篇。

## 01 源码剖析（10 篇）

| # | 标题 | 重点 |
| --- | --- | --- |
| 01 | [模块地图](./01-源码剖析/01-模块地图.md) | 抽象层 vs 集成层、依赖图 |
| 02 | [调用旅程](./01-源码剖析/02-调用旅程.md) | 一次 ChatClient 调用的完整泳道 |
| 03 | [ChatModel & ChatOptions](./01-源码剖析/03-chatmodel-options.md) | 跨 Provider 的最小契约 |
| 04 | [Advisor 链](./01-源码剖析/04-advisor-链.md) | Spring AI 的总线（主章） |
| 05 | [Tool Calling 双路径](./01-源码剖析/05-tool-calling.md) | provider 内部循环 vs advisor 链循环 |
| 06 | [结构化输出](./01-源码剖析/06-结构化输出.md) | 把 schema 塞进链里 |
| 07 | [Modular RAG](./01-源码剖析/07-modular-rag.md) | 七阶段流水线 |
| 08 | [VectorStore & Filter IR](./01-源码剖析/08-vectorstore-filter.md) | 便携 metadata 过滤的代价 |
| 09 | [Boot 整合 + 观测](./01-源码剖析/09-boot-观测.md) | autoconfig / starter / 多层 span |
| 10 | [MCP & ChatMemory](./01-源码剖析/10-mcp-memory.md) | 把外部能力转成既有抽象 |

## 02 实践用法（8 篇 · 编写中）

| # | 标题 | 你能跑出什么 |
| --- | --- | --- |
| 01 | 五分钟跑通第一个 ChatClient | 一个 Spring Boot REST 接口接 OpenAI / Ollama |
| 02 | 多 Provider 切换 | 同一应用里 OpenAI / Anthropic / Ollama 共存 |
| 03 | ChatMemory 多轮 + 持久化 | 多用户、可恢复的多轮对话 |
| 04 | 流式输出 SSE | `Flux<ChatResponse>` → 前端边流边渲染 |
| 05 | RAG 从 PDF 到问答 | 一个能用 PDF 知识库回答问题的接口 |
| 06 | Tool Calling 实战 | 让 AI 调用你的 Spring Bean |
| 07 | 结构化输出 Java DTO | `entity(MyDto.class)` 直接拿到对象 |
| 08 | MCP 实战 | 连一个外部 MCP Server，让模型用第三方工具 |
