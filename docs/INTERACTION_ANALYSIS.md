# Supersonic 前后端交互与语义层解析

本文档基于代码库分析，详细阐述了 Supersonic 前端（Webapp）与后端（Headless/Chat）之间的交互机制，重点关注语义层的管理与查询解析流程。

## 1. 架构概览

Supersonic 采用前后端分离架构：
*   **前端 (Frontend)**: 基于 React + Umi 框架，主要包含 `supersonic-fe`（管理后台）和 `chat-sdk`（对话组件）两个核心包。
*   **后端 (Backend)**: 基于 Java Spring Boot，分为 `headless`（语义层核心）和 `chat`（对话服务）等模块。
*   **交互协议**: RESTful API，数据格式为 JSON。
*   **认证机制**: 基于 Token 的认证。前端通过 `request.ts` 拦截器在 HTTP Header 中注入 `Authorization: Bearer <token>`。

## 2. 语义层管理交互 (Semantic Layer Management)

语义层的核心在于对数据模型（Model）、指标（Metric）和维度（Dimension）的定义。这部分交互主要由 `supersonic-fe` 发起，后端由 `headless` 模块处理。

### 2.1 核心服务文件
*   **前端**: `webapp/packages/supersonic-fe/src/pages/SemanticModel/service.ts`
*   **后端**: `headless/server/.../rest/` 下的各类 Controller。

### 2.2 关键交互接口

| 功能模块 | 前端函数示例 | 后端 API 路径 | 后端 Controller | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| **主题域 (Domain)** | `getDomainList` | `/api/semantic/domain/getDomainList` | `DomainController` | 获取业务域列表 |
| **模型 (Model)** | `createModel`, `getModelList` | `/api/semantic/model/...` | `ModelController` | 定义逻辑数据模型，绑定物理表 |
| **指标 (Metric)** | `createMetric`, `queryMetric` | `/api/semantic/metric/...` | `MetricController` | 定义原子指标和派生指标 |
| **维度 (Dimension)** | `createDimension`, `getDimensionList` | `/api/semantic/dimension/...` | `DimensionController` | 定义分析维度 |
| **数据源 (Datasource)** | `createDatasource` | `/api/semantic/datasource/...` | `DataSetController` | 管理底层数据库连接 |

### 2.3 数据流向
1.  **定义**: 用户在前端“语义建模”页面配置指标和维度。
2.  **请求**: 前端将配置封装为 JSON 对象（如 `ModelReq`, `MetricReq`）发送给后端。
3.  **持久化**: 后端 Controller 接收请求，通过 Service 层调用 DAO 将元数据存入数据库（H2/MySQL）。
4.  **生效**: `SemanticSchemaManager` 会定期或事件触发刷新内存中的 `SemanticSchema`，使新的指标立即对 Chat 模块可见。

## 3. 对话与查询交互 (Chat & Query Process)

这是 Supersonic 的核心业务流程，实现了“自然语言 -> SQL -> 数据”的转化。

### 3.1 核心服务文件
*   **前端**: `webapp/packages/chat-sdk/src/service/index.ts`
*   **后端**: `chat/server/.../rest/ChatQueryController.java`

### 3.2 交互三部曲

Supersonic 的查询通常不是一次性的，而是分为 **Parse (解析)** 和 **Execute (执行)** 两个阶段，有时包含 **Query (综合查询)** 阶段。

#### 第一阶段：解析 (Parse)
*   **目的**: 理解用户意图，消除歧义。
*   **前端调用**: `chatParse(...)`
*   **API**: `POST /api/chat/query/parse`
*   **后端处理**:
    1.  `ChatQueryController` 接收自然语言问题（`queryText`）。
    2.  调用 `SemanticParser`（如 LLM 解析器或规则解析器）。
    3.  结合 `SemanticSchema` 进行意图识别。
    4.  **返回**: `ParseResp`，其中包含 `ParseDataType`。如果存在歧义（例如“销售额”可能有“昨日销售额”和“累计销售额”两种定义），会返回多个候选集供用户选择。

#### 第二阶段：执行 (Execute)
*   **目的**: 生成 SQL 并查询数据。
*   **前端调用**: `chatExecute(...)`
*   **API**: `POST /api/chat/query/execute`
*   **后端处理**:
    1.  接收前端确认的 `ParseInfo`（包含明确的指标、维度、筛选条件）。
    2.  调用 `SemanticTranslator` 将语义对象翻译为物理 SQL。
    3.  使用 Apache Calcite 进行 SQL 优化和重写。
    4.  连接数据库执行 SQL。
    5.  **返回**: `QueryResp`，包含表头信息（columns）和数据行（resultList），以及生成的 SQL 语句（用于调试或展示）。

#### 综合查询 (Query)
*   **前端调用**: `chatQuery(...)`
*   **API**: `POST /api/chat/query/query`
*   **描述**: 在某些简易模式下，可能直接调用此接口一步到位获取结果，或者作为初始入口判断是否需要进一步交互。

### 3.3 其他辅助接口
*   **推荐搜索**: `searchRecommend` (`/api/chat/query/search`) - 提供输入联想。
*   **历史记录**: `getHistoryMsg` (`/api/chat/manage/pageQueryInfo`) - 获取对话历史。
*   **反馈**: `updateQAFeedback` (`/api/chat/manage/updateQAFeedback`) - 用户对查询结果的点赞/点踩。

## 4. 总结

Supersonic 的前后端交互清晰地反映了其**语义层架构**的设计思想：

1.  **管理端 (supersonic-fe)** 通过 `/api/semantic/*` 接口构建丰富的语义元数据（Schema）。
2.  **用户端 (chat-sdk)** 通过 `/api/chat/*` 接口利用这些元数据，通过 **Parse-Execute** 两段式交互，确保了自然语言查询的准确性和可控性。
3.  后端作为无头（Headless）服务，屏蔽了底层物理数据库的复杂性，向前端暴露统一的语义接口。
