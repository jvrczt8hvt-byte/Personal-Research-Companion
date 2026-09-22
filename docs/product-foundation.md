# Personal Research Companion：产品基础架构与长期记忆

> 状态：产品讨论基线（持续更新，不等同于最终技术规格）  
> 首次归档：2026-09-22  
> 用途：记录已经确认的产品方向、架构边界和关键设计原则，作为后续产品讨论的共同上下文。

## 1. 产品定位

Personal Research Companion 是一个面向科研新手与跨领域研究者的长期科研助手。

它不只是一次性的论文总结工具或学习陪练，而是帮助用户在时间有限的情况下，以“边研究、边学习、边实践”的方式进入陌生领域。用户可以从一篇前沿论文、技术文档或研究问题出发，系统按当前任务即时补充必要的基础知识，并逐步引导用户完成阅读、笔记和最小实验。

个人知识地图是长期使用过程中形成的结果，不是产品本体。

## 2. 目标用户与核心场景

### 目标用户

- 刚开始科研的研究生；
- 从一个学科进入另一个陌生领域的研究者；
- 已能理解概念定义，但不知道学习顺序、技术步骤和实验方法的用户。

### 代表场景

用户开始阅读 GraphRAG 的文档或论文，能够查询并理解“GraphRAG 是什么”，但仍然不知道：

- 需要哪些前置知识；
- 技术名词之间有什么关系；
- 应按什么顺序学习；
- GitHub 项目如何运行；
- 一个最小实验如何设计；
- 前沿论文与基础知识如何连接。

## 3. 核心产品原则

1. **前沿问题牵引**：从用户正在阅读的论文、文档或实际研究问题出发。
2. **按需补充基础**：将知识分为“现在必须懂”“用到时再懂”“暂时可以跳过”。
3. **边学边用**：不要求用户先完成一整套课程，再开始研究。
4. **长期伴随**：每日发现、论文阅读、笔记、实验和想法使用同一份长期记忆。
5. **证据可追溯**：论文状态、来源和 Agent 解释必须清晰区分，不伪造引用。
6. **人保持控制权**：收藏、写入笔记、安装依赖和运行代码等重要动作需要用户确认。

## 4. 产品核心循环

### 日常循环

```text
追踪兴趣 → 检索前沿论文 → 个性化筛选 → 每日推送 3–5 篇 → 用户反馈
```

默认交互目标是让用户在 5–10 分钟内完成论文筛选。每篇推荐应说明：

- 论文解决了什么问题；
- 为什么与当前用户相关；
- 发表状态与来源；
- 阅读难度和必要前置知识；
- 建议略读、精读、收藏或暂时跳过。

### 深度任务循环

```text
选择论文/文档 → 识别意图 → 诊断知识缺口 → 按需补基础
→ 解析论文 → 生成结构化笔记 → 指导最小实验 → 复盘并更新记忆
```

## 5. 总体技术架构

```text
用户请求 / 每日定时任务
          │
          ▼
LangGraph Research Supervisor
├─ 全局状态 State
├─ Agent 路由与子图编排
├─ Checkpoint 与任务恢复
├─ Interrupt 与人工确认
└─ Retry / Replay / Fork
          │
          ├─ 调用 Dify 专业 Agent / Workflow
          ├─ 调用学术检索 API
          └─ 调用 Local Research Bridge
                    ├─ Zotero
                    └─ Obsidian
```

### LangGraph 的职责

LangGraph 是全局有状态协调器，也是流程状态的唯一权威来源：

- 管理任务生命周期和 Agent 调用顺序；
- 维护共享 State；
- 编排每日推送、论文阅读与实验子图；
- 处理暂停、恢复、人工确认和失败重试；
- 通过 checkpoint 支持 replay 与 fork；
- 通过长期 Store 维护跨任务用户记忆。

### Dify 的职责

Dify 用于减少专业 Agent 的编码量。Dify 中的 Agent 或 Workflow 应尽量无状态，并通过 API 被 LangGraph 调用：

- 可视化配置模型、提示词和工具；
- 实现文献检索、论文筛选、笔记生成和知识缺口分析等专业能力；
- 提供文档提取、知识检索和调用外部 API 的节点；
- 提供单个 Agent 的调试、日志和模型切换能力。

Dify 的对话历史和 Workflow 变量不是全局任务状态的权威来源。

### Zotero 与 Obsidian 的职责

- **Zotero**：保存论文元数据、PDF、集合、标签和引用键；
- **Obsidian**：保存论文笔记、概念笔记、实验记录和研究想法；
- **LangGraph State**：只保存 Zotero item key、Obsidian 路径、内容哈希和写入回执，不复制整个科研资产。

## 6. MVP Agent 划分

### 6.1 Intent Classifier

识别用户当前意图，例如检索论文、快速理解、生成笔记、补基础、进行实验或记录想法。首版优先使用结构化分类，不必做成拥有自由工具调用能力的 Agent。

### 6.2 Literature Search Agent

- 从 DBLP、OpenAlex、Semantic Scholar、OpenReview 等来源检索论文；
- 以具体研究子领域的顶会白名单作为主要筛选规则；
- 区分正式发表论文与预印本；
- 聚合、标准化和去重，不负责最终个性化推荐。

候选池分为：

- 主池：目标领域顶会的正式论文；
- 探索池：与用户近期问题高度相关的少量预印本。

### 6.3 Paper Screening Agent

根据主题相关性、发表状态、用户知识水平、历史阅读和难度，将候选论文压缩成每日 3–5 篇，并解释推荐或跳过的理由。

### 6.4 Paper Notes Agent

输出结构化论文笔记，至少包含问题、方法、相对已有工作的变化、数据集、实验设置、主要结果、局限性、关键术语、关联笔记和用户问题。必须区分论文事实、作者结论和 Agent 解释。

### 6.5 Knowledge Gap Agent

识别阻碍当前任务的前置知识，按必要程度排序，并使用当前论文或实验的上下文进行解释，不扩展成完整课程。

### 6.6 Experiment Mentor Agent

将论文或文档转化为可执行的最小实验，分步解释环境、命令、输入、预期输出、指标和常见错误。每个重要步骤完成后进行验证，再进入下一步。

## 7. State 所有权设计

State 分为三层：

### 7.1 任务状态：Checkpoint

保存一次 digest、论文阅读或实验任务恢复所需的信息，例如：

```text
task_id, user_id, task_type, status, intent,
candidate_papers, selected_paper, evidence,
knowledge_gaps, note_draft, experiment,
pending_action, retry_count, errors, receipts, next_action
```

### 7.2 长期用户记忆：Store

```text
research_interests, preferred_venues, known_concepts,
uncertain_concepts, reading_history, paper_feedback,
completed_experiments, emerging_ideas
```

### 7.3 外部科研资产

保存在 Zotero 与 Obsidian 中，通过稳定标识符和回执与 LangGraph 状态关联。

## 8. Thread 与 Checkpoint 策略

不同任务使用不同的 `thread_id`，但共享同一用户 Store：

```text
user:{user_id}:digest:{date}
user:{user_id}:paper:{paper_id}
user:{user_id}:experiment:{experiment_id}
```

关键 checkpoint 设置在：

1. 意图确认后；
2. 外部检索和去重完成后；
3. 用户选择论文后；
4. PDF 解析与证据提取完成后；
5. 笔记草稿生成后；
6. 写入 Zotero/Obsidian 前；
7. 外部写入成功并取得 receipt 后；
8. 每个实验步骤验证通过后。

实验子图需要细粒度 checkpoint，避免回退时重跑整个实验流程。

## 9. 中断、恢复与回退

### 必须中断并等待用户确认的动作

- 选择要深入阅读的论文；
- 导入 Zotero；
- 新建或覆盖 Obsidian 笔记；
- 安装依赖或运行代码；
- Agent 对用户知识水平判断置信度不足；
- 实验结果异常，需要用户提供输出。

重要副作用必须采用以下顺序：

```text
生成动作计划 → 保存 checkpoint → interrupt 请求确认
→ 使用同一 thread_id 恢复 → 执行动作 → 保存 receipt
```

### 回退类型

- **Retry**：当前节点的暂时错误，限次重试并使用退避策略；
- **Replay**：从历史 checkpoint 重新执行后续节点；
- **Fork**：用户改变目标或修正状态时，从旧 checkpoint 建立新分支，保留原历史；
- **Compensation**：处理已经发生的外部副作用，不假设 checkpoint 可以自动撤销 Zotero 或 Obsidian 写入。

外部操作必须使用幂等键、内容哈希和写入回执。Obsidian 写入前应保留版本；Zotero 导入通过 DOI 或引用键去重，不自动删除条目。

## 10. 本地部署基线

当前建议使用本地 Dify Community Edition：

```text
Windows 主机
├─ Zotero Desktop
├─ Obsidian Vault
└─ Local Research Bridge
          ▲
          │ host.docker.internal / 受控本地接口
Docker Desktop
├─ Dify
├─ LangGraph API
├─ PostgreSQL / Checkpointer
├─ Redis
└─ 向量数据库
```

推荐实施顺序：

1. 检查 Docker Desktop、WSL2、Git、资源与端口；
2. 启动干净的本地 Dify；
3. 配置模型并验证 Dify Workflow API；
4. 增加 LangGraph 与持久化 checkpointer；
5. 接入 Zotero/Obsidian Bridge；
6. 实现第一个 Literature Search Subgraph。

## 11. 当前已确认决策

- 产品面向科研新手与跨领域研究者；
- 使用前沿问题牵引、按需补充基础的学习方式；
- 日常推送目标为每天 3–5 篇、5–10 分钟完成筛选；
- 支持从选中论文进入结构化笔记和最小实验；
- LangGraph 是多智能体协调器和状态权威来源；
- Dify 用于低代码实现可替换的专业 Agent；
- Zotero 管文献资产，Obsidian 管知识资产；
- 当前建议本地部署 Dify；
- 所有重要外部写入和代码执行保留人工确认。

## 12. 后续产品问题

以下内容尚未最终确定，将在后续产品讨论中逐项确认：

- 产品正式英文名称；
- 第一批支持的研究领域与顶会白名单；
- 每日推送渠道与交互形式；
- 论文筛选评分模型；
- Zotero 与 Obsidian 的具体模板和同步规则；
- 最小实验的安全边界；
- 选题思考与想法记录的交互方式；
- MVP 的验证指标和目标用户访谈计划。

## 13. 参考资料

- [Dify 官方仓库与本地部署说明](https://github.com/langgenius/dify)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Time Travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel)
