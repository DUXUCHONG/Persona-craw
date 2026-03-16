# Persona-craw — Your AI That Learns From How You Talk

Most AI assistants forget everything the moment the conversation ends.

Persona-craw is different.

It picks up on your everyday words and quietly gets better over time. No buttons to click. No data to label. You just use it, and it learns.

> The agent gets better every day.

---

## The Idea

Think about working with a new teammate.

Day one, you explain everything. "I like it this way." "Not that format." "Always check this first."

By week three, you barely explain anything. They just know.

Persona-craw works the same way. Your daily conversations — the corrections, the preferences, the "actually, try this instead" — all of it becomes learning material. Over time, it starts to feel like it just knows how you work.

---

## Your Words Are the Training Data

You don't need special commands. Normal, everyday language is enough.

> "Perfect, that's exactly what I wanted."

> "No, use the staging server, not production."

> "I like bullet points better than long paragraphs."

> "Try searching GitHub issues first before writing code from scratch."

> "That's wrong again. I told you — always use pytest."

> "Nah, too long."

Every one of these — praise, correction, preference, frustration, even a casual "nah" — shapes how the agent works next time. Over time, these micro-signals build a detailed picture of how you think, what you value, and how you like things done.

---

## What It Does

### It Learns Skills From Your Habits

You don't teach it explicitly. It picks up patterns from daily use.

After a few days, you notice it already uses pytest without being asked. It formats reports the way you like. It checks staging before production. These aren't rules you wrote — they're **skills** the system extracted from your conversations.

And skills don't stay the same. If you change your mind — "actually, I switched to uv" — the system updates. Old habits fade. New ones take over.

Skills also build on each other. A coding style skill combines with a testing preference skill. A writing tone skill combines with a formatting preference. The more you use it, the richer the combination.

---

### It Remembers What Matters, Forgets What Doesn't

You've been using the assistant for months. Hundreds of conversations.

But it doesn't drown in history. It doesn't paste old conversations into every prompt. Instead, it **compresses** what it learned into compact skills — and forgets the raw details.

It's like how you remember "I know how to ride a bike" without remembering every practice session. The assistant remembers **what works**, not every conversation that led to it.

When you ask for help, it pulls exactly the right knowledge for the task — your preferences, your strategies, your past corrections — without wasting space on irrelevant history.

---

### A Senior Expert Prepares the Playbook, A Junior Runs the Day

For tough, unfamiliar tasks, a powerful AI explores the problem — tries different approaches, figures out what works and what fails, and writes down the winning strategy.

Then a faster, cheaper AI takes over for daily work, armed with that playbook.

You get the quality of a top-tier model without paying top-tier costs for every message. The heavy thinking happens once; the daily execution is fast and cheap.

And the playbook keeps growing. Every time the big model handles a hard case, the daily model gains new capability. Over time, the daily model handles more and more on its own.

---

### Your Corrections Actually Train the AI

When you say "no, not like that" — that's not just a correction. It's a training signal.

When you say "perfect" — that's a reward.

When you silently accept the output and move on — that's a signal too.

Persona-craw turns these everyday reactions into real learning. Not just "remember this preference" learning — actual policy improvement. The AI's underlying behavior changes based on what you approve, what you reject, and what you correct.

This happens continuously, in the background. You don't schedule training. You don't upload data. You just use the assistant, and it trains itself from your words.

---

### Fast and Personal Locally, Powerful When Needed

Persona-craw runs a small, fast model on your device for daily tasks. This model is personalized to you — it knows your style, your preferences, your workflows.

For complex tasks — planning a big project, analyzing a difficult document, handling something entirely new — it seamlessly reaches out to a powerful cloud model.

The result: most interactions are instant and free. Complex ones get full power. And everything the cloud model figures out eventually feeds back into your local model, making it smarter.

Over time, the local model handles more and more. Tasks that used to need the cloud gradually become local. Your personal AI keeps getting more capable.

---

## Scenarios

### Personal Assistant

You ask Persona-craw to plan a trip to Tokyo.

It suggests flights, hotels, and an itinerary.

> "Good plan, but I always prefer boutique hotels over chains. And I like walking-distance restaurants."

Next time — Tokyo, Paris, anywhere — it already knows your travel style. Boutique hotels. Walkable dining. No chains. It picked up this skill from one conversation and applies it everywhere.

A month later, you're planning a complex multi-city trip. Persona-craw calls on a more powerful model behind the scenes to work out the logistics, then serves you a clean plan in your preferred format. You don't notice the handoff — it just works.

---

### Coding Assistant

You ask it to scaffold a new Python project.

> "Too many files. I keep things flat until the project grows. And always pytest, never unittest."

From now on, every scaffold is flat with pytest. But it goes further — it noticed you also prefer minimal CI configs and no Docker for small projects. It combined multiple observations into a composite coding style skill.

When you hit a tricky architecture problem, a senior model explores options and distills the best approach. Your daily model applies that pattern to future projects — no need to re-solve it.

---

### Research Assistant

> "Summarize this paper for me."

It produces a summary. You say:

> "Focus on experiments, skip the intro. And compare it to the last paper I read."

Over time, every summary adapts: experiment-heavy, minimal background, with comparisons to your recent reading. That's a skill it learned from your words alone.

When you drop a dense 50-page paper, the heavy lifting runs on a powerful model. The summary arrives in your preferred format, fast and cheap.

---

### Daily Workflows

You use Persona-craw every day: reports, emails, data analysis, meeting notes.

At first, you correct it often. "Wrong format." "Too formal." "Include the metrics table."

After a few weeks, corrections become rare. It learned your format, your tone, your structure.

And it's not just remembering — it's improving. The suggestions get better. The analysis gets sharper. The emails sound more like you. All because your daily reactions are continuously shaping its behavior.

Most of this runs locally, instantly. When a complex analysis needs more power, it reaches out to the cloud — and feeds the result back into your personal model.

---

## What Makes This Different

Most AI products are frozen. They ship a model and that model never changes based on how you use it.

Persona-craw is a living system:

```
your daily words
      |
      v
skills extracted
      |
      v
smarter agent
      |
      v
better conversations
      |
      v
even smarter agent
```

No training buttons. No data upload. No separate "learning mode."

The system learns **while you talk to it**.

---

## The Vision

We believe future AI should feel less like a tool and more like a colleague.

Tools don't learn. Colleagues do.

They pick up your habits. They remember your preferences. They stop making the same mistakes. They get better at anticipating what you need.

Persona-craw is a step toward that — an AI that grows with you, not one you have to re-explain yourself to every day.
