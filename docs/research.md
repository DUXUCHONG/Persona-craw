# Persona-craw — Research

## Self-Evolving Agents: Reinforcement Learning from Daily Words

Persona-craw investigates how AI agents can continuously improve by learning from everyday natural language interaction.

The core thesis:

> Everyday words are implicit reward. Daily conversation is online RL.

Current LLM agents are static after deployment. User corrections, preferences, and feedback are discarded at the end of each session. We propose a system that treats daily interaction as the primary training signal — converting natural language into rewards, distilling experience into skills, and updating agent policy through online reinforcement learning.

The research is organized into five directions.

---

## 1. Skill System

### How does an agent extract, evolve, and compose reusable skills from daily interaction?

Every day, users correct, guide, and refine the agent's behavior through natural conversation. These interactions contain repeating patterns — preferred workflows, common corrections, effective strategies. The Skill System turns these patterns into structured, reusable knowledge.

### 1.1 Skill Extraction from Daily Use

When a user says "always use pytest" three times across different projects, that's not just a correction — it's a skill waiting to be distilled.

**Research questions:**

- How to detect recurring patterns across conversations and extract them as structured skills (trigger condition, strategy steps, anti-patterns)?
- How to distinguish genuine skills from noise — one-off corrections vs. persistent preferences?
- How to extract skills from both success ("that worked perfectly") and failure ("don't ever do that again")?

### 1.2 Skill Evolution through RL

Skills should not be static. As the agent accumulates more experience, skills must evolve.

**Research questions:**

- How to use reinforcement learning to refine skill quality over time? A skill that leads to user approval gets reinforced; one that gets corrected gets updated.
- How to handle skill conflicts — when a new preference contradicts an old skill?
- How to deprecate skills that no longer match user behavior?

### 1.3 Skill Hierarchy and Composition

Not all skills are at the same level. Some are universal principles; others are narrow task-specific tricks. Complex tasks require composing multiple skills together.

**Research questions:**

- How to organize skills into layers: general principles, domain strategies, task-specific procedures?
- How to compose multiple skills for complex tasks — e.g., combining a "code style" skill with a "testing strategy" skill when scaffolding a project?
- How to promote successful task-specific skills to domain-level or general-level over time?

**Related work:** SkillRL (trajectory-to-skill distillation, hierarchical SkillBank, recursive evolution), SAGE (skills in the RL training loop), MetaClaw (skill retrieval and prompt injection).

---

## 2. Context and Memory Transformation

### How does an agent compress long interaction history into usable knowledge?

An agent that talks to a user every day for months accumulates massive interaction history. Storing raw conversation as memory leads to context explosion, retrieval noise, and degraded performance.

The key insight: **memory should not be what happened, but what was learned.**

**Research questions:**

- **Context-to-skill transformation.** How to convert raw conversational context (corrections, preferences, feedback scattered across hundreds of sessions) into compact skill representations that fit within context budgets?

- **Memory consolidation.** How to merge, compress, and summarize related memories into unified skills — similar to how humans consolidate episodic memory into procedural knowledge during sleep?

- **Selective forgetting.** Not all history is valuable. How to identify and discard outdated preferences, superseded corrections, and low-signal interactions?

- **Retrieval under budget.** Given a fixed context window, how to allocate tokens optimally between the current task, retrieved skills, and recent conversation history?

- **Dynamic context switching.** How to transition between skill-based memory (long-term knowledge) and episodic memory (recent conversation) depending on task demands?

**Related work:** SkillRL (SkillBank as structured memory), MetaClaw (semantic skill retrieval with embedding search), context distillation in long-context LLMs.

---

## 3. Master-Apprentice Learning

### How can a strong model teach a small model to be effective?

Running a large frontier model for every user interaction is expensive. But small models lack the reasoning capability to handle complex tasks alone. The Master-Apprentice paradigm bridges this gap.

The **master** (strong model) explores difficult tasks, discovers effective execution paths, and records both successes and failures. This knowledge is then distilled and made available to the **apprentice** (small, cheap model) for daily execution.

### 3.1 Master Exploration

The strong model handles complex, novel, or ambiguous tasks. It explores multiple approaches, evaluates outcomes, and identifies what works and what doesn't.

**Research questions:**

- How should the master record its exploration? Raw trajectories? Distilled skills? Annotated decision trees? What representation transfers best to the apprentice?
- How to capture failure reasons alongside success strategies — so the apprentice learns not just what to do, but what to avoid?

### 3.2 Knowledge Distillation Formats

The master's knowledge must be packaged for the apprentice. Several formats are possible:

- **Skills** — structured strategies with trigger conditions and steps. Compact, reusable, but may lose nuance.
- **Trajectories** — full execution paths. Rich in detail, but expensive to store and retrieve.
- **Decision annotations** — key decision points with rationale. Balanced between detail and compression.
- **Negative examples** — explicit failure cases with root cause analysis. Prevents the apprentice from repeating mistakes.

**Research questions:**

- Which distillation format produces the best apprentice performance per token of context budget?
- How to combine formats — e.g., skills for routine tasks, annotated trajectories for complex ones?

### 3.3 Apprentice Execution

The small model handles daily interaction using distilled knowledge from the master.

**Research questions:**

- How does the apprentice decide when to use a master-distilled skill vs. its own reasoning?
- When should the apprentice escalate to the master — i.e., recognize it's out of its depth?
- How to continuously update the apprentice's knowledge base as the master explores new territory?

**Related work:** On-policy distillation (OPD), knowledge distillation in LLMs, MetaClaw (RL + distillation modes), speculative decoding as a lightweight master-apprentice pattern.

---

## 4. RL by Daily Words

### How to turn everyday conversation into a reinforcement learning training loop?

This is the central technical challenge of Persona-craw. Users do not provide structured rewards. They express satisfaction, correction, and preference through natural language. The system must convert these signals into RL training.

### 4.1 Daily Words to RL Rewards

Users speak naturally:

- *"Perfect."* — positive reward
- *"No, not like that."* — negative reward
- *"I prefer shorter answers."* — preference signal
- *"Try the other API."* — corrective signal
- *"That's wrong again."* — repeated failure penalty
- Silence / continuing the task — weak positive

**Research questions:**

- How to classify natural language utterances into reward categories (approval, correction, preference, frustration, neutral) reliably?
- How to handle ambiguity — "interesting" could be positive or dismissive?
- How to propagate delayed rewards — a correction in turn 8 may refer to a decision in turn 2?
- How to extract implicit behavioral signals beyond language: retry patterns, session abandonment, output adoption rates?

### 4.2 RL Training Algorithms

Once rewards are extracted, they must drive policy updates.

**Research questions:**

- Which RL algorithms are best suited for sparse, noisy, natural-language-derived rewards? Candidates include GRPO (group relative policy optimization), DPO (direct preference optimization), and online PPO variants.
- How to handle extremely small batch sizes — daily interaction produces far less data than synthetic RL environments?
- How to balance exploration (trying new strategies) vs. exploitation (sticking with what the user approved) in a real-user setting?
- How to do step-level credit assignment when reward is given at the conversation level?

### 4.3 RL Infrastructure

Online RL from daily interaction requires production-grade infrastructure.

**Research questions:**

- How to build a conversation-to-training pipeline that converts live interaction into (state, action, reward) tuples without blocking response latency?
- How to decouple the training loop from the serving loop — so RL updates don't interrupt the user experience?
- How to schedule training safely: idle-time training, night training, calendar-window training?
- How to implement regression testing and automatic rollback when a training update degrades performance?

**Related work:** OpenClaw-RL (async online RL loop), Claw-R1 (Gateway/DataPool architecture), MetaClaw (idle-window updates), DeepSeek-R1 (GRPO), Tinker / MinT (distributed RL backends).

---

## 5. Local-Cloud Hybrid: Small LoRA-RL + Large Cloud Model

### How to combine a locally fine-tuned small model with a cloud-based large model?

Running everything on a frontier cloud model is expensive. Running everything on a local small model sacrifices quality. The hybrid approach combines both:

- A **local small model** (fine-tuned with LoRA-RL from the user's daily interaction) handles routine tasks fast and cheap, personalized to the user.
- A **cloud large model** handles complex tasks that exceed the local model's capability, and serves as the "master" for distillation.

### 5.1 Local LoRA-RL Personalization

The local model is continuously fine-tuned on the user's interaction using LoRA (low-rank adaptation) + RL.

**Research questions:**

- How to train LoRA adapters from daily conversation rewards efficiently on consumer hardware?
- How to prevent catastrophic forgetting — ensuring personalization doesn't destroy the model's general capability?
- How to manage LoRA checkpoint versioning and rollback when a training run degrades quality?

### 5.2 Routing: When Local, When Cloud?

Not every request needs a large model. Not every request can be handled by a small one.

**Research questions:**

- How to build a task router that decides whether to use the local model or escalate to the cloud model?
- What signals indicate the local model is out of its depth — uncertainty estimation, task complexity heuristics, domain mismatch?
- How to minimize cloud API cost while maintaining quality — what's the optimal routing threshold?

### 5.3 Cloud-to-Local Knowledge Transfer

When the cloud model handles a complex task, its execution should feed back into the local model's training.

**Research questions:**

- How to distill the cloud model's successful execution into LoRA training data for the local model?
- Over time, can the local model absorb enough knowledge to handle tasks that previously required cloud escalation — i.e., does the local model's capability boundary expand?
- How to measure and track this capability expansion over time?

**Related work:** Speculative decoding, mixture-of-experts routing, on-device fine-tuning (Apple MLX, llama.cpp), LoRA composition and merging techniques.

---

## Research Roadmap

### Phase 1 — Skill Foundation

- Skill extraction from daily conversation
- Hierarchical skill bank with retrieval and injection
- Context-to-skill transformation (memory compression)
- Baseline evaluation: skill accumulation without RL

### Phase 2 — Daily Words Reward Pipeline

- Natural language reward classifier
- Implicit behavioral signal extraction
- Reward fusion (language + behavior + task outcome)
- Credit assignment for delayed, conversation-level feedback

### Phase 3 — Master-Apprentice + RL Training

- Master (strong model) exploration and knowledge recording
- Knowledge distillation: skills, trajectories, failure annotations
- Apprentice (small model) execution with distilled knowledge
- Online RL training from daily word rewards (GRPO / DPO / PPO)
- Skill evolution through RL feedback loop

### Phase 4 — Local-Cloud Hybrid + Autonomous Evolution

- Local LoRA-RL personalization on consumer hardware
- Task routing between local and cloud models
- Cloud-to-local knowledge transfer pipeline
- Safe update scheduling (night / idle / calendar)
- Continual benchmarking and regression detection
- Fully autonomous self-evolving agent system

---

## References

- SkillRL — Skill distillation and hierarchical SkillBank ([arXiv 2602.08234](https://arxiv.org/abs/2602.08234))
- SAGE — Skill-augmented RL ([arXiv 2512.17102](https://arxiv.org/abs/2512.17102))
- DreamGym — Agent-level experience synthesis ([arXiv 2511.03773](https://arxiv.org/abs/2511.03773))
- Agent Lightning — Training-agent disaggregation ([arXiv 2508.03680](https://arxiv.org/abs/2508.03680))
- OpenClaw-RL — Async online RL for LLM agents
- Claw-R1 — Gateway/DataPool middleware
- MetaClaw — Skill injection and idle-window model updates
- MinT — Lightweight online training backend
- Tinker — Distributed RL training orchestration
- DeepSeek-R1 — GRPO ([arXiv 2501.12948](https://arxiv.org/abs/2501.12948))

Survey: [Awesome Agentic RL](https://github.com/DUXUCHONG/Awesome-Agentic-RL)
