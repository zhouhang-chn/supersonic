# Supersonic Knowledge 模块深度解析

`headless/chat/knowledge` 模块是 Supersonic 自然语言理解 (NLU) 能力的基石。它负责构建、维护和查询领域知识库，使得系统能够将用户的自然语言（如“销售额”、“北京”）准确映射到语义模型的实体（指标、维度、维度值）上。

## 1. 核心定位

在 Chat BI 流程中，Knowledge 模块主要服务于 **MAPPING (实体识别)** 阶段。它充当了一个增强版的“词典服务”，不仅包含通用的中文词汇，更动态地加载了当前业务域下的所有 Schema 信息。

其核心价值在于：
*   **消除歧义**: 区分“苹果”是水果还是公司（通过上下文和领域词典）。
*   **泛化匹配**: 支持前缀搜索、后缀搜索和拼写容错（Edit Distance）。
*   **动态更新**: 当用户在管理后台新建指标时，知识库能实时更新，无需重启服务。

## 2. 架构概览

该模块采用了 **分层架构**，底层依赖 HanLP 进行分词和词性标注，上层封装了业务特定的搜索和管理逻辑。

| 组件 | 类名 | 职责 |
| :--- | :--- | :--- |
| **Service Layer** | `KnowledgeBaseService` | 对外暴露的统一入口，负责协调词典的加载、更新和查询。 |
| **Search Engine** | `SearchService` | 实现了基于 Trie 树的高性能内存搜索（前缀/后缀匹配）。 |
| **NLP Engine** | `HanlpHelper` | HanLP 库的适配器，管理 `DynamicCustomDictionary` 和分词器配置。 |
| **Builders** | `*WordBuilder` | 负责将语义对象（Metric/Dimension）转换为词典中的 `DictWord`。 |

## 3. 核心机制解析

### 3.1 动态词典构建 (Dictionary Construction)

Supersonic 将 Schema 元素视为“词汇”注入到 NLP 引擎中。这是通过 `headless/chat/knowledge/builder` 包下的构建器实现的。

*   **词性编码 (Nature Encoding)**: 系统为每个 Schema 元素生成唯一的“词性”标签。
    *   例如：`n_metric_1001` 可能代表 ID 为 1001 的指标。
    *   这种编码方式使得分词结果直接携带了元数据信息。
*   **构建流程**:
    1.  `MetricWordBuilder`: 提取指标名称、别名。
    2.  `DimensionWordBuilder`: 提取维度名称、别名。
    3.  `ValueWordBuilder`: 提取维度的具体取值（如“上海”、“广东”）。
    4.  这些信息被封装为 `DictWord` 对象，包含 `word` (文本) 和 `nature` (词性)。

### 3.2 内存索引与搜索 (SearchService)

`SearchService` 维护了两个核心数据结构（基于 `BinTrie`）：
1.  **Prefix Trie (`trie`)**: 用于标准的前缀搜索。
2.  **Suffix Trie (`suffixTrie`)**: 存储倒序字符串，用于高效的后缀搜索。

**搜索特性**:
*   **模糊匹配**: 结合 `EditDistanceUtils` 计算相似度，支持一定程度的拼写错误。
*   **上下文过滤**: `prefixSearch` 和 `suffixSearch` 方法接收 `detectDataSetIds` 参数，确保只返回当前上下文相关的数据集实体，避免跨域干扰。

### 3.3 HanLP 集成 (HanlpHelper)

`HanlpHelper` 是连接 Supersonic 业务逻辑与 HanLP 底层的胶水代码。
*   **Custom Dictionary**: 使用 `MultiCustomDictionary` 管理自定义词汇。它支持从文件系统（本地或 HDFS）加载词典。
*   **热更新**: `addToCustomDictionary` 和 `removeFromCustomDictionary` 方法允许在运行时动态修改分词器的词典，这对于 BI 系统至关重要（业务术语经常变动）。

## 4. 关键工作流

### 4.1 知识库初始化/重载
1.  `KnowledgeBaseService.reloadAllData()` 被调用（通常是定时任务或事件触发）。
2.  `HanlpHelper.reloadCustomDictionary()` 重置 HanLP 词典。
3.  各类 `WordBuilder` 遍历语义模型，生成 `DictWord` 列表。
4.  `KnowledgeBaseService.updateSemanticKnowledge()` 将新词注入 HanLP 和 `SearchService` 的 Trie 树。

### 4.2 实体识别 (Mapping)
1.  用户输入 Query：“查看上海的销售额”。
2.  `KnowledgeBaseService.getTerms()` 调用 `HanlpHelper.getTerms()`。
3.  HanLP 分词器工作，识别出“上海” (Tag: `v_city_shanghai`) 和 “销售额” (Tag: `n_metric_sales`)。
4.  系统解析这些 Tag，将其映射回具体的 Schema 对象 ID。

## 5. 总结

Supersonic 的 Knowledge 模块通过巧妙地利用 NLP 分词技术，将复杂的 Schema 映射问题转化为了经典的 **分词+词性标注** 问题。

*   **优点**: 充分利用了成熟 NLP 库的能力，匹配准确率高，且支持复杂的中文语境。
*   **挑战**: 随着维度值（Value）数量的爆炸（例如百万级的用户 ID），内存字典可能会成为瓶颈。这也是为什么 Supersonic 引入了 `EmbeddingMapper`（向量检索）作为补充，以处理大规模、长尾的实体识别。
