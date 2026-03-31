# 基于工具链协同的多智能体无线网络优化系统 - 记忆机制图

## 架构设计

本记忆系统采用**分层存储、四阶段生命周期**设计，支持混合检索和多模态索引，实现历史经验的自动沉淀与智能复用。

## Mermaid 记忆机制图

```mermaid
flowchart TD
    %% ============ 记忆生成阶段 ============
    Start[新任务完成<br/>诊断结果+执行日志] --> Extract{经验抽取}
    
    Extract -->|自动提取| AutoExtract[LLM自动抽取<br/>│ • 问题摘要<br/>│ • 根因分析<br/>│ • 解决方案<br/>│ • 效果评估<br/>│ • 关键标签]
    Extract -->|人工标注| ManualExtract[专家人工标注<br/>补充关键标签与验证]
    
    AutoExtract --> Validate{案例验证}
    ManualExtract --> Validate
    
    Validate -->|通过| GenCase[生成结构化案例]
    Validate -->|不通过| Revise[人工修正]
    Revise --> GenCase
    
    %% ============ 记忆索引阶段 ============
    GenCase --> StoreSQL[存入SQLite<br/>摘要层]
    StoreSQL --> Vectorize[向量化处理<br/>BGE-M3 768维]
    Vectorize --> IndexChroma[更新ChromaDB<br/>向量层]
    IndexChroma --> BuildIndex[建立标签索引<br/>倒排索引]
    BuildIndex --> Archive[压缩归档<br/>原始记录层]
    
    %% ============ 记忆检索阶段 ============
    subgraph Retrieval[混合检索机制]
        Query[用户新问题] --> Parse[问题解析<br/>LLM提取特征 + 关键词]
        Parse --> VectorSearch[向量检索<br/>ChromaDB<br/>Top-50]
        Parse --> TagSearch[标签检索<br/>SQL LIKE<br/>Top-20]
        Parse --> KeywordSearch[关键词检索<br/>BM25全文检索<br/>Top-30]
        
        VectorSearch --> Merge[结果融合<br/>去重 + 分数归一化<br/>加权排序]
        TagSearch --> Merge
        KeywordSearch --> Merge
        
        Merge --> Dedup[去重去噪]
        Dedup --> Rank[综合评分排序]
        Rank --> ReturnTopK[返回Top-5<br/>最相关案例]
    end
    
    %% ============ 记忆更新阶段 ============
    NewTask[新任务完成] --> AutoExtract2[LLM自动抽取<br/>（同生成阶段）]
    AutoExtract2 --> StoreSQL2[SQLite增量写入]
    StoreSQL2 --> Vectorize2[向量实时更新]
    Vectorize2 --> IndexChroma2[ChromaDB实时索引]
    IndexChroma2 --> CheckCompress{检查压缩条件?}
    CheckCompress -->|是| WeeklyCompress[每周压缩任务<br/>合并去重归档]
    CheckCompress -->|否| Keep[保持在线]
    
    %% ============ 分层存储结构 ============
    subgraph LayeredStorage[分层存储结构]
        SQLiteLayer[摘要层<br/>SQLite<br/>问题|根因|方案|效果]
        VectorLayer[向量层<br/>ChromaDB<br/>embedding向量]
        RawLayer[原始记录层<br/>压缩文件<br/>完整日志+原始数据]
    end
    
    StoreSQL --> SQLiteLayer
    IndexChroma --> VectorLayer
    Archive --> RawLayer
    
    %% ============ 应用集成 ============
    S3[子任务派发<br/>S3阶段] -.->|混合检索| Retrieval
    S5[反思沉淀<br/>S5阶段] -.->|触发更新| NewTask
    
    %% ============ 关键数据流 ============
    ReturnTopK -.->|注入上下文| S3
    S3 -->|执行结果| NewTask
    
    %% ============ 样式定义 ============
    classDef phase fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef process fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef storage fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    classDef retrieval fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    
    class Start,Extract,Validate,GenCase,StoreSQL,Vectorize,IndexChroma,BuildIndex,Archive,StoreSQL2,Vectorize2,IndexChroma2,CheckCompress,WeeklyCompress,Keep phase
    class AutoExtract,ManualExtract,Revise,AutoExtract2 process
    class Retrieval,Query,Parse,VectorSearch,TagSearch,KeywordSearch,Merge,Dedup,Rank,ReturnTopK retrieval
    class LayeredStorage,SQLiteLayer,VectorLayer,RawLayer storage
```

---

## 记忆系统四阶段详解

### 阶段一：记忆生成（Generation）

**触发时机**：子任务完成且结果满足验收标准

**抽取方式**：
- **自动提取**（主要）：LLM从执行日志、输入输出、KPI变化中提取结构化信息
- **人工标注**（辅助）：专家对复杂案例进行标签标注和根因验证

**输出结构**：
```json
{
  "case_id": "case_20260331_001",
  "problem_summary": "流量拥塞且覆盖边缘",
  "root_cause": "弱覆盖导致边缘用户集中",
  "solution": ["功率提升3dB", "切换参数优化"],
  "effectiveness": {
    "traffic_load": "↓15%",
    "user_satisfaction": "↑8%"
  },
  "tags": ["流量", "覆盖", "功率调整"],
  "timestamp": "2026-03-31T10:30:00Z",
  "source_task": "t_001_rate_optimization"
}
```

**质量控制**：案例分析，必须包含根因、方案、效果三个核心要素，不满意的案例退回人工修正。

---

### 阶段二：记忆索引（Indexing）

**向量化**：
- 使用`BGE-M3`模型生成case的embedding（768维）
- 输入文本：`problem_summary + root_cause + solution_text`
- 存储到ChromaDB Collection `case_embeddings`

**标签化**：
- 自动提取：LLM从`problem_summary`和`solution`中提取关键词
- 人工补充：专家标注的关键标签
- 存储到SQLite字段`tags_json`

**倒排索引**：
- 对`tags_json`、`location`、`problem_type`等字段建立索引
- 使用SQLite FTS5实现全文检索能力

**分层存储**：
```
/shared_workspace/memory/
├── cases.sqlite                          # 摘要层 - 结构化数据
├── chroma_db/                           # 向量层 - embedding索引
│   └── collections/
│       └── case_embeddings/
└── raw_archive/                         # 原始记录层 - 压缩存储
    └── case_20260331_001.json.gz
```

---

### 阶段三：记忆检索（Retrieval）

**触发点**：S3阶段子任务派发前，需要注入历史经验

**混合检索策略**：

| 检索方式 | 数据库 | 召回数量 | 特点 |
|---------|--------|---------|------|
| 向量相似度 | ChromaDB | Top-50 | 语义匹配，召回率高 |
| 标签过滤 | SQLite | Top-20 | 精确匹配，查准率高 |
| 关键词FTS | SQLite FTS5 | Top-30 | 字段匹配，兼顾两者 |

**融合算法**：
```
score_final = 0.5 * score_vector
             + 0.3 * score_tag
             + 0.2 * score_keyword
```

**降级机制**：若ChromaDB或embedding服务不可用，自动降级为仅标签+关键词检索

**输出**：Top-5最相关案例，包含完整问题描述、根因链路、工具调用序列

---

### 阶段四：记忆更新（Updating）

**增量写入**：
- 新案例直接INSERT到SQLite
- 同时add到ChromaDB（实时索引）
- 事件日志追加到`events/event_log.jsonl`

**定期压缩**：
- **触发条件**：每周凌晨低峰期
- **操作**：
  1. 合并碎片化的原始记录
  2. 去重（基于`problem_summary`语义相似度）
  3. 归档超过1年的低频案例到冷存储
  4. 重建向量索引（去除已归档案例）

**版本控制**：每个案例有`case_id`唯一标识，支持修订版（`case_001_v2`），历史版本可追溯

---

## 关键API接口

```python
class MemorySystem:
    """记忆系统统一接口"""

    async def retrieve(query: str, top_k: int = 5, filters: dict = None) -> List[Case]:
        """混合检索入口"""
        # 1. 问题向量化
        embedding = await embed_text(query)

        # 2. 多路召回
        vector_hits = await chroma.query(embedding, n_results=top_k*2)
        tag_hits = await sqlite.tag_search(extract_tags(query), limit=top_k)
        keyword_hits = await sqlite.fts_search(query, limit=top_k)

        # 3. 融合排序
        merged = hybrid_merge(vector_hits, tag_hits, keyword_hits)
        ranked = rerank_by_similarity(merged, query_embedding)

        # 4. 返回Top-k
        return ranked[:top_k]

    async def store(case: Case):
        """案例入库（生成阶段）"""
        # 1. 验证质量
        if not validate_case(case):
            raise InvalidCaseError("案例缺少根因或效果评估")

        # 2. 分配ID
        case.case_id = generate_case_id()

        # 3. 写入SQLite
        await sqlite.insert_case(case)

        # 4. 向量化并写入ChromaDB
        embedding = await embed_text(case.to_text())
        await chroma.add(
            ids=[case.case_id],
            embeddings=[embedding],
            metadatas=[case.to_metadata()]
        )

        # 5. 归档原始记录
        await gzip_archive(case.to_json(), f"raw_archive/{case.case_id}.json.gz")

    async def update_from_task_completion(task_id: str):
        """从任务完成自动触发记忆更新"""
        # 1. 读取任务执行日志
        logs = await read_task_logs(task_id)

        # 2. LLM抽取结构化案例
        case = await llm_extract_case(logs)

        # 3. 入库
        await self.store(case)
```

---

## 与系统其他模块的交互

### 1. 与S3子任务派发的交互
- **时机**：派发前调用`memory.retrieve()`获取Top-5相似案例
- **注入方式**：将案例序列化为文本，附加到子任务prompt的上下文部分
- **效果**：垂直Agent在工具调用前已了解同类问题的处理模式

### 2. 与S5反思沉淀的交互
- **时机**：子任务完成后触发`memory.update_from_task_completion()`
- **数据源**：任务执行日志 + 诊断结论 + 效果验证KPI
- **异步处理**：抽取和向量化在后台执行，不阻塞主流程

### 3. 与共享工作区的交互
- **存储位置**：所有记忆文件位于`/shared_workspace/memory/`
- **并发控制**：使用文件锁（fcntl）保证多Agent同时读写安全
- **备份策略**：每日自动备份到独立存储卷

---

## 性能指标

| 指标 | 目标值 | 测量方法 |
|-----|-------|---------|
| 检索延迟 | < 500ms | P95延迟，QPS>50 |
| 向量检索召回率 | > 85% | 人工标注测试集 |
| 标签检索精确率 | > 90% | 人工标注测试集 |
| 案例入库延迟 | < 2s | 从提取到完成索引 |
| 存储增长率 | < 1GB/月 | 按日均10个案例估算 |

---

## 文件说明

- **文件名**: `memory-mechanism-diagram.md`
- **内容**: 完整的记忆机制Mermaid图 + 四阶段详解
- **用法**: 可用Mermaid渲染工具（Typora、Obsidian、GitHub、Mermaid Live Editor）查看图形化流程

该图清晰展示了记忆系统从**生成→索引→检索→更新**的完整生命周期，以及分层存储结构和混合检索机制，可与执行流程图的S3、S5阶段对应理解。