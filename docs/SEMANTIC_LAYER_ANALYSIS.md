# Supersonic 语义层架构解析

## 1. 概述

Supersonic 的核心价值在于其“语义层”（Semantic Layer），它充当了自然语言与底层物理数据之间的智能中介。该层的主要职责是理解用户的业务意图，将其映射到预定义的语义模型（指标、维度、实体），并最终生成可执行的 SQL 查询。

在代码库中，语义层的核心逻辑集中在 `headless` 模块中。该模块采用了清晰的分层架构，将接口定义、意图理解、查询翻译和元数据管理解耦。

## 2. 核心模块结构 (`headless`)

`headless` 模块内部进一步划分为四个子模块，各司其职：

*   **`headless-api` (契约层)**:
    *   定义了语义层的数据交互标准。
    *   核心对象：
        *   `SemanticParseInfo`: 代表解析后的结构化查询意图（包含指标、维度、筛选条件等）。
        *   `SemanticSchema`: 语义元数据的内存镜像（包含数据集定义、模型关系等）。
    *   作用：确保各模块间通过统一的 POJO 进行通信，避免强耦合。

*   **`headless-chat` (理解层)**:
    *   负责 Natural Language Understanding (NLU)。
    *   核心组件：
        *   `SemanticParser`: 语义解析器接口。
        *   **实现策略**：
            *   `RuleSqlParser`: 基于规则的解析，处理明确的时间范围、聚合类型等。
            *   `LLMSqlParser`: 基于大语言模型（LLM）的解析，处理复杂的非结构化提问。
    *   作用：将用户的自然语言“翻译”成 `SemanticParseInfo`。

*   **`headless-core` (翻译层)**:
    *   负责将结构化的语义对象转换为物理 SQL。
    *   核心组件：
        *   `SemanticTranslator`: 翻译器接口。
        *   `DefaultSemanticTranslator`: 默认实现，通过一系列 `QueryParser` 和 `QueryOptimizer` 进行处理。
        *   `QueryParser` 实现：如 `OntologyQueryParser`（本体查询）、`MetricExpressionParser`（指标表达式）等。
    *   技术栈：深度集成了 **Apache Calcite**，用于 SQL 的解析、验证和重写（`SqlMergeWithUtils`）。

*   **`headless-server` (服务层)**:
    *   作为对外暴露的 Facade，编排整个处理流程。
    *   核心组件：
        *   `S2SemanticLayerService`: 核心服务入口。
        *   `SemanticSchemaManager`: 管理语义元数据的加载和更新。

## 3. 核心数据流 (Data Flow)

一个标准的用户查询（`SemanticQueryReq`）的处理流程如下：

1.  **入口 (Entry)**:
    *   请求进入 `chat` 模块，转发给 `S2SemanticLayerService.queryByReq()`。

2.  **解析 (Parsing)**:
    *   服务层调用 `headless-chat` 中的解析器。
    *   解析器结合 `SemanticSchema`（上下文），将自然语言转换为 `SemanticParseInfo`。
    *   系统可能会生成多个候选解析结果，并根据置信度打分选择最优解。

3.  **构建 (Building)**:
    *   `S2SemanticLayerService` 将 `SemanticParseInfo` 封装为 `QueryStatement` 对象。
    *   `QueryStatement` 包含了查询所需的所有上下文信息，包括环境配置、用户权限等。

4.  **翻译 (Translation)**:
    *   调用 `DefaultSemanticTranslator.translate(QueryStatement)`。
    *   **Pipeline 处理**：
        *   **Parsing**: 遍历 `QueryParser` 链（如处理指标计算逻辑、维度映射）。
        *   **Optimization**: 遍历 `QueryOptimizer` 链进行查询优化。
        *   **SQL Generation**: 利用 Calcite 或字符串模板生成最终的物理 SQL。
        *   **Merging**: `SqlMergeWithUtils` 负责将内部的本体查询（Ontology Query）与外部的 SQL 结构（如 `WITH` 子句）进行合并。

5.  **执行 (Execution)**:
    *   生成的 SQL 通过 `QueryExecutor` 发送到底层数据源（如 MySQL, ClickHouse 等）。
    *   结果返回后，可能还会进行一些后处理（如格式化、单位转换）。

## 4. 关键类解析

*   **`S2SemanticLayerService`**:
    *   位置：`headless/server/.../impl/S2SemanticLayerService.java`
    *   职责：总指挥。它不关心具体的解析或翻译逻辑，而是负责连接各个组件，管理缓存 (`QueryCache`)，记录统计信息 (`StatUtils`)，并处理权限校验 (`@S2DataPermission`)。

*   **`SemanticSchema`**:
    *   位置：`headless/api/.../pojo/SemanticSchema.java`
    *   职责：字典。解析器和翻译器都高度依赖它来理解“销售额”对应哪个数据库字段，“去年”对应哪个时间范围。

*   **`DefaultSemanticTranslator`**:
    *   位置：`headless/core/.../translator/DefaultSemanticTranslator.java`
    *   职责：编译器。它将“业务方言”编译成“数据库方言”。它采用了 **Chain of Responsibility**（责任链）模式来组织不同的解析器和优化器，极大地提高了扩展性。

## 5. 总结

Supersonic 的架构设计体现了优秀的**关注点分离**原则：
*   **Chat 模块**专注于“听懂”人话。
*   **Core 模块**专注于“生成”机器代码。
*   **Server 模块**专注于“管理”流程和资源。
*   **API 模块**保证了各层之间的协议稳定。

这种架构使得 Supersonic 能够灵活地替换底层的 LLM 模型（只需修改 Chat 层），或者适配新的数据库引擎（只需扩展 Core 层），具有强大的生命力和扩展空间。
