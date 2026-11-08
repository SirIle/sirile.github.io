---
title: "Where rabbit holes go to wait"
date: 2026-10-29
draft: false
tags: ["ai", "agents", "workflow", "tooling", "memory", "pensieve"]
ShowToc: false
cover:
    image: "images/pensieve/hero-loom-a-krea.png"
    variants: ["images/pensieve/hero-loom-b-krea.png"]
    alt: "Flat illustration of a dark weaver's workshop: five glowing bobbins of thread on a wooden rack, each with a blank paper tag, one thread running into a hand loom that weaves a cloth of light, a knotted cut thread hanging from a hook on the left"
    caption: "Strands on bobbins, waiting. One of them is being woven; the knotted one on the hook is a dead end. AI-generated image (Krea 2 Turbo)."
---

Every working session with an AI agent throws off tangents. A link in a thread. A tool someone mentions. A "what if the watch could show that" while debugging something else entirely. The tangents arrive faster than I can follow them, and I had two bad options: follow one and derail the task I was actually doing, or let it go and lose it. For months my answer was a third option that was worse than both, a session parked at the root of my projects folder where links and half-ideas went to die. It had no log. Explorations evaporated the moment something else came up.

So I built a place for them to wait.

## A strand is a thought with a status

The unit is a strand: one Markdown file with frontmatter in a pool at `~/Projects/pensieve/strands`. A strand has a status, parents, a boundary, a "what it is", a "current thinking", open questions, and a resume pack that tells the next session where to pick up. The frontmatter is the whole truth; any index is derived and can be thrown away.

```yaml
strand: garmin-watch-face-from-jarvis
status: landed
boundary: personal
parents: []
created: 2026-10-24
last_touch: 2026-10-26
produced: [/Users/anttonen/Projects/watch-face]
landed: {project: /Users/anttonen/Projects/watch-face, name: watch-face}
```

Five statuses, and the lifecycle is most of the idea.

<div class="mermaid">
stateDiagram-v2
    [*] --> draft
    draft --> active
    active --> parked
    parked --> active
    active --> dead_end
    active --> landed
    active --> active: merge (child with two parents)
</div>

Draft is what the weekly sweep (described below) found and I haven't looked at yet. Active is a strand I'm in. Parked is "not now". Dead end is the one I care about most, because a dead end has to carry a reason. Not walking the same dead end twice is half the value of the whole pool. Landed means the strand became a project and the file points at it. Nothing is ever deleted, and a merge is a new strand with two parents, so the lineage survives free association.

Every line in a strand says who wrote it. Me, a model, or the tool itself during a run. That answers the question I would ask of anyone else's idea pool: how much of this is machine noise? You can see it per entry.

## Capturing without leaving

The capture has to happen from inside whatever conversation produced the tangent, or it won't happen. So it's one command the agent can run for me while we keep working on the real task:

```bash
ai-cli dive "custom Garmin watch face fed by Jarvis data"
```

That creates and commits the strand, then opens a tab into a fresh session anchored to it. If the idea is a link, a background step on a cheap model fills in "what it is" before I ever open the strand. Forking from an existing strand is `--parent`, picking one up later is `--resume`, and `ai-cli strands --fzf` lists the pool when I want to browse.

`dive` opens the strand in whatever route `ai-cli` currently ranks best for it, and a route is more than a model name: model, harness, credential route, region and context mode, scored against a policy and a live health snapshot, with retries and failover when a route misbehaves. That router deserves its own post. It gets a paragraph here because the pool only needs something to show up, whichever model that is.

The verbs live in `ai-cli`, my routing tool, and the pool is just files. The Pensieve app itself is a thin terminal UI over those verbs: a list, a filter, an intake box, and a suggestion panel. The intake box is why the app exists. A half-thought or a URL arrives while I'm doing something else, and the question is whether it's new or belongs to a strand I already have. The suggestion panel answers that before the strand is created.

![The Pensieve terminal UI: a list of strands with status and last touch, a filter box, and the intake box with its suggestion panel pointing at an existing strand.](/images/pensieve/tui.png)

*The intake box and its suggestion panel. Type a half-thought, see which existing strand it probably belongs to.*

## A worked example

On Saturday 24 October, in the middle of something unrelated, I wondered whether my Garmin could show the training-readiness number my home dashboard already knows. One `dive`. The strand got a scope, three decisions and two open questions over the next two days, written by me and by the sessions that did the research. On Monday it graduated:

```bash
ai-cli graduate garmin-watch-face-from-jarvis --name watch-face
```

Graduation creates the repository and seeds its memory files and project plan from the strand, then leaves two-way pointers. By Monday evening the new project had a toolchain and a passing simulator spike. The strand is marked landed and hasn't been touched since, which is the point.

The oldest file in the pool, carried over from that root-folder session with its original dates, tells the other story. A strand about running an agent orchestration framework in the cloud sat untouched from February to September. When I finally looked, the maintainer had left the company and my own agents had moved on. It's a dead end now, with that reason written in it, and I won't spend an evening rediscovering it.

## The pool fills itself too

The second intake is a scheduled sweep I call radar. Once a week it reads two sources: the articles my dashboard scored 8 or higher, and a fixed set of Hacker News queries. It drops what the pool already has and ranks the rest with a single model call against a rubric describing what I work on. At most five of them land as draft strands, with a digest note recording what was skipped and why. In the app, one key shows the radar inbox; I accept, park or kill each in a few seconds.

## Where it is

As I write this the pool has 20 strands: 13 active, 3 landed, 1 parked, 1 dead end, 2 drafts from the radar. Three strands have graduated into projects since early September, including this tool itself, whose first file is the strand that proposed it. The honest number is the 13: more are marked active than I will return to this month, and the next piece of the mechanism is a pool review that proposes which of them to park or merge. I planned a graph view of the parent links and an export of the pool to Obsidian as linked notes. The export exists and Obsidian's graph view does the job, so my own graph view is still deferred. That's enough for now.

The name is borrowed, obviously. In the books the basin shows you memories you already have. This one holds the thoughts I haven't had time to finish, and the difference is the loom in the picture: you don't go back to relive a strand, you go back to weave it.
