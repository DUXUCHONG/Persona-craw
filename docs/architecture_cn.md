# Persona-craw — 架构

<p align="center">
  <a href="architecture.md"><img src="https://img.shields.io/badge/English-blue?style=for-the-badge" alt="English"></a>
  <a href="architecture_cn.md"><img src="https://img.shields.io/badge/中文-red?style=for-the-badge" alt="中文"></a>
</p>

## 1. 从 OpenClaw 到混合架构

### 1.1 OpenClaw 原始架构

OpenClaw / OpenClaw-RL 的核心思想：将 Agent Runtime 与学习基础设施解耦。

```
Agent Runtime → Experience → Reward → RL Training
```

系统概览：

```
Agent Runtime
     │
     ▼
  Gateway
     │
     ▼
Experience Store
     │
 ┌───┴─────┐
 ▼         ▼
Reward     Skill
Pipeline   Distill
 │          │
 ▼          ▼
 RL Trainer
```

核心特点：

- Runtime 和 Training **异步** —— 用户永远不会被学习过程阻塞。
- Experience 统一进入 **datapool**，不论来源。
- Reward Pipeline 从原始经验生成 **RL 信号**。
- Trainer 独立更新 **策略** 和蒸馏 **技能**。

这个架构很强大，但假设了全云端部署：所有组件都运行在云基础设施上。

### 1.2 纯本地 AI 的问题

一种常见的观点："AI Agent 在本地运行更安全、更好。" 但实际上，纯本地 AI 有严重的局限性。

**算力限制。** 消费级设备 GPU 不稳定、内存有限。长期 RL 训练不现实，持续训练会降低设备可用性。

**安全风险（经常被忽视）。** 拥有终端访问、浏览器自动化和文件系统控制权的本地 Agent，攻击面其实很大。如果模型被 prompt injection 攻击，它可以：

- 执行破坏性 shell 命令
- 窃取文件或凭证
- 运行未授权脚本

用户个人机器上的攻击面比沙箱化的云环境**更大**，而不是更小。

**数据孤岛。** 完全本地的 Agent 只能从单个用户的经验中学习。学习速度慢，经验无法共享，优化无法规模化。每个用户的 Agent 长期停留在低水平。

**更新困难。** 模型更新 —— 完整权重、LoRA 检查点、技能库修订 —— 都需要用户自行管理。对非技术用户不现实，对所有人都容易出错。

### 1.3 混合架构：核心思想

解决方案：**Local Claw + Cloud Claw**。

> 本地 Agent 执行并保护；云端 Agent 学习并协调。

- **本地**负责：任务执行、隐私保护、低延迟交互、权限隔离。
- **云端**负责：跨用户经验聚合、技能蒸馏、奖励建模、RL 训练、版本分发。

这解决了隐私/延迟（本地）与能力/规模（云端）之间的根本矛盾。

---

## 2. 系统架构

### 2.1 论文级系统图

```
                         ┌─────────────────────────────────────┐
                         │             Cloud Claw              │
                         │                                     │
                         │  ┌───────────────────────────────┐  │
                         │  │   Global Experience Store     │  │
                         │  │  trajectories / feedback /    │  │
                         │  │  metadata / anonymized logs   │  │
                         │  └───────────────┬───────────────┘  │
                         │                  │                  │
                         │      ┌───────────┴───────────┐      │
                         │      │                       │      │
                         │      ▼                       ▼      │
                         │  Reward Pipeline      Skill Distiller│
                         │  verifier / RM /      summarize /   │
                         │  implicit reward      merge / cluster│
                         │      │                       │      │
                         │      ▼                       ▼      │
                         │  RL / LoRA Trainer   Global Skill Bank
                         │      │                       │      │
                         │      └───────────┬───────────┘      │
                         │                  │                  │
                         │       Update Orchestrator           │
                         │  model versions / skill packs /     │
                         │  policy rules / rollout config      │
                         └──────────────────┬──────────────────┘
                                            ▲
                                            │ encrypted sync
                                            │ anonymized upload
                                            ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │                          Local Claw Node                            │
  │                                                                     │
  │  ┌────────────────────┐       ┌──────────────────────────────────┐  │
  │  │   Local Runtime    │──────▶│        Safety Guard Layer        │  │
  │  │ LLM / tools / UI   │       │ permission / sandbox / policy    │  │
  │  └─────────┬──────────┘       └───────────────┬──────────────────┘  │
  │            │                                  │                     │
  │            ▼                                  ▼                     │
  │  ┌────────────────────┐       ┌──────────────────────────────────┐  │
  │  │ Experience Buffer  │       │ Local Skill Cache / Retriever    │  │
  │  │ recent traces /    │       │ personalized skills / hot skills │  │
  │  │ feedback / results │       └──────────────────────────────────┘  │
  │  └─────────┬──────────┘                                             │
  │            │                                                        │
  │            ▼                                                        │
  │  ┌────────────────────┐                                             │
  │  │ Privacy Filter &   │                                             │
  │  │ Upload Scheduler   │                                             │
  │  │ redact / compress  │                                             │
  │  └────────────────────┘                                             │
  └──────────────────────────────────────────────────────────────────────┘
```

### 2.2 四层模块分解

系统分为四个概念层来理解。

**第一层 —— 交互层（Interaction Layer）。** 直接面对用户。包含 Chat UI、任务提交、日常对话反馈、本地 Agent Runtime。这一层的核心作用不是"训练"，而是把日常交互自然转成 experience。当用户说 *"这个不错"*、*"不是这个意思"*、*"你先搜 GitHub"*、*"太长了，短一点"* —— 这些生活化表达都进入经验缓冲区，后面变成 reward signal 或 skill evidence。

**第二层 —— 执行与安全层（Execution & Safety Layer）。** 主要在本地。包含终端执行器、浏览器控制器、文件访问管理器、权限系统、动作沙箱、策略守卫。这层很关键，因为本地 Agent 最大的风险不在"模型答错"，而在模型真的能**动你的电脑**。分离原则：Runtime 负责提出动作；Safety Guard 负责审批动作。

| 操作 | 默认策略 |
|------|---------|
| 浏览网页 | 允许 |
| 读取 ~/Downloads | 需要细粒度权限 |
| 删除文件 | 高风险 —— 拦截 |
| Shell 命令 | 必须过 policy check |

**第三层 —— 知识与经验层（Knowledge & Experience Layer）。** 本地和云端都有，但职责不同。

| | 本地 | 云端 |
|-|------|------|
| **存储** | 近期会话、个性化偏好、热技能缓存 | 全局经验、跨用户聚类 |
| **操作** | 快速检索，不依赖网络 | 技能合并、去重、重写、排名 |
| **目标** | 立即服务当前用户 | 让记忆随使用越来越抽象 |

关键洞察：memory 不能只是越存越多，而要越用越抽象。

**第四层 —— 学习与协调层（Learning & Coordination Layer）。** 主要在云端。包含 Reward Pipeline、Verifier、Reward Model、Skill Distillation、RL/LoRA Trainer、Update Orchestrator。这层解决两个问题：（1）如何把真实世界交互变成学习信号，（2）如何把学习结果安全地下发回本地。

---

## 3. 本地与云端组件划分

| 组件 | 本地 | 云端 | 原因 |
|------|:----:|:----:|------|
| Chat / Runtime | **是** | 否 | 低延迟、直接交互 |
| Browser / Terminal / IDE 执行 | **是** | 否 | 依赖本地环境与权限 |
| Safety Guard | **是** | 否 | 风险必须在执行前本地拦截 |
| Experience Buffer | **是** | 否 | 先本地暂存与脱敏 |
| Privacy Filter | **是** | 否 | 敏感数据应先在本地清洗 |
| Local Skill Cache | **是** | 否 | 降低时延、支持离线 |
| Personalized Preferences | **是** | 可选备份 | 用户私有偏好不宜全局共享 |
| Global Experience Store | 否 | **是** | 跨用户聚合学习 |
| Reward Pipeline | 否 | **是** | 计算重、依赖大模型或 judge |
| Skill Distiller | 否 | **是** | 需要全局归纳与去重 |
| Global Skill Bank | 否 | **是** | 统一版本管理 |
| RL / LoRA Trainer | 否 | **是** | GPU 与调度要求高 |
| Update Orchestrator | 否 | **是** | 统一发布模型与技能包 |

---

## 4. Local Claw 组件

Local Claw 负责**执行、隐私保护与即时交互**。

### 4.1 Agent Runtime

Agent 在本地运行，拥有对用户环境的访问权：

- 终端 / Shell
- 浏览器自动化
- IDE 集成
- 本地文件系统和工具
- Chat UI 和任务提交

这些操作需要本地权限和本地上下文，不能安全地委托给远程系统。Runtime 的核心作用不是训练 —— 它自然地将日常交互转化为经验。

### 4.2 Safety Guard Layer（安全守卫层）

最关键的本地组件。基本原则：**Runtime 提出动作；Safety Guard 审批动作。**

执行机制：

- **命令沙箱** —— Shell 命令在执行前检查白名单/黑名单
- **文件系统分区** —— 只读区域、高风险删除拦截
- **浏览器隔离** —— 敏感域名限制、沙箱化自动化上下文
- **上传拦截** —— 外发数据传输需要二次确认
- **策略守卫** —— 所有工具调用都经过 policy check 层

安全层独立于模型运行 —— 即使模型被攻击也无法绕过。

### 4.3 Experience Buffer（经验缓冲区）

本地系统在上传前记录结构化经验：

```
experience
 ├── task
 ├── context
 ├── actions
 ├── observations
 ├── outcome
 ├── user_feedback
 ├── tool_feedback
 └── risk_metadata
```

缓冲区在本地存储近期轨迹、反馈和结果。服务两个目的：（1）为本地模型的下次交互提供上下文，（2）暂存数据以待脱敏后上传云端。

### 4.4 Privacy Filter & Upload Scheduler（隐私过滤器与上传调度器）

任何经验到达云端之前：

- **PII 删除** —— 剥离个人标识符
- **凭证脱敏** —— 路径、token、cookie、邮箱等脱敏
- **长文本压缩** —— 冗长内容摘要化
- **高风险片段处理** —— 敏感片段丢弃或仅本地保留

到达云端的是：**可学习、可聚合、不可直接还原敏感现场**的经验版本。

上传采用调度模式（非实时），批量高效传输，避免打断用户。

### 4.5 Local Skill Cache / Retriever（本地技能缓存/检索器）

Global Skill Bank 的同步子集：

- **个性化技能** —— 匹配该用户过往行为的技能
- **热门技能** —— 全局高频使用技能
- **领域技能** —— 与用户活跃领域相关的技能

优势：减少云端请求、加快推理速度、支持离线使用。

### 4.6 Local Model（本地模型）

一个小型高效模型在本地运行，用于：

- 常规任务的低延迟交互
- 无需云端往返的简单任务完成
- 无网络连接时的离线模式

对于超出本地模型能力的复杂任务，系统**透明地回退到云端模型**。本地模型定期更新来自云端的新 LoRA 权重和技能库修订。

---

## 5. Cloud Claw 组件

Cloud Claw 负责**学习、智能与协调**。

### 5.1 Global Experience Store（全局经验存储）

所有 Local Claw 节点上传匿名化经验：

- 轨迹（状态-动作-观察序列）
- 用户反馈信号
- 任务结果和元数据
- 匿名化交互日志

聚合的优势：来自众多用户的数据使得更快的学习、经验共享和大规模优化成为可能 —— 这是单个用户的数据无法支撑的。

### 5.2 Reward Pipeline（奖励管线）

云端运行计算密集型的奖励构建：

- **Verifier** —— 任务结果和代码正确性的自动化验证
- **Reward Model (RM)** —— 训练好的分类器，用于自然语言反馈
- **Implicit reward extraction** —— 行为模式（重试、放弃、采纳）转化为信号

这些模型太大、太耗 GPU，无法在本地运行。

### 5.3 Skill Distiller（技能蒸馏器）

从聚合经验中提取可复用技能：

- **Summarize** —— 将轨迹压缩为结构化技能描述
- **Merge** —— 合并不同用户发现的相似技能
- **Rewrite** —— 提高技能的清晰度和通用性
- **Cluster** —— 按领域、复杂度和使用模式组织技能

多用户经验融合是关键优势：如果某个用户发现了更有效的编程策略，系统将其蒸馏为所有用户可用的技能。

### 5.4 RL / LoRA Trainer（强化学习训练器）

强化学习训练专门在云端运行。

需求：

- 大规模 GPU 集群
- 分布式 rollout 基础设施
- 训练编排和调度

训练输出：

- 更新的模型权重
- 新的 LoRA 适配器
- 修订的技能库条目
- 策略检查点

训练器基于全局经验运行，产生的改进惠及所有用户。

### 5.5 Update Orchestrator（更新编排器）

管理将学习结果分发回本地节点的完整生命周期。两类更新：

**Skill Pack（技能包）：**

- 通用技能
- 领域特定技能
- 用户相关技能候选（个性化排序）

**Model / Policy Update（模型/策略更新）：**

- LoRA 适配器权重
- Prompt 策略修订
- 工具策略规则
- 风险规则和安全策略更新
- 检索器配置

分发特性：模型版本管理、金丝雀发布（全面部署前在部分用户上测试）、检测到回归时自动回滚。

---

## 6. 数据流：完整闭环

系统的端到端闭环分为七个步骤。

### Step 1 —— 用户发起任务或日常对话

用户给出任务，或在对话中自然表达偏好。

例如：*"帮我订个下周去东京的行程"* / *"这次总结太长了"* / *"以后先看 GitHub 再回答"*

### Step 2 —— 本地 Runtime 执行

本地 Agent 调用工具执行任务，产生：

- 执行的动作
- 接收的观察
- 工具结果
- 中间推理轨迹（可选）
- 用户反应

### Step 3 —— Safety Guard 审批高风险动作

所有危险操作在本地被策略层拦截或降权：

- Shell 命令检查白名单/黑名单
- 文件系统只读区域强制执行
- 浏览器敏感域名限制
- 外发上传动作需要二次确认

### Step 4 —— Experience Buffer 记录原始经验

本地缓冲区写入结构化经验：

```
experience
 ├── task
 ├── context
 ├── actions
 ├── observations
 ├── outcome
 ├── user_feedback
 ├── tool_feedback
 └── risk_metadata
```

### Step 5 —— Privacy Filter 脱敏并上传

上传前执行：

- PII 删除
- 路径、token、cookie、邮箱等脱敏
- 长文本摘要压缩
- 高风险片段丢弃或本地保留

到达云端的是：可学习、可聚合、不可直接还原敏感现场的经验版本。

### Step 6 —— 云端学习

云端并行执行三条管线：

**A. Reward Pipeline** —— 从任务成败、用户反馈、行为模式中构建 reward。

**B. Skill Distillation** —— 从轨迹中抽取可复用 skill，做 merge / rewrite / rank。

**C. RL Training** —— 基于全局经验更新 policy / LoRA / small model adapter。

### Step 7 —— 云端下发更新

两类内容流回本地节点：

**Skill Pack（技能包）** —— 通用技能、领域技能、用户相关技能候选。

**Model / Policy Update（模型/策略更新）** —— LoRA 适配器、prompt 策略、工具策略、风险规则、检索器配置。

```
Step 1          Step 2          Step 3           Step 4
用户 ──────▶ Local Runtime ──▶ Safety Guard ──▶ Experience Buffer
                                                       │
                                                       ▼ Step 5
                                                 Privacy Filter
                                                       │
                                          匿名化上传    │
                                                       ▼
                                                 ┌───────────┐
Step 7                                           │   Cloud    │ Step 6
Skill Pack / Model Update ◀──────────────────── │  Learning  │
       │                                         └───────────┘
       ▼
  Local Claw
  （已更新）
```

---

## 7. 为什么混合架构更好

### 7.1 安全上更合理

很多人直觉上觉得"本地更安全"，但真正执行危险动作的是本地环境本身。安全的关键不是**模型在哪**，而是**执行权限是否被本地 guard 严格控制**。

混合架构把**学习能力**放到云端，但把**执行审批权**留在本地。这是更稳的安全姿态：云端模型无法直接执行 shell 命令，本地安全层不会被模型绕过。

### 7.2 学习上更高效

纯本地只能学自己的经验。混合系统聚合很多用户的非敏感经验，形成：

- 更强的全局 skill bank
- 更稳定的 reward model
- 更快的策略改进

集体智慧 —— 每个用户贡献经验，每个用户从聚合中受益。

### 7.3 体验上更平衡

| 任务类型 | 处理方 | 好处 |
|---------|--------|------|
| 简单/常规 | 本地模型 | 低延迟，无云端成本 |
| 复杂/新颖 | 云端模型 | 完整能力 |
| 离线 | 本地模型 + 技能缓存 | 基本能力保留 |
| 在线 | 本地 + 云端协同 | 持续进化 |

### 7.4 对比总表

| 关注点 | 纯本地 | 纯云端 | 混合架构（Persona-craw） |
|--------|--------|--------|------------------------|
| **隐私** | 敏感数据留在本地 | 数据离开用户设备 | 敏感数据留在本地；仅上传匿名化经验 |
| **安全** | 用户机器攻击面大 | 云端沙箱限制影响范围 | 本地安全层 + 云端沙箱；执行审批权留在本地 |
| **学习速度** | 慢 —— 仅单用户数据 | 快 —— 但无个性化 | 快 —— 聚合经验 + 个人微调 |
| **延迟** | 简单任务低延迟 | 每次请求都有网络往返 | 常规任务低延迟；仅需要时走云端 |
| **成本** | 计算免费但能力有限 | 每次请求都贵 | 优化 —— 本地处理常规，云端处理复杂 |
| **进化** | 手动更新，无共享学习 | 持续但千人一面 | 持续 + 个性化 —— 兼得两者之长 |

**总结：**

> 本地 Agent **执行**。云端 Agent **学习**。

---

## 8. 实现目录结构

```
persona-craw/
├── serving/
│   ├── runtime/
│   │   ├── agent.py
│   │   ├── tool_manager.py
│   │   ├── trajectory_recorder.py
│   │   └── session.py
│   └── gateway/
│       ├── proxy.py
│       ├── skill_injector.py
│       ├── trace_capture.py
│       └── openai_compat.py
├── data/
│   ├── experience_store/
│   │   ├── hot_tier.py
│   │   ├── cold_tier.py
│   │   ├── schema.py
│   │   └── migration.py
│   └── skill_bank/
│       ├── store.py
│       ├── retrieval.py
│       ├── embedding.py
│       ├── hierarchy.py
│       └── schema.py
├── learning/
│   ├── reward/
│   │   ├── heuristic_reward.py
│   │   ├── learned_reward.py
│   │   ├── outcome_reward.py
│   │   ├── reward_fusion.py
│   │   └── credit_assignment.py
│   ├── skill_distillation/
│   │   ├── pattern_detector.py
│   │   ├── memory_consolidator.py
│   │   ├── skill_extractor.py
│   │   ├── skill_evolution.py
│   │   └── skill_promotion.py
│   └── trainer/
│       ├── engine.py
│       ├── grpo.py
│       ├── dpo.py
│       ├── lora_rl.py
│       ├── sft.py
│       └── backends/
│           ├── tinker.py
│           ├── mint.py
│           └── local.py
├── ops/
│   ├── scheduler/
│   │   ├── deploy_scheduler.py
│   │   ├── training_window.py
│   │   ├── canary_rollout.py
│   │   └── rollback.py
│   └── evaluation/
│       ├── online_eval.py
│       ├── offline_benchmark.py
│       ├── skill_quality.py
│       └── ab_test.py
├── hybrid/
│   ├── local_claw/
│   │   ├── skill_cache.py
│   │   ├── safety_guard.py
│   │   ├── experience_collector.py
│   │   ├── anonymizer.py
│   │   └── local_model.py
│   └── cloud_claw/
│       ├── experience_aggregator.py
│       ├── global_skill_bank.py
│       ├── model_distributor.py
│       ├── canary_rollout.py
│       └── checkpoint_manager.py
├── routing/
│   ├── task_router.py
│   ├── complexity_estimator.py
│   └── cloud_local_bridge.py
├── master_apprentice/
│   ├── master_explorer.py
│   ├── knowledge_distiller.py
│   ├── apprentice_executor.py
│   └── escalation.py
└── config/
    ├── default.yaml
    ├── reward.yaml
    ├── training.yaml
    └── routing.yaml
```
