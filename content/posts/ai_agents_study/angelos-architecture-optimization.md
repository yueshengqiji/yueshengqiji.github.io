---
title: "Angelos 架构现状审视与四项优化建议"
date: 2026-11-18T13:30:00+08:00
draft: false
tags: ["Angelos", "Agent架构", "架构优化"]
categories: ["ai_agents_study"]
description: "基于代码库与执行管线实测，系统梳理当前 Angelos 架构特征、核心瓶颈与四项落地优化建议"
---

## 一、 当前架构的核心特征与现状

1. **分层执行与严苛防御（Heavy Contract & Defense-in-Depth）**：
   - 引入了详尽的任务契约（Task Contract）、独立回执账本（Ledger Receipts）、本地回答校验与安全边界防护（Ultra/Approval 隔离）。
   - 优势是操作可控性强、审计链完整、避免虚假完成；代价是执行协议和 Prompt 块较为繁复。
2. **上下文感知与有界预算（Working Budget & Context Awareness）**：
   - 内置了 `working_budget` 触发压缩（Trigger 64k / Target 48k / Max 96k）、材料按需分页（`task_material_read`）以及执行路径的资源守卫（`max_bytes: 16MB`）。
3. **单向隔离的跨渠道与记忆体系（Cross-Channel & Projected Memory）**：
   - 渠道间（群聊/私聊/控制台）具备严格的物理单向隔离，近期沉淀了「近期跨渠道交互交接浮标（Recent Cross-Channel Handoff）」与针对主管群聊发言的白名单记忆沉淀机制。
4. **全功能宿主工具集合（Monolithic Tool Set）**：
   - 注册了 50 个工具，可用 46 个，涵盖 Shell、文件、浏览器、桌面交互、音视频转换、代码补丁及日志审计等。每次执行均在 System Prompt 中全量注入这些工具的 Schema 与规则。

---

## 二、 核心瓶颈与架构权衡（Trade-offs）

在实际运转中，目前的架构暴露出以下四个结构性痛点：

1. **静态全量工具投射导致的 Context 膨胀与决策噪声**：
   - 46 个工具的 Schema 和配套约束规则一次性注入上下文，占用了数千 Token 的固定开销。
   - 工具越多，大模型在选择工具时的「选择面扩散」越严重（容易出现多余尝试或因 Schema 歧义引发空输出/报错）。
2. **协议与防御层过厚，对底层模型的指令遵循度要求过高**：
   - 包含了大量防重、单次生效、本地回答格式等显式约束规则。在主力模型（如 `gemini-3.8-flash-tiered` 或较轻量模型）遇到复杂多步上下文时，容易产生格式摩擦或空回复（如历史记录中曾出现的 `conversation_output_empty`）。
3. **记忆体系的“被动抽取”与“冷热孤岛”**：
   - 现在的记忆系统主要依赖：① 静态 Markdown 文件（`MEMORY/`）；② 近期对话浮标提取。
   - 缺少记忆的**自动衰减机制**和**关联聚类（Consolidation）**能力。随着使用周期增加，人工维护 Markdown 索引的成本线性上升，而模型被动检索容易出现漏检或检索冗余。
4. **子系统职责分散（`shinku` / `aozora` / `goozam`）**：
   - `shinku` 是实际运行的核心 Agent 与工具宿主；
   - `aozora` 偏向确定性决策内核（Kernel / Decider / Contracts）；
   - `goozam` 偏向学习、经验上下文与离线模型网关（Offline / Learning / Model Gateway）。
   - 三者之间边界稍显割裂，部分概念（如模型运行时与上下文管理）在各子系统中有相似抽象，缺乏清晰的主从流转关系。

---

## 三、 四大方向优化建议（按 ROI 排序）

### 方向一：动态工具投射与按需激活（Dynamic Tool Projection）
- **优先级**：高（ROI 最高，立竿见影提升速度与降低 Token 消耗）
- **核心思路**：
  将工具按职责划分为 **Core（基础常驻）** 与 **Domain Extensions（领域插件）**：
  - **Core 级**（约 8–10 个）：`workspace_read_lines`, `workspace_search`, `workspace_preview_patch`, `workspace_apply_patch`, `private_shell`, `read_service_logs`, `read_self_state`。
  - **Domain 级**（按需加载）：
    - 浏览器集群（`browser_*`, `private_browser_*`, `tweet_*`）
    - 多媒体与下载集群（`private_media_*`, `private_download*`, `private_pdf_*`）
    - 外部查询与音乐（`private_web_search`, `private_netease_*`）
- **实现方式**：
  在 `agent_runtime/tool_context.py` 中引入任务路由或显式激活协议。当任务判定为纯工作区代码开发时，只下发 Core 工具集，将单轮 Prompt 减少 30%–50%，大幅降低模型选错工具的概率。

### 方向二：瘦身协议与回执摘要化（Lean Contract & Receipt Distillation）
- **优先级**：高（直接改善执行稳定性）
- **核心思路**：
  - **回执降维**：历史工具的执行结果（尤其是包含大量输出的 Shell stdout、日志块或长文本）在进入下一轮推理历史时，由运行时做自动摘要和证据提炼（如只保留执行状态、核心事实、关键 diff 与哈希），而非把多轮长输出原样堆叠在历史消息中。
  - **协议渐进式暴露**：区分「简单问答/诊断模式」与「高危写操作/批量任务模式」。简单只读场景免除繁重的 Task Contract Schema 约束，仅在高危写操作或长链路任务中动态激活契约状态机。

### 方向三：冷热分层与自动化聚合的“统一记忆总线”（Unified Memory Bus）
- **优先级**：中（决定长期陪伴与个性化体验的深度）
- **核心思路**：
  重构短期浮标与长期记忆的生命周期，形成三层流转管道：
  1. **L1 热记忆（Working Memory）**：当前任务与会话上下文 + 近期跨渠道交接浮标（保留当前优秀的单向隔离机制）。
  2. **L2 温记忆（Episodic Memory / 经验池）**：自动将完成的任务事实/主管偏好暂存在本地 SQLite/DuckDB（结构化包含：时间戳、主题、决策依据、有效性评分），定期在后台做归纳蒸馏。
  3. **L3 冷记忆（Semantic Knowledge / 规则与约束）**：即现在的 `AGENTS.md` 和 `MEMORY/*.md`，只保留最高权威的长期约定与全局规则。
  - **收益**：避免每次手工编写和更新 Markdown 记忆索引，使跨会话、跨任务的上下文更自然顺畅。

### 方向四：分级路由引擎（Tiered Fast-path / Deep-path Orchestration）
- **优先级**：中长线架构演进
- **核心思路**：
  结合 `goozam` 的模型网关与 `shinku` 的执行内核，建立双轨执行模型：
  - **Fast-path（敏捷快轨）**：针对简单闲聊、状态查询、只读搜索、轻量校验，直接由低延迟小模型走 Direct Reply 闭环，毫秒级响应。
  - **Deep-path（深度慢轨）**：针对复杂代码重构、跨模块架构变更、长故障排查，启动带完整任务契约、计划拆解与分级验证的慢思考循环。

---

## 四、 实施路径建议

建议采取**“先外围治理、再管线瘦身、后记忆总线”**的渐进演化策略：

1. **第一阶段（工具与 Prompt 瘦身）**：
   - 实施**方向一（动态工具投射）**：在 `tool_context.py` 实现白名单/模式化过滤，代码开发任务只挂载 Workspace + Shell 工具。
   - 验证指标：单轮 System Prompt Token 降低幅度、选工具错误率降低情况。
2. **第二阶段（历史回执摘要优化）**：
   - 优化 `native_messages.py` 与 `context.py` 中的历史消息装配逻辑，长输出回执保留核心事实与哈希指纹，防止上下文无意义打满。
3. **第三阶段（整合子系统与记忆机制）**：
   - 梳理 `goozam` / `aozora` 与 `shinku` 内部的重叠模块，建立统一的温记忆提取后台任务。
