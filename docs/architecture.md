# Persona-craw — Architecture

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
