# RAG 实战：从 PDF 到问答

读完这一篇，你会得到一个能用 PDF 当知识库的问答接口：本地 ingest 一份 PDF，模型回答时会先从向量库里取最相关的几段，再据此作答。先用内存版 `SimpleVectorStore` 跑通，再切到 PgVector 持久化；最后把"一锤子" `QuestionAnswerAdvisor` 升级成"流水线" `RetrievalAugmentationAdvisor`。

整篇围绕一个最小工程：一个 Spring Boot 服务，含一段 ingest 代码 + 一个查询接口 + 一份 docker-compose。

## 前置依赖

JDK 17+，Spring Boot 3.x（Spring AI `2.0.0-SNAPSHOT` 也在 4.x 上构建，但 3.x 工程也能跑）。最小 pom：

```xml
<dependencies>
    <!-- 模型：chat + embedding 都来自 OpenAI -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <!-- PDF 解析（不带 starter，直接引核心 lib） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>

    <!-- 内存版向量库：在 spring-ai-vector-store 里，被 starter-model-openai 传递引入 -->

    <!-- web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
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
      embedding:
        options:
          model: text-embedding-3-small

rag:
  pdf-path: classpath:/docs/spring-ai-overview.pdf
```

注意 `chat` 和 `embedding` 是两套独立配置：模型不同、token 计费方式也不同，常被混淆。

## Ingest：把 PDF 喂进向量库

`PagePdfDocumentReader`（`document-readers/pdf-reader/.../PagePdfDocumentReader.java:51-80`）按页切，把每页变成一个 `Document`，metadata 自动带 `page_number` / `file_name`。然后用 `TokenTextSplitter`（`spring-ai-commons/.../transformer/splitter/TokenTextSplitter.java`）按 token 数二次切片，避免单块超出 embedding 上下文。

注册 `SimpleVectorStore` 作为 bean，然后 ingest：

```java
@Configuration
class VectorStoreConfig {

    @Bean
    SimpleVectorStore simpleVectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}

@Component
@RequiredArgsConstructor
class PdfIngester implements ApplicationRunner {

    private final SimpleVectorStore vectorStore;

    @Value("${rag.pdf-path}")
    private Resource pdfResource;

    @Override
    public void run(ApplicationArguments args) {
        var reader = new PagePdfDocumentReader(pdfResource,
            PdfDocumentReaderConfig.builder()
                .withPageTopMargin(0)
                .withPageBottomMargin(0)
                .build());

        // 1 page → 1 Document，先不切；下一步用 splitter 二次切
        List<Document> pages = reader.read();

        var splitter = new TokenTextSplitter(/* chunkSize */ 500, /* minChunkSize */ 100,
                /* minChunkLength */ 5, /* maxNumChunks */ 10000, /* keepSeparator */ true);
        List<Document> chunks = splitter.apply(pages);

        vectorStore.add(chunks);
    }
}
```

`vectorStore.add(chunks)` 内部会调 `EmbeddingModel.embed(documents)` 拿向量，再写进底层 `ConcurrentHashMap`（`SimpleVectorStore` 实现见 `spring-ai-vector-store/.../SimpleVectorStore.java:55-77`）。第一次跑会有几秒到几十秒，看 PDF 页数。

可选：跑完后调 `vectorStore.save(new File("vector.json"))` 把向量持久化到磁盘，下次直接 `load`，不用每次重新 embed。

## Query：第一版 QuestionAnswerAdvisor

最简单的做法是挂上 `QuestionAnswerAdvisor`：它在 advisor 链的 `before` 阶段把用户问题作为查询去 VectorStore 取 topK 文档，拼到 prompt 模板里再交给模型。

```java
@RestController
@RequiredArgsConstructor
class RagController {

    private final ChatClient chatClient;

    public RagController(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
            .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder()
                    .topK(4)
                    .similarityThreshold(0.5)
                    .build())
                .build())
            .build();
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) {
        return chatClient.prompt().user(q).call().content();
    }
}
```

curl 验证：

```
$ curl 'localhost:8080/ask?q=Spring AI 的 ChatClient 是什么'
ChatClient 是 Spring AI 提供的高阶 API，封装了对模型的调用 ...
```

`QuestionAnswerAdvisor` 的 prompt 模板（`advisors/spring-ai-advisors-vector-store/.../QuestionAnswerAdvisor.java:61-73`）大致是：

```
{原问题}

Context information is below, surrounded by ---------------------
---------------------
{检索到的文档拼接}
---------------------

Given the context and provided history information and not prior knowledge,
reply to the user comment. If the answer is not in the context, inform
the user that you can't answer the question.
```

如果想换措辞或换语种，传一个自定义 `PromptTemplate` 进 `QuestionAnswerAdvisor.builder()` 即可。

## 升级：RetrievalAugmentationAdvisor 流水线

`QuestionAnswerAdvisor` 一锤子搞定取文 + 拼模板，但不能改写问题、不能多查询、不能后处理文档。`RetrievalAugmentationAdvisor`（`spring-ai-rag/.../advisor/RetrievalAugmentationAdvisor.java:107-154`）把这些环节都拆成接口：`QueryTransformer` → `QueryExpander` → `DocumentRetriever` → `DocumentJoiner` → `DocumentPostProcessor` → `QueryAugmenter`。

升级版控制器：

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore, ChatModel chatModel) {

    var rewriter = RewriteQueryTransformer.builder()
        .chatClientBuilder(ChatClient.builder(chatModel))
        .build();

    var retriever = VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .similarityThreshold(0.5)
        .topK(4)
        .build();

    var augmenter = ContextualQueryAugmenter.builder()
        .allowEmptyContext(false)
        .build();

    var rag = RetrievalAugmentationAdvisor.builder()
        .queryTransformers(rewriter)
        .documentRetriever(retriever)
        .queryAugmenter(augmenter)
        .build();

    return builder.defaultAdvisors(rag).build();
}
```

四点变化值得说：

1. `RewriteQueryTransformer` 在召回前先让一个轻量 LLM 把口语化的问题改写成更适合检索的语句（"它有几种风格？" → "Spring AI ChatClient 支持哪些风格？"），命中率明显提高
2. `VectorStoreDocumentRetriever`（`spring-ai-rag/.../retrieval/search/VectorStoreDocumentRetriever.java`）是 `DocumentRetriever` 接口的默认实现，把 `Query` 翻译成 `SearchRequest` 调 `VectorStore.similaritySearch`
3. `ContextualQueryAugmenter` 把检索结果拼成最终的增强 prompt，`allowEmptyContext(false)` 表示检索为空时让模型回答"不知道"而不是凭空编
4. 加 `QueryExpander`（如 `MultiQueryExpander`）能把一个问题扩成多个，并行召回再合并——`RetrievalAugmentationAdvisor` 内部用 `CompletableFuture.supplyAsync` 多查并行（同文件 `:129-134`），并行度由 `taskExecutor` 决定

## 切换到 PgVector 持久化

`SimpleVectorStore` 的 javadoc 明确写"NOT designed for production use"。生产用 PgVector 这类有索引、能事务的存储。

`docker-compose.yml`：

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ai
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
volumes:
  pgdata:
```

换 starter：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

`application.yml` 加：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ai
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
        initialize-schema: true
        schema-name: public
        table-name: vector_store
```

`initialize-schema: true` 让 starter 启动时自动建 `vector_store` 表和索引（生产建议手动建表后置 false）。把上面 `VectorStoreConfig` 里的 `SimpleVectorStore` bean 删掉——starter 会注入一个 `PgVectorStore`，自动满足 `VectorStore` 类型。代码其它地方不用动，这正是 `VectorStore` 抽象的好处（参见 [01-源码剖析/08-vectorstore-filter](../01-源码剖析/08-vectorstore-filter.md)）。

## 元数据过滤

PDF ingest 时给文档打 tag，查询时只检索某一类。改 ingest：

```java
chunks.forEach(c -> c.getMetadata().put("doc_type", "spec"));
vectorStore.add(chunks);
```

查询时通过 advisor 的 context 字段传过滤表达式（`QuestionAnswerAdvisor.FILTER_EXPRESSION`）：

```java
chatClient.prompt()
    .user(q)
    .advisors(a -> a.param(QuestionAnswerAdvisor.FILTER_EXPRESSION, "doc_type == 'spec'"))
    .call()
    .content();
```

字符串表达式由 `FilterExpressionTextParser`（ANTLR4 实现）解析成 `Filter.Expression`，再被各 store 的 converter 翻成原生 SQL/JSON。详细见 [01-源码剖析/08-vectorstore-filter](../01-源码剖析/08-vectorstore-filter.md)。

## 踩坑

1. **chat / embedding 模型分开配**：上面 yml 里两个 `model` 字段独立。混用一个会报"模型不存在 embeddings"
2. **TopK 不是越大越好**：`topK(20)` 把 prompt 撑爆，反而稀释了相关上下文。从 4 起步，按命中质量调
3. **similarityThreshold 别拍脑袋**：OpenAI text-embedding-3-small 的余弦相似度通常 0.5–0.8 算"相关"。设 0.9 会把大量真相关结果过滤掉
4. **PDF 解析空白页**：扫描版 PDF（图片型）`PagePdfDocumentReader` 拿不到文本，返回空 `Document`。需要先用 OCR 处理或换 `tika-reader`
5. **重复 ingest**：`vectorStore.add` 不会去重，重启再跑会写两份。生产上要么先 `delete` 旧的、要么用 `Document.id` 自定义保证幂等
6. **embedding 配额**：1000 页 PDF 切成几千个 chunk，一次 embed 几万 token——OpenAI 会限速。`text-embedding-3-small` 比 `text-embedding-ada-002` 便宜，且 batch API 会自动按 chunk size 分批
7. **PgVector dimensions**：必须和你用的 embedding 模型对得上（1536 是 small；3072 是 large）。设错了 starter 启动建表后第一次写入就报维度不匹配
8. **PgVector 索引类型**：HNSW 比 IVFFlat 召回好但建索引慢。小数据先 IVFFlat，大数据上 HNSW

## 关键代码索引

| 类 / 方法 | 路径 |
| --- | --- |
| `PagePdfDocumentReader` | `document-readers/pdf-reader/.../PagePdfDocumentReader.java:51` |
| `TokenTextSplitter` | `spring-ai-commons/.../transformer/splitter/TokenTextSplitter.java` |
| `SimpleVectorStore` | `spring-ai-vector-store/.../SimpleVectorStore.java:55` |
| `QuestionAnswerAdvisor.before` | `advisors/spring-ai-advisors-vector-store/.../QuestionAnswerAdvisor.java:108` |
| `RetrievalAugmentationAdvisor.before`（七步） | `spring-ai-rag/.../advisor/RetrievalAugmentationAdvisor.java:107-154` |
| `VectorStoreDocumentRetriever` | `spring-ai-rag/.../retrieval/search/VectorStoreDocumentRetriever.java` |
| `RewriteQueryTransformer` | `spring-ai-rag/.../preretrieval/query/transformation/RewriteQueryTransformer.java` |
| `ContextualQueryAugmenter` | `spring-ai-rag/.../generation/augmentation/ContextualQueryAugmenter.java` |
| `PgVectorStore` | `vector-stores/spring-ai-pgvector-store/.../PgVectorStore.java` |
| `FilterExpressionTextParser` | `spring-ai-vector-store/.../filter/antlr4/FilterExpressionTextParser.java` |

## 思考题

1. 如果同一个问题召回的两个文档片段相互矛盾，`ConcatenationDocumentJoiner` 直接拼起来交给模型——你会在 advisor 链的哪一步加去重 / 冲突检测？
2. PDF 一次性 ingest vs 来一份新文档增量 ingest，向量库 schema 要不要带 `source` / `version` metadata？删旧版本时怎么避免删错？
3. `RetrievalAugmentationAdvisor` 的 `taskExecutor` 默认是 `ThreadPoolTaskExecutor`，如果你的应用已经用 reactor，要不要改成共享的 `Schedulers.boundedElastic()`？两种调度器在被 advisor 链复用时的行为差在哪？

## 延伸阅读

- [01-源码剖析/07-modular-rag](../01-源码剖析/07-modular-rag.md)：七阶段流水线为什么要这样切
- [01-源码剖析/08-vectorstore-filter](../01-源码剖析/08-vectorstore-filter.md)：`Filter.Expression` 怎么编译成各家 store 的原生过滤
- [实践 06-tool-calling](./06-tool-calling.md)：让模型主动决定"要不要调 RAG"——把检索包成 tool

> 基于 spring-ai commit 9cde97c1
