# Persona-craw — Research

## Self-Evolving Agents: Reinforcement Learning from Daily Words

Persona-craw investigates a fundamental question in agent learning:

**Can AI agents continuously improve their policy by treating everyday natural language interaction as a reinforcement learning signal?**

Current LLM agents are static after deployment. Regardless of how many conversations they process, they begin every session from the same fixed policy. User corrections, preferences, and feedback are discarded at the end of each session.

We propose a system that closes this gap by treating **daily conversational interaction** as the primary source of training signal. The user's natural language — corrections, approvals, preferences, frustrations — is converted into reward signals that drive online policy updates through reinforcement learning.

The core thesis:

> Everyday words are implicit reward. Daily conversation is online RL.

---

## Research Problems

The system is organized around four core research questions.

---

### 1. RL from Daily Words: Implicit Reward Extraction

**Central Question:** How can natural language feedback be converted into reward signals for reinforcement learning?

Users do not provide structured feedback. They do not click thumbs-up or rate responses on a scale. Instead, they express satisfaction, correction, and preference through natural language:

- *"Perfect."* — positive reward
- *"No, not like that."* — negative reward
- *"I prefer shorter answers."* — preference signal
- *"Try the other API."* — corrective signal
- *"That's wrong again."* — repeated failure penalty
- Silence / moving on — weak positive or neutral

The research challenge is to build a **reward extraction pipeline** that reliably maps these diverse, noisy, and often ambiguous natural language signals into scalar or structured reward values suitable for RL training.

**Research Directions:**

- **Natural language reward classification.** Classifying user utterances into reward categories (approval, correction, preference, frustration, neutral) using lightweight models or LLM-based judges.

- **Implicit behavioral signals.** Beyond explicit language, reward can be inferred from user behavior: whether the user accepts or rejects a suggestion, retries a task, continues a workflow, or abandons the session. These behavioral patterns provide additional training signal.

- **Delayed and sparse reward.** Users may not react immediately. A correction in turn 8 may refer to a decision made in turn 2. Propagating reward to the correct earlier action requires temporal credit assignment over long, partially observable episodes.

- **Reward noise and ambiguity.** Natural language is inherently ambiguous. "That's interesting" may be positive or dismissive. The system must handle noisy reward signals robustly, avoiding overfitting to misinterpreted feedback.

**Related Work:** OpenClaw-RL demonstrates that next-state feedback can serve as online learning signal. RLAIF explores AI-generated feedback as a substitute for human annotation. Process reward models (PRM) provide step-level reward signals for reasoning tasks.

---

### 2. Experience-to-Skill Abstraction

**Central Question:** How can raw interaction trajectories be transformed into reusable, transferable skills?

Storing past interactions as raw memory leads to context explosion and retrieval inefficiency. A more principled approach treats daily experience not as data to be recalled, but as material to be **abstracted** into structured strategies.

When a user repeatedly corrects the same pattern — "always use pytest," "keep it concise," "check the staging server first" — the system should distill these corrections into a persistent **skill** that the agent applies proactively in future tasks.

**Research Directions:**

- **Skill distillation from conversational feedback.** Extracting structured skills (trigger condition, strategy, anti-pattern) from clusters of natural language corrections and preferences.

- **Hierarchical skill organization.** General skills ("verify before executing irreversible actions") apply everywhere. Domain skills ("use pytest for Python projects") apply within specific contexts. Task-specific skills ("this user wants experiment-heavy paper summaries") apply narrowly. The system must support this hierarchy.

- **Skill evolution.** As more daily interactions accumulate, existing skills should be refined, merged when redundant, and deprecated when contradicted by newer feedback. This recursive evolution keeps the skill bank current.

- **Skill quality assessment.** Not all extracted skills are useful. Some may encode noise or overly specific patterns. Methods for evaluating skill transferability and filtering low-value skills are needed.

**Related Work:** SkillRL introduces trajectory-to-skill distillation with hierarchical SkillBank and recursive evolution. SAGE integrates skills into the RL training loop. MetaClaw implements skill retrieval and injection via system prompt augmentation.

---

### 3. Agent Memory Compression

**Central Question:** How can long-term agent knowledge be represented compactly under strict context budgets?

An agent that processes hundreds of daily conversations cannot store all interaction history. Raw trajectory storage leads to context overflow and degrades inference quality.

Persona-craw proposes **skill-based memory**: instead of remembering what happened, the agent remembers what it learned. Skills serve as compressed, structured knowledge that directly supports task execution.

**Research Directions:**

- **Skill bank as memory.** Replacing episodic memory with a curated skill bank. Each skill is a compressed representation of many past interactions, organized for efficient retrieval.

- **Context distillation.** When the skill bank grows large, selecting and compressing the most relevant knowledge for a given task becomes critical. This involves semantic matching, recency weighting, and domain filtering.

- **Forgetting and consolidation.** Not all skills should persist indefinitely. Mechanisms for forgetting outdated skills (user preference changed) and consolidating related skills (merging similar corrections) are needed.

- **Retrieval under budget.** Given a fixed context window, how should the system allocate tokens between task description, retrieved skills, and conversation history? Optimizing this allocation is an open problem.

**Related Work:** SkillRL's hierarchical SkillBank provides structured alternatives to flat memory. MetaClaw implements semantic skill retrieval with embedding-based search. Recent work on context distillation in long-context LLMs offers complementary techniques.

---

### 4. Online Training Infrastructure

**Central Question:** How can agents safely perform reinforcement learning from daily interaction while remaining functional?

Building a system where daily conversations drive online RL requires infrastructure that handles asynchronous data collection, reward extraction, policy updates, and safe model deployment as concurrent, decoupled processes.

**Research Directions:**

- **Conversation-to-training pipeline.** Converting ongoing conversations into structured training data (state, action, reward, next state) in real time, without blocking the agent's response latency.

- **Training-serving separation.** The RL training loop must operate independently of the serving loop. A data intermediary buffers experience for training consumption, and mechanisms for hot-swapping model weights ensure uninterrupted service.

- **Safe update scheduling.** Model updates from daily RL carry risk — a poorly trained checkpoint may degrade the user's experience. The system must support scheduled training windows (idle time, night hours), regression testing, and automatic rollback.

- **Sample efficiency.** Daily interaction produces limited data compared to synthetic training. The RL algorithm must be highly sample-efficient, learning meaningful policy updates from small batches of real conversation.

**Related Work:** Claw-R1 introduces the Gateway/DataPool architecture for decoupling serving from training. OpenClaw-RL demonstrates a four-component async loop (serving, rollout, judging, training). MetaClaw implements idle-window model updates. Tinker and MinT provide distributed training backends.

---

## Research Contributions

This project aims to contribute to the following areas:

1. **RL from natural language** — Methods for extracting reward signals from everyday conversational feedback
2. **Conversational skill distillation** — Transforming daily corrections and preferences into reusable agent skills
3. **Skill-based agent memory** — Compressed, structured long-term knowledge representation with semantic retrieval
4. **Online RL infrastructure for conversational agents** — Modular system design for safe continuous learning from daily interaction

---

## Future Directions

### Reward Model for Daily Language

Training specialized reward models that understand the nuances of everyday conversational feedback — distinguishing genuine approval from politeness, sarcasm from correction, preference from passing remark.

### Skill-Policy Co-Evolution

Investigating the coupled dynamics between the skill bank and the RL policy. Skills inform policy through prompt injection; policy performance generates new conversations that update skills. Optimizing this feedback loop is an open problem.

### Personalization vs. Generalization

Understanding the tension between personalizing to one user's style and maintaining general capability. How much should an agent specialize, and when does specialization hurt performance on novel tasks?

### Multi-User Skill Transfer

Extending from single-user learning to multi-user settings. Can skills learned from one user's daily interactions benefit another user with similar preferences? How should shared and personal skill banks interact?

### Longitudinal Evaluation

Building evaluation frameworks for agents that evolve over weeks and months. Standard benchmarks measure static performance; longitudinal benchmarks must track improvement rate, preference alignment, regression detection, and user satisfaction over time.

---

## References

- SkillRL — Experience-based skill distillation and hierarchical SkillBank ([arXiv 2602.08234](https://arxiv.org/abs/2602.08234))
- SAGE — Skill-augmented RL with skill-integrated reward ([arXiv 2512.17102](https://arxiv.org/abs/2512.17102))
- DreamGym — Agent-level experience synthesis ([arXiv 2511.03773](https://arxiv.org/abs/2511.03773))
- Agent Lightning — Training-agent disaggregation ([arXiv 2508.03680](https://arxiv.org/abs/2508.03680))
- OpenClaw-RL — Asynchronous online RL for LLM agents
- Claw-R1 — Gateway/DataPool middleware architecture
- MetaClaw — Skill injection and idle-window model updates
- MinT — Lightweight online training backend
- Tinker — Distributed RL training orchestration
- DeepSeek-R1 — GRPO for long-chain reasoning ([arXiv 2501.12948](https://arxiv.org/abs/2501.12948))

Survey: [Awesome Agentic RL](https://github.com/DUXUCHONG/Awesome-Agentic-RL)
