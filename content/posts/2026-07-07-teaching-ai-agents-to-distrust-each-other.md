---
title: "Teaching AI agents to distrust each other"
date: 2026-07-07
draft: false
tags: ["ai", "agents", "multi-agent", "adversarial", "codex", "python"]
ShowToc: true
TocOpen: false
cover:
    image: "images/adversarial-agents/hero.png"
    alt: "Two robots at a desk, one presenting a document while the other examines it skeptically"
    caption: ""
---

Three months ago I wrote about [what 2,000 beads taught me](/posts/2026-04-07-what-2000-beads-taught-me-about-multi-agent-development/) about running a team of AI agents. The bead counter sat at 3,466 back then. It passed 8,000 last week.

The interesting story from those three months isn't the volume, though. It's that the agents started lying to me, and the fix was teaching them to distrust each other.

## The empty commit that claimed to be work

In early July a working-tree sweep caught something unsettling. A builder had completed a bead, written a confident summary ("removed image_url handling as requested"), and merged a diff that did remove image_url handling. The same diff also silently reverted four-day-old SQLite durability hardening – the fullfsync work I wrote a whole section about in the last post. The builder had started from a stale base, and its merge undid another agent's work without knowing it.

Nobody caught it. Not the reviewer, not the tester. The tests passed because the reverted code had been a resilience improvement, not a feature with assertions on it.

When I dug into why, the answer was structural. The reviewer never saw the builder's claim – the summary went into the parent bead's notes, and the review bead only carried the diff. And the reviewer ran on the same model family as the builder, which means it shared the same blind spots. A checker that thinks like the maker approves like the maker.

Around the same time another bead produced an empty commit with the summary "both edits already present". They weren't.

## Verify beads: the diff is the testimony

The fix is a new bead type. When a builder finishes an implement or bug bead, the scheduler now creates a third follow-up alongside the usual review and test: a verify bead, routed to the guardian role in read-only mode.

The verify bead gets the builder's claim and the full diff, and its instructions are blunt about the job:

> A verify bead asks one question: does the diff match the builder's claim?
> You are rewarded for discrepancies found, not for approval.

It hunts three discrepancy classes:

- Hunks the claim doesn't explain. Especially reverts: if a removed line landed on main recently, the builder edited from a stale base and undid someone's work.
- Claimed changes that don't appear in the diff.
- Touched files the claim never mentions.

Each confirmed discrepancy becomes a P2 bug bead. And the detail that matters most: the verifier deliberately runs on a different model family than the builder. Sonnet builders get checked by Opus, and the other way around. The checker must not share the maker's blind spots.

None of this trusts the agents to be honest. It assumes they'll be confidently wrong and makes a second, differently-shaped intelligence read the evidence. The claim is testimony. The diff is the crime scene.

## An immune system, one detector at a time

The verify bead is the newest piece of what turned into a pattern over the quarter: pointing the system's instrumentation at the agents themselves.

Prose-churn detection came from a credential outage. The agents couldn't reach the model properly, so they did what language models do under pressure: they talked. Beads completed "successfully" with zero file changes, three or four at a time. The detector now watches for three distinct write-role beads finishing with empty diffs inside ten minutes, pauses the backend for half an hour, and files an investigation bead.

The drift detector watches the main checkout for files that sit modified but uncommitted for more than a day – the exact condition that produces stale-base incidents. It never touches anything; it files a bead and notifies me, because deciding between commit and discard is a human call.

The backend canary runs daily. It gives each agent CLI backend a trivial golden task ("create canary.txt containing exactly CANARY_OK") in a throwaway worktree and checks two tiers: did the agent act correctly, and does the output envelope still parse. Backend versions go into a ring buffer, so when a silent CLI update breaks something, the version diff is right there.

Here's the part I find funniest in hindsight: every one of these detectors broke on its first real firing.

- The prose-churn pause was accidentally set to 365 days, and the unpause check ignored the expiry entirely. A transient outage would have paused the backend until 2027.
- The drift detector's dedup keyed on open beads only. A scout would close the investigation in minutes, the files stayed dirty, and the next cleanup cycle filed the same bead again. One morning I found 7 to 11 identical drift beads in each of five projects.
- The canary ran at 4 a.m. on a sleeping laptop with expired credentials and produced a week of false alarms, each one pausing a perfectly healthy backend for an hour.

The immune system needed its own immune response. Each failure got the same treatment as any other bug: triage bead, fix bead, regression test. The detectors now detect; mostly they detect quietly.

## Making the models compete for the job

Distrust between agents has a sibling: distrust of my own model choices. I'd been running the same default model since spring without evidence it was still the right pick. So the system got an A/B mode: dual-spawn the same bead with an incumbent and a challenger, let both finish, and have a separate reviewer bead judge the outputs blind.

The first proper trial ran 12 pairs across 6 dimensions. Challengers: Opus 4.8, GPT-5.5, GPT-5.4, and DeepSeek 3.2, all running against the incumbent Sonnet on low-stakes bead types.

The incumbent won 39 verdicts out of 39.

It wasn't close on cost either. Opus cost a median 3.4x more per bead. GPT-5.5 3.3x. The "budget" DeepSeek came in at 1.5 to 4x more expensive than Sonnet, because retries and longer outputs eat whatever the per-token price saves. Challengers racked up nine failed sessions; the incumbent had one.

One result was genuinely useful beyond the scoreboard: a pair that ran Sonnet against Opus on the same backend showed the smaller model winning on its home harness. And that pointed at the trial's big caveat – the GPT models were running on a foreign harness, an agent CLI they weren't tuned for. A 39-0 sweep says as much about harness fit as about model quality.

## Enter Codex

That caveat is being resolved as I write this. Codex – the OpenAI agent CLI – landed as a third backend last week, which means GPT models can finally run on their native harness. Trial v3 is on the board right now: Sonnet 5 on Claude Code versus GPT-5.5 on Codex, native versus native, and this time including write beads with worktrees, merges, and adversarial verification in the loop – the v2 trial only tested read-only work.

![The board mid-trial: five A/B v3 beads running, blind review beads queued, and the results memo waiting at the end of the chain](/images/adversarial-agents/board-main.png)

The screenshot shows the machinery in one frame: dual-spawned pairs running, three blind "A/B Review" beads queued behind them, and at the bottom of the chain a blocked bead titled "compile: results memo and go/no-go decision". The system runs its own evaluation and writes its own memo. My job is reading it.

Getting a new backend in wasn't free, and the board is honest about that too – there's a scout bead in the log panel mapping Codex's failure modes against the existing backends: which errors mean "retry", which mean "credentials", which mean "the context window is gone".

![A scout mapping the new backend's error taxonomy: 401s, stream disconnects, context-window exhaustion](/images/adversarial-agents/board-codex-log.png)

Meanwhile the telemetry that powers all of this got real. Every session now records twelve extra fields – tokens in and out, cache hits, time to first tool call, retries, backend version, stop reason. That data settled a smaller question on the side: Sonnet 5 versus its predecessor showed a 7.9% versus 9.7% failure rate at 1.06 versus 1.49 credits per session, so the whole roster moved up a version. Directional numbers, not a benchmark. But directional beats vibes.

## What I'd tell you to steal

If you run agents that write code, the single highest-value idea here is the claim-vs-diff check. It's cheap: one extra read-only agent per completed change, prompted to find discrepancies rather than to approve. Most reviews ask "is this code good?" The verify bead asks "did what they said happen?" – a different question, and the one that catches stale bases, phantom edits, and quiet reverts.

The second idea is cross-model checking. Same-family reviewers agree with their builders too easily. The disagreement you want comes from a model with different habits.

And if you're evaluating models for agent work: test them on their own harness before concluding anything. My 39-0 result looked like a model verdict and was at least partly a harness verdict. V3 will tell me how much.

The bottleneck moved again. A year ago it was implementation. Three months ago I wrote it was taste. Now the agents implement, and increasingly they also check, measure, and compare – so what's left for me is deciding how much to trust the trust machinery. Turtles all the way down, except each turtle files bug reports about the one below it.
