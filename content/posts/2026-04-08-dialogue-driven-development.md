---
title: "Dialogue driven development: when code becomes the documentation"
date: 2026-04-08
draft: false
tags: ["ai", "agents", "development", "methodology"]
ShowToc: true
TocOpen: false
cover:
    image: "images/ddd/hero.png"
    alt: "Two silhouettes in conversation with golden speech bubbles and code flowing between them"
    caption: ""
---

Twenty-five years ago, extreme programming made a bold promise: code should be the documentation. Good idea. Didn't work. Humans can't efficiently parse 50,000 lines of code to understand what a system does and why. So we kept writing design docs, architecture diagrams, and specification documents – and they kept going stale.

AI changes that. An agent can read an entire codebase and produce a feature summary that's more accurate than the design doc written six months ago, because it reflects what actually exists. The XP promise is finally landing – just not the way anyone expected.

This opens the door to a development approach I've been practicing while building multi-agent systems: Dialogue Driven Development, or DDD.

## The planning-first trap

Two approaches dominate AI-assisted development today.

Spec Driven Development (SDD) front-loads heavy design. You write detailed requirements, architecture documents, and task breakdowns before any code is written. The idea is that agents can then work autonomously for extended periods without course corrections. In practice, nobody reads the mountain of markdown produced in the planning phase. The specs drift from reality after the first pivot, and maintaining them becomes a tax on every change.

Prompt Driven Development (PDD) structures work around reusable prompts – templates that encode patterns and best practices. Less rigid than SDD but still assumes you know what you're building before you start building it. The prompts become a framework that constrains rather than enables.

Both share the same assumption: that you can design your way to a good outcome before you've learned what works. For well-understood problems with clear requirements, that holds. For anything novel or exploratory, it breaks down fast.

## What happens when plans meet reality

I like to iterate and pilot things. Build a feature, try it out, pivot and change it multiple times. Most software I build scratches a specific itch, so designing for abstraction upfront is difficult – you don't yet know which abstractions matter.

If documentation or heavy plans are done first, they still miss things and require tweaking. If they aren't updated during iteration, their value deteriorates. It becomes negative. An outdated architecture diagram is dangerous because it tells a wrong story with authority.

Build something small, see if it works, adjust, repeat. The question is whether your development workflow supports that or fights it.

## Dialogue Driven Development

DDD treats the conversation between human and AI as the primary development artifact. Instead of writing specs that an agent executes, you bounce ideas back and forth until they're ready to be broken down into work. The dialogue itself captures the reasoning, the tradeoffs, and the decisions – not a document written after the fact.

In practice:

1. Start with a conversation. Describe the itch. Explore approaches. Push back on each other's ideas. The dialogue naturally surfaces edge cases and constraints that a spec would miss.

2. When an idea is ready, break it into small implementation units. In my case, these are "beads" – atomic tasks that agents pick up and execute. Each bead is small enough to build behind a feature flag and easy to throw away if the idea doesn't pan out.

3. Try the implementation. If it works, keep it. If it doesn't, pivot. The cost of pivoting is low because the units are small and the heavy documentation doesn't exist yet.

4. Generate documentation from the implementation. Not the other way around. The feature database, the architecture summary, the one-pager – all derived from what was actually built, not what was planned.

This flips SDD on its head. Instead of design -> implement -> maintain docs, it's dialogue -> implement -> generate docs. The documentation is always accurate because it's produced from the source of truth: the code itself.

## Documentation as a lens, not an artifact

If AI can read code directly, who is documentation for?

Not for AI – agents parse code just fine. Documentation is now a human-readable lens into a codebase that AI maintains and humans oversee. Its purpose has shifted from "instructions for the next developer" to "does the human have an accurate mental model of what the system does."

Instead of authored artifacts that rot, you want generated views that stay current:

- A feature-level lens: what does the system do? A feature database that tracks implemented features, rejected ideas, and work in progress – generated from commits and implementation history.
- An architecture-level lens: how is it structured? Produced by an agent reading the codebase, not by a human drawing boxes and arrows that go stale.
- A decision-level lens: why was it built this way? Captured naturally in the dialogue that drove the implementation, not reconstructed after the fact in an ADR.

Each one generated on demand, not manually maintained. Updated by running a command, not by remembering to edit a file.

Finding the right balance of lenses is the interesting challenge. Too many views and you're back to documentation overhead. Too few and the human loses the plot. The sweet spot is the minimum set of views that keep a human effectively in the loop – able to understand, govern, and steer the system without reading every line of code.

## A concrete example

I've been building [ai-team](/posts/2026-03-14-building-a-self-coordinating-ai-development-team/), a multi-agent coordination system where autonomous agents pick up tasks, implement them, and submit results. The feature database is a recent addition that illustrates the DDD cycle.

The idea started as a conversation: agents keep re-suggesting ideas that were already rejected. That's annoying. What if we tracked feature states – "implemented", "rejected", "not refined yet"? The dialogue explored the shape of the solution, pushed back on over-engineering, and landed on something simple.

Implementation went through beads – small, focused tasks that agents executed. The feature database can be backfilled from existing beads and commits, so the history wasn't lost. Going forward, it generates one-pagers about implemented features and stores rejected ideas so they don't resurface.

No spec was written before implementation. No architecture document preceded the code. The feature exists, it works, and the documentation was generated from it. If the idea hadn't worked, the cost would have been a few small beads – not a spec, a design review, and an implementation that all need to be unwound.

## When DDD fits and when it doesn't

DDD works well for novel, exploratory work where you're discovering requirements as you build. It works when the cost of pivoting needs to be low and the speed of iteration matters more than upfront completeness.

It's not a good fit when you need maximum backwards compatibility, when regulatory requirements demand traceable design decisions before implementation, or when you're building something so well-understood that a spec genuinely captures the full picture upfront.

Most real work falls somewhere in between. Use DDD for the exploratory phase – figure out what works through dialogue and iteration – then solidify with generated documentation once the shape is clear.

## The unexpected payoff

There's a human element I didn't expect. DDD produces a different kind of developer satisfaction. The traditional dopamine hit comes from writing clever code – solving a puzzle at the implementation level. DDD shifts that to a higher altitude: getting a successful feature out the door, seeing an idea work in practice, iterating until something clicks.

The results are on another level compared to low-level coding satisfaction, because you're operating on features and outcomes rather than functions and classes. The quick turnaround from idea to working feature keeps momentum high. And the natural dialogue flow replaces the review-heavy cycle of generated content.

When you remove the friction between having an idea and seeing it work, development becomes fun in a different way. Not "I solved a hard algorithm" fun, but "I built something useful today" fun. I'd take the latter.

## The XP promise, delivered

Extreme programming said code should be the documentation. Agile said working software over comprehensive documentation. For twenty-five years, these were aspirations that didn't quite land because humans couldn't efficiently use code as a knowledge source.

AI closes that gap. Agents can read a codebase and produce accurate, current documentation on demand. That makes front-loaded design docs optional for exploratory work – not just in theory, but in practice.

Talk through the idea. Build it small. Try it. Pivot if needed. Generate the docs from what you built. Keep the human in the loop through lenses, not through artifacts that demand maintenance.

The code is finally the documentation. It just needed a reader that could keep up.
