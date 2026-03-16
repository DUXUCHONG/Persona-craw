# EvoAgentOS — Research

EvoAgentOS investigates how AI agents can continuously improve from real-world interaction without explicit human supervision of the learning process.

The project addresses a fundamental gap in current LLM agent systems: while these agents can execute complex tasks through tool use, browsing, and code generation, they do not learn from deployment experience. Each session starts from the same static policy, regardless of accumulated interaction history.

We propose a unified system that integrates **experience learning**, **skill abstraction**, **memory compression**, and **online reinforcement learning** into a single operational framework. The goal is to develop infrastructure that allows AI agents to evolve safely and continuously while deployed.

---

## Research Problems

The system is organized around four core research questions, each corresponding to an open problem in the agent learning literature.

---

### 1. Agent Experience Learning

**Central Question:** How can agents extract training signals from natural interaction without explicit annotation?

Deployed LLM agents generate rich interaction data — tool calls, environment observations, user responses, task outcomes — yet this data is typically discarded after each session. The challenge is to transform this stream of interaction into a reliable source of learning signal.

**Research Directions:**

- **Implicit reward learning.** Users rarely provide explicit feedback. Instead, signals must be inferred from behavioral patterns: whether the user accepts or rejects a suggestion, retries a task, continues a workflow, or abandons the session. The mapping from these natural signals to reward values is noisy and delayed, requiring robust reward inference methods.

- **Trajectory mining.** Raw interaction trajectories contain both successful strategies and failure modes. Identifying which subsequences contribute to positive outcomes — and which lead to errors — requires temporal credit assignment over long, partially observable episodes.

- **Online reinforcement learning.** Unlike offline RL from static datasets, online agent learning must handle non-stationary task distributions, evolving user preferences, and the constraint that the agent must remain functional during training. This demands sample-efficient algorithms that can learn from small batches of recent experience.

**Related Work:** OpenClaw-RL demonstrates that next-state feedback can serve as an online learning signal. MinT provides lightweight infrastructure for online training loops. Claw-R1 proposes a DataPool architecture that decouples experience collection from training consumption.

---

### 2. Skill Abstraction

**Central Question:** How can raw interaction trajectories be transformed into reusable, transferable skills?

Naive storage of past trajectories as memory leads to retrieval inefficiency and context pollution. A more principled approach treats experience not as data to be recalled, but as material to be **abstracted** into structured, reusable strategies.

**Research Directions:**

- **Skill distillation.** Given a collection of successful trajectories for a task class, how should the common strategy be extracted? This involves identifying invariant action patterns, abstracting away task-specific details, and producing a representation that generalizes to unseen instances.

- **Hierarchical skill organization.** Skills naturally exist at multiple levels of abstraction. General skills (e.g., "verify preconditions before executing irreversible actions") apply across domains. Domain skills (e.g., "check rate limits before bulk API calls") apply within specific contexts. Task-specific skills encode narrow but highly effective procedures. The system must support this hierarchy and select the appropriate level at inference time.

- **Skill evolution.** Skills should not be static once extracted. As the agent accumulates more experience, existing skills should be refined, merged when redundant, and deprecated when they no longer improve performance. This recursive evolution process is critical for long-term system quality.

- **Skill quality assessment.** Not all distilled skills are useful. Some may encode spurious correlations or overly specific patterns. The system requires methods for evaluating skill quality, measuring transferability, and filtering low-value entries from the skill bank.

**Related Work:** SkillRL introduces the SkillBank concept with trajectory-to-skill distillation and recursive skill evolution. SAGE integrates skills into the RL training loop, using skill utilization as a reward component. MetaClaw implements skill retrieval and injection at inference time via system prompt augmentation.

---

### 3. Agent Memory Compression

**Central Question:** How can agent memory be made compact, relevant, and useful under strict context budgets?

LLM agents operating over long horizons accumulate interaction histories that quickly exceed context window limits. Raw trajectory storage is both wasteful and counterproductive — injecting irrelevant past interactions degrades current performance.

**Research Directions:**

- **Skill-based memory representation.** Rather than storing interaction history directly, the agent's long-term memory can be represented as a skill bank. This provides a compressed, structured alternative to episodic memory that directly supports task execution.

- **Context distillation.** When the full skill bank exceeds context capacity, the system must select and compress the most relevant knowledge for the current task. This involves semantic similarity matching, recency weighting, and domain filtering to construct an optimal context window.

- **Trajectory summarization.** For episodes that have not yet been distilled into skills, intermediate summarization methods are needed to preserve key decision points while discarding low-information steps.

- **Retrieval mechanisms.** Efficient retrieval over the skill bank requires embedding-based indexing, re-ranking for task relevance, and methods to handle the cold-start problem when skills are sparse.

**Related Work:** SkillRL's hierarchical SkillBank provides a structured alternative to flat memory. MetaClaw implements semantic skill retrieval with embedding-based search and prompt injection. Recent work on context distillation in long-context LLMs offers complementary techniques.

---

### 4. Online Training Infrastructure

**Central Question:** How can agent training be performed safely and efficiently during continuous deployment?

Building a system where agents learn online requires more than algorithms — it requires engineering infrastructure that handles asynchronous data collection, reward computation, policy updates, and safe model deployment as concurrent, decoupled processes.

**Research Directions:**

- **Distributed rollout collection.** Agent interactions happen across diverse environments (browser, terminal, IDE, APIs) and must be collected, structured, and buffered without blocking the agent's primary serving function.

- **Asynchronous reward computation.** Reward signals may require external verification (running tests, checking task completion, evaluating with a judge model). These computations must be decoupled from the serving path and processed asynchronously.

- **Training-serving separation.** The training loop must operate independently of the serving loop. This requires a data intermediary (DataPool) that buffers experience for training consumption, version-controlled model checkpoints, and mechanisms for hot-swapping model weights without service interruption.

- **Safe update scheduling.** Model updates carry risk — a poorly trained checkpoint can degrade user experience. The system must support scheduled training windows (idle time, night hours), regression testing before deployment, and automatic rollback on performance degradation.

**Related Work:** Claw-R1 introduces the Gateway/DataPool architecture for decoupling agent serving from training. OpenClaw-RL demonstrates a four-component async loop (serving, rollout, judging, training). MetaClaw implements idle-window model updates and safe checkpoint management. Tinker and MinT provide distributed training backends.

---

## Research Contributions

This project aims to make contributions in the following areas:

1. **Experience-driven agent learning** — Methods for converting natural interaction into training signal without explicit annotation
2. **Skill-based knowledge representation** — Trajectory-to-skill distillation with hierarchical organization and recursive evolution
3. **Memory-efficient agent architectures** — Skill bank as compressed, structured long-term memory with semantic retrieval
4. **Online RL infrastructure** — Modular, decoupled system design for safe continuous agent training

---

## Future Directions

### Implicit Reward Learning

Developing robust methods for inferring reward from natural user behavior — acceptance patterns, retry frequency, session continuity, output adoption rates — without requiring any explicit feedback interface.

### Skill-Policy Co-Evolution

Investigating the coupled dynamics of skill bank quality and policy performance. Skills inform policy through prompt injection; policy performance generates new trajectories that update skills. Understanding and optimizing this feedback loop is an open problem.

### Hierarchical Memory Systems

Designing multi-level memory architectures that combine episodic memory (recent interactions), semantic memory (distilled skills), and procedural memory (learned policy weights) with principled methods for allocating context budget across levels.

### Continual Agent Benchmarking

Building evaluation frameworks for agents that evolve over time. Standard benchmarks measure static performance; continual benchmarks must track improvement rate, skill accumulation, regression detection, and long-term stability.

### Multi-Agent Skill Sharing

Extending the skill bank from a single-agent to a multi-agent setting, where agents operating in different domains can contribute skills to a shared pool, enabling cross-domain transfer and collective improvement.

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

For a comprehensive survey of the Agentic RL landscape, see [Awesome Agentic RL](https://github.com/DUXUCHONG/Awesome-Agentic-RL).
