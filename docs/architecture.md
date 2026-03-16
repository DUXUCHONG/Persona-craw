# Persona-craw — Architecture

<p align="center">
  <a href="architecture.md"><img src="https://img.shields.io/badge/English-blue?style=for-the-badge" alt="English"></a>
  <a href="architecture_cn.md"><img src="https://img.shields.io/badge/中文-red?style=for-the-badge" alt="中文"></a>
</p>

## 1. From OpenClaw to Hybrid Architecture

### 1.1 OpenClaw Original Architecture

OpenClaw / OpenClaw-RL's core idea: decouple agent runtime from learning infrastructure.

```
Agent Runtime → Experience → Reward → RL Training
```

System overview:

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

Key properties:

- Runtime and training are **asynchronous** — the user is never blocked by learning.
- Experience enters a **unified datapool** regardless of source.
- Reward pipeline generates **RL signal** from raw experience.
- Trainer updates **policy** and distills **skills** independently.

This architecture is powerful but assumes a cloud-centric deployment: all components run on cloud infrastructure.

### 1.2 The Problem with Pure Local AI

A common belief: "running AI agents locally is safer and better." In practice, pure local AI has critical limitations.

**Compute constraints.** Consumer devices have limited, unstable GPU resources and memory. Long-horizon RL training is impractical. Sustained training degrades device usability.

**Security risks (often overlooked).** A local agent with terminal access, browser automation, and file system control has a large attack surface. If the model is compromised by prompt injection, it can:

- Execute destructive shell commands
- Exfiltrate files or credentials
- Run unauthorized scripts

The attack surface on a user's personal machine is *larger*, not smaller, than a sandboxed cloud environment.

**Data isolation.** Fully local agents learn only from one user's experience. Learning is slow, experiences cannot be shared, and optimization cannot scale. Each user's agent stays at a low capability level indefinitely.

**Update burden.** Model updates — full weights, LoRA checkpoints, skill bank revisions — must be managed by the user. This is impractical for non-technical users and error-prone for everyone.

### 1.3 Hybrid Architecture: Core Idea

The solution: **Local Claw + Cloud Claw**.

> Local agents execute and protect; cloud agents learn and coordinate.

- **Local** handles: task execution, privacy protection, low-latency interaction, permission isolation.
- **Cloud** handles: cross-user experience aggregation, skill distillation, reward modeling, RL training, version distribution.

This resolves the fundamental tension between privacy/latency (local) and capability/scale (cloud).

---

## 2. System Architecture

### 2.1 Paper-Level System Diagram

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

### 2.2 Four-Layer Module Decomposition

The system is organized into four conceptual layers.

**Layer 1 — Interaction Layer.** Directly faces the user. Contains chat UI, task submission, daily conversation feedback, and local agent runtime. This layer's purpose is not training — it naturally converts daily interaction into experience. When a user says *"that's good,"* *"not what I meant,"* *"search GitHub first,"* or *"too long, shorter"* — these everyday expressions enter the experience buffer as future reward signals or skill evidence.

**Layer 2 — Execution & Safety Layer.** Primarily local. Contains terminal executor, browser controller, file access manager, permission system, action sandbox, and policy guard. This layer is critical because the biggest risk of a local agent is not wrong answers — it's that the model can *act on your machine*. The separation principle: Runtime proposes actions; Safety Guard approves actions.

| Action | Default Policy |
|--------|---------------|
| Browse web | Allow |
| Read ~/Downloads | Fine-grained permission required |
| Delete files | High-risk — intercept |
| Shell command | Must pass policy check |

**Layer 3 — Knowledge & Experience Layer.** Spans both local and cloud with different responsibilities.

| | Local | Cloud |
|-|-------|-------|
| **Stores** | Recent sessions, personal preferences, hot skill cache | Global experience, cross-user clusters |
| **Operations** | Fast retrieval, no network dependency | Skill merge, dedup, rewrite, ranking |
| **Goal** | Serve the current user immediately | Make memory increasingly abstract over time |

The key insight: memory must not just accumulate — it must become more abstract with use.

**Layer 4 — Learning & Coordination Layer.** Primarily cloud. Contains reward pipeline, verifier, reward model, skill distillation, RL/LoRA trainer, and update orchestrator. This layer solves two problems: (1) how to turn real-world interaction into learning signal, and (2) how to safely distribute learning results back to local nodes.

---

## 3. Local–Cloud Component Mapping

| Component | Local | Cloud | Rationale |
|-----------|:-----:|:-----:|-----------|
| Chat / Runtime | **yes** | no | Low latency, direct interaction |
| Browser / Terminal / IDE execution | **yes** | no | Depends on local environment and permissions |
| Safety Guard | **yes** | no | Risk must be intercepted locally before execution |
| Experience Buffer | **yes** | no | Local staging and sanitization first |
| Privacy Filter | **yes** | no | Sensitive data must be cleaned locally |
| Local Skill Cache | **yes** | no | Reduce latency, enable offline |
| Personalized Preferences | **yes** | optional backup | User-private preferences should not be globally shared |
| Global Experience Store | no | **yes** | Cross-user aggregation for learning |
| Reward Pipeline | no | **yes** | Compute-heavy, depends on large models or judges |
| Skill Distiller | no | **yes** | Requires global induction and deduplication |
| Global Skill Bank | no | **yes** | Unified version management |
| RL / LoRA Trainer | no | **yes** | High GPU and orchestration requirements |
| Update Orchestrator | no | **yes** | Unified release of model and skill packs |

---

## 4. Local Claw Components

The Local Claw is responsible for **execution, privacy, and immediate interaction**.

### 4.1 Agent Runtime

The agent runs locally with access to the user's environment:

- Terminal / shell
- Browser automation
- IDE integration
- Local file system and tools
- Chat UI and task submission

These operations require local permissions and local context. They cannot be safely delegated to a remote system. The runtime's core role is not training — it naturally converts daily interaction into experience.

### 4.2 Safety Guard Layer

The most critical local component. The fundamental principle: **Runtime proposes actions; Safety Guard approves them.**

Enforcement mechanisms:

- **Command sandbox** — shell commands checked against allowlist/blocklist before execution
- **File system zones** — read-only regions, high-risk deletion interception
- **Browser isolation** — sensitive domain restrictions, sandboxed automation context
- **Upload interception** — outbound data transfers require secondary confirmation
- **Policy guard** — all tool invocations pass through a policy check layer

The safety guard operates independently of the model — even a compromised model cannot bypass it.

### 4.3 Experience Buffer

The local system records structured experience before any upload:

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

The buffer stores recent traces, feedback, and results locally. This serves two purposes: (1) providing context for the local model's next interaction, and (2) staging data for eventual cloud upload after sanitization.

### 4.4 Privacy Filter & Upload Scheduler

Before any experience reaches the cloud:

- **PII removal** — personal identifiers stripped
- **Credential redaction** — paths, tokens, cookies, emails sanitized
- **Long-text compression** — verbose content summarized
- **High-risk fragment handling** — sensitive segments discarded or retained locally only

What reaches the cloud is a *learnable, aggregatable version that cannot reconstruct the sensitive original*.

Upload is scheduled (not real-time) to batch efficiently and avoid interrupting the user.

### 4.5 Local Skill Cache / Retriever

A synchronized subset of the Global Skill Bank:

- **Personalized skills** — skills that matched this user's past behavior
- **Hot skills** — frequently used skills across the population
- **Domain skills** — skills relevant to the user's active domains

Benefits: reduced cloud requests, faster inference, offline operation.

### 4.6 Local Model

A small, efficient model runs locally for:

- Low-latency interaction on routine tasks
- Simple task completion without cloud roundtrip
- Offline mode when connectivity is unavailable

For complex tasks that exceed local model capability, the system **falls back to the cloud model** transparently. The local model is periodically updated with new LoRA weights and skill bank revisions from the cloud.

---

## 5. Cloud Claw Components

The Cloud Claw is responsible for **learning, intelligence, and coordination**.

### 5.1 Global Experience Store

All Local Claw nodes upload anonymized experience:

- Trajectories (state-action-observation sequences)
- User feedback signals
- Task outcomes and metadata
- Anonymized interaction logs

The aggregation advantage: data from many users enables faster learning, experience sharing, and large-scale optimization that no single user's data could support.

### 5.2 Reward Pipeline

The cloud runs compute-intensive reward construction:

- **Verifier** — automated checking of task results and code correctness
- **Reward model (RM)** — trained classifiers for natural language feedback
- **Implicit reward extraction** — behavioral patterns (retry, abandon, adopt) converted to signal

These models are too large and GPU-hungry to run locally.

### 5.3 Skill Distiller

Extracts reusable skills from aggregated experience:

- **Summarize** — compress trajectories into structured skill descriptions
- **Merge** — combine similar skills discovered by different users
- **Rewrite** — improve skill clarity and generality
- **Cluster** — organize skills by domain, complexity, and usage pattern

Multi-user experience fusion is a key advantage: if one user discovers an effective coding strategy, the system distills it into a skill available to all users.

### 5.4 RL / LoRA Trainer

Reinforcement learning training runs exclusively on the cloud.

Requirements:

- Large-scale GPU clusters
- Distributed rollout infrastructure
- Training orchestration and scheduling

Training outputs:

- Updated model weights
- New LoRA adapters
- Revised skill bank entries
- Policy checkpoints

The trainer operates on global experience, producing improvements that benefit all users.

### 5.5 Update Orchestrator

Manages the full lifecycle of distributing learning results back to local nodes. Two categories of updates:

**Skill Packs:**

- General-purpose skills
- Domain-specific skills
- User-relevant skill candidates (personalized ranking)

**Model / Policy Updates:**

- LoRA adapter weights
- Prompt policy revisions
- Tool policy rules
- Risk rules and safety policy updates
- Retriever configuration

Distribution features: model versioning, canary rollout (test on subset before full deployment), automatic rollback if regression detected.

---

## 6. Data Flow: The Complete Loop

The system's end-to-end closed loop operates in seven steps.

### Step 1 — User initiates task or daily conversation

The user issues a task or naturally expresses preferences in conversation.

Examples: *"Plan a trip to Tokyo next week"* / *"That summary was too long"* / *"Search GitHub first before answering"*

### Step 2 — Local Runtime executes

The local agent invokes tools and executes the task, producing:

- Actions taken
- Observations received
- Tool results
- Intermediate reasoning traces (optional)
- User reaction

### Step 3 — Safety Guard approves high-risk actions

All dangerous operations are intercepted or downgraded by the local policy layer:

- Shell commands checked against allowlist / blocklist
- File system read-only zones enforced
- Sensitive browser domains restricted
- Outbound upload actions require secondary confirmation

### Step 4 — Experience Buffer records raw experience

The local buffer writes structured experience:

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

### Step 5 — Privacy Filter sanitizes and uploads

Before upload:

- PII removal
- Path, token, cookie, email redaction
- Long-text summarization and compression
- High-risk fragments discarded or retained locally

What reaches the cloud: a learnable, aggregatable version that cannot reconstruct the sensitive original.

### Step 6 — Cloud learns

The cloud runs three parallel pipelines:

**A. Reward Pipeline** — constructs reward from task success/failure, user feedback, and behavioral patterns.

**B. Skill Distillation** — extracts reusable skills from trajectories; merges, rewrites, and ranks across users.

**C. RL Training** — updates policy / LoRA / small model adapter based on global experience.

### Step 7 — Cloud distributes updates

Two categories flow back to local nodes:

**Skill Packs** — general skills, domain skills, user-relevant skill candidates.

**Model / Policy Updates** — LoRA adapters, prompt policies, tool policies, risk rules, retriever configuration.

```
Step 1          Step 2          Step 3           Step 4
User ──────▶ Local Runtime ──▶ Safety Guard ──▶ Experience Buffer
                                                       │
                                                       ▼ Step 5
                                                 Privacy Filter
                                                       │
                                          anonymized   │
                                          upload       ▼
                                                 ┌───────────┐
Step 7                                           │   Cloud    │ Step 6
Skill Packs / Model Updates ◀─────────────────── │  Learning  │
       │                                         └───────────┘
       ▼
  Local Claw
  (updated)
```

---

## 7. Why Hybrid Is Better

### 7.1 Security

Many people intuitively feel "local is safer," but the entity that actually executes dangerous actions is the local environment itself. The real security question is not *where the model runs* but *whether execution permissions are strictly controlled by a local guard*.

The hybrid architecture places **learning capability** in the cloud but keeps **execution approval authority** local. This is a more stable security posture: the cloud model cannot directly execute shell commands, and the local safety guard cannot be bypassed by the model.

### 7.2 Learning Efficiency

Pure local agents learn only from their own experience. The hybrid system aggregates non-sensitive experience from many users, producing:

- A stronger global skill bank
- A more stable reward model
- Faster policy improvement

Collective intelligence — every user contributes experience, every user benefits from the aggregate.

### 7.3 User Experience

| Task Type | Handled By | Benefit |
|-----------|-----------|---------|
| Simple / routine | Local model | Low latency, no cloud cost |
| Complex / novel | Cloud model | Full capability |
| Offline | Local model + skill cache | Basic capability preserved |
| Online | Local + cloud coordination | Continuous evolution |

### 7.4 Comparison Table

| Concern | Pure Local | Pure Cloud | Hybrid (Persona-craw) |
|---------|-----------|------------|----------------------|
| **Privacy** | Sensitive data stays local | Data leaves user device | Sensitive data stays local; only anonymized experience uploaded |
| **Security** | Large attack surface on user machine | Cloud sandbox limits blast radius | Local safety guard + cloud sandbox; execution approval stays local |
| **Learning speed** | Slow — single user data only | Fast — but no personalization | Fast — aggregated experience + personal fine-tuning |
| **Latency** | Low for simple tasks | Network roundtrip on every request | Low for routine tasks; cloud only when needed |
| **Cost** | Free compute but limited capability | Expensive for every request | Optimized — local handles routine, cloud handles complex |
| **Evolution** | Manual updates, no shared learning | Continuous but one-size-fits-all | Continuous + personalized — best of both |

**Summary:**

> Local agents **act**. Cloud agents **learn**.

---

## 8. Implementation Structure

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
