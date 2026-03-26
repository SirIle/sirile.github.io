---
title: "Three ways to make AI agents talk to each other"
date: 2026-03-20
draft: false
tags: ["ai", "agents", "multi-agent", "communication", "architecture"]
ShowToc: true
TocOpen: false
cover:
    image: "images/agent-communication/hero.png"
    alt: "Three different patterns for AI agent communication"
    caption: ""
---

If you've tried building a system where multiple AI agents work together, you've hit the communication problem. How do agents coordinate? How do they share results? How do you keep them from talking past each other or getting stuck in loops?

I've tried three different approaches over the past months. Each solves some problems and creates others. None of them is perfect, but I'm converging on something that works.

## The problem with subagents

Most AI coding tools support some form of subagent delegation. The main agent spawns a helper, gives it a task, waits for the result. Simple in theory, painful in practice:

- Tool permissions need constant tweaking or human approvals
- You can't see what delegated subagents are doing or intervene when they go sideways
- There's usually a hard limit on concurrent subagents
- No good way to leave a traceable log trail of what each subagent actually did

These limitations pushed me to look for better patterns. I've ended up trying three distinct models, each with real projects behind them.

## Model 1: Coordinator (hub and spoke)

The first approach is the most intuitive. A coordinator agent sits in the middle and tags other agents who respond in a shared conversation – basically a group chat with a moderator.

The coordinator assigns tasks, agents report back, and the conversation thread becomes the shared context. It works, and the traceability is excellent – you can read the conversation and understand exactly what happened and why.

The problems show up at scale:

- Context bloats fast. Every agent's response adds to the shared window, and after enough back-and-forth the coordinator starts losing track of earlier decisions.
- You need guardrails to prevent acknowledgement loops where agents keep confirming they understood each other instead of doing work. "Got it!" "Thanks!" "Acknowledged!" – tokens spent saying nothing.
- Steering is needed to keep agents focused on the task rather than being overly polite. Left unchecked, agents will spend more tokens on pleasantries than on actual work.
- Supporting tools like persistent memory and history access become essential to compensate for the context window filling up.

I tested this pattern in my first multi-agent project. It works for small teams (2-3 agents) on focused tasks, but doesn't scale well when you need sustained parallel work.

## Model 2: Direct messaging between top-level agents

The second approach gives each agent its own CLI session and has them send messages directly to each other. No coordinator – peer-to-peer communication.

This sounds cleaner but it's kludgy in practice. The fundamental issue is that current tools don't have native support for inter-agent messaging. You end up with a shell process pushing messages into agent sessions, and messages can only be delivered when the target agent's prompt is idle.

This leads to a specific failure mode: if the target agent is thinking or busy when a message arrives, the delivery fails silently. The sending agent assumes the target is dead and tries to kill and respawn it. By the time the target finishes its work and becomes idle, it's already been terminated. The result is a cycle of agents killing each other's work.

The upside is that all agents are "top level" – you can directly inspect, manipulate, or intervene with any of them. The UI is typically tmux with different panes per agent, which is technical but transparent.

KiroHive and similar tools use this pattern. It works when you're actively watching and can intervene, but it's fragile for unattended operation.

## Model 3: Turning the problem on its head with beads

The third approach comes from a different direction entirely. Instead of agents talking to each other, agents communicate through a shared task graph. The concept is inspired by Steve Yegge's [beads](https://github.com/steveyegge/beads) project.

Every unit of work is a bead. Agents don't send messages – they pick up beads, do the work, and mark them done. The output of one bead becomes the input context for the next. A builder completes an implementation bead, which unblocks a review bead, which a reviewer picks up. The "conversation" is the chain of beads and their artifacts.

Each agent session is spawned fresh for a single bead. It gets fed exactly the context it needs: project docs, relevant decisions, the bead description, AST structure of affected files, and entries from long-term memory. It does the work and exits.

This has several consequences:

- Context rot can't happen. Sessions are short-lived and purpose-built. Bead #900 runs just as well as bead #1 because every agent starts clean.
- Parallelism is straightforward. Independent beads run simultaneously on different agents without coordination overhead.
- The trail of actions is excellent. Every bead has a status, an agent assignment, a log, and a result. You can reconstruct exactly what happened.
- Automatic retries and triage work naturally. A failed bead gets retried, and if it fails repeatedly, it escalates.
- Token usage is high by design. Every fresh session means re-reading project context. But this is a deliberate trade-off for reliability.

The startup cost per bead is real – each agent needs to orient itself in the codebase before doing useful work. AST tree analysis and a document database help here, giving agents a structural view without having to read every file.

I've been running this pattern in [ai-team](/posts/2026-03-14-building-a-self-coordinating-ai-development-team/) with over 1,000 beads processed. It works well for sustained, minimally-supervised development.

## Where I've landed: a mix of 1 and 3

After trying all three, my current approach combines the coordinator and bead models depending on the phase of work.

For planning, a long-running conversation works better. Discussing architecture, exploring trade-offs, asking clarifying questions – this is dialogue, and it benefits from shared context and back-and-forth. The coordinator model fits here.

For execution, beads take over. The lead agent breaks the plan into beads with dependencies, and the scheduler handles the rest. No conversation needed – just a queue of well-defined tasks flowing through specialized agents.

Some projects also need a different balance. Analysis work using Jupyter notebooks benefits from staying in a conversational context where you can look at results and ask follow-up questions. Pure coding projects are better suited to the bead pipeline where each change is isolated, tested, and reviewed independently.

## The bigger picture

This is an interesting problem space that's being explored in multiple places simultaneously. Every team building multi-agent systems is hitting the same communication challenges, and different solutions are emerging.

The community hasn't converged on a standard pattern yet. Direct messaging frameworks exist but feel immature. Task graph approaches like beads are gaining traction. Hybrid approaches that mix conversation and structured task execution seem promising.

I'll keep watching where this goes. The tools are evolving fast, and what's kludgy today might be well-supported tomorrow. For now, the combination of conversation for planning and beads for execution is the most reliable pattern I've found.
