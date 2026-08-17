---
title: "Yet another stupid death: AI plays NetHack"
date: 2026-08-07
draft: false
tags: ["ai", "agents", "nethack", "claude", "bedrock", "games"]
ShowToc: true
TocOpen: false
cover:
    image: "images/nethackai/hero.png"
    variants: ["images/nethackai/hero.png", "images/nethackai/hero-krea.png"]
    alt: "A robot playing a dungeon crawler on an old terminal while a smaller robot watches from the side"
    caption: "The strategist at work. The small one is waiting for its turn – more on that in a future post. Reload the page and a different artist may have drawn this scene."
---

Some recent deaths from my project logs:

- killed by a jackal, while praying
- killed by a killer bee, while fainted from lack of food
- killed by a boulder
- killed by a boulder (again)
- killed by an Uruk-hai, while praying – 16 turns after the previous prayer

The NetHack community has a name for this: YASD, Yet Another Stupid Death. It's the shorthand veterans use when a character dies in a way that was entirely preventable and slightly embarrassing. My AI agent is a YASD factory.

I'm fine with that, because every one of those deaths made the system better. Let me explain.

## The setup

NethackAI is my testbed for agentic control: an LLM plays real NetHack. The runs go through the NetHack Learning Environment (NLE) in a container – it wraps the actual game code, not a simplified imitation, and hands back the screen as a structured 80x24 character grid. There's also a fallback backend that drives the native binary in a pseudo-terminal and reads the VT100 stream the way a human player's eyes would. Either way I can follow the gameplay live in another terminal while the agent plays.

The strategist seat takes any model behind one adapter – the deaths above came from Claude Opus 4.8 runs, and these days a self-hosted Qwen sits in the same seat for the training runs (more on why later). Whoever plays gets a survival rulebook as its system prompt and a markdown strategy journal as persistent memory, because a full game of NetHack fits in no context window known to science. On top of that sits a library of scripted skills – currently 24 of them, over two thousand lines of Python. Autoexplore with BFS pathfinding and frontier detection. Combat with target assessment. Travel, descend with pet handling, flee, loot, rest, pray. The model doesn't press arrow keys; it issues intents, and deterministic code executes them.

That split matters. The model burns thinking tokens on questions worth thinking about – should I descend, is this fight winnable, what might this unidentified potion be – and the skills handle the tedium at machine speed.

## Every death is a bug report

Here's the engine of the whole project: NetHack kills the agent in ways that name the exact defect.

Both boulder deaths? The trap fired mid-skill, and the skill loop kept stepping. "Click! You hear a rolling boulder" appeared on the message line, and the autoexplore code, blind to it, walked into the boulder's path. The fix was a danger interrupt: skills now abort on trap and damage messages instead of finishing their loop. No more deaths inside a routine that ignores the game screaming at it.

Died praying 16 turns after the previous prayer? NetHack's gods punish impatience, which the model knew in principle and forgot in a panic. The fix wasn't a better prompt – it was a cooldown guard in the pray skill itself. The skill now refuses within 900 game turns of the last prayer, no matter how enthusiastically the strategist asks.

Swung eight hits, twenty-two times in a row, into solid stone? Wall check before the hit loop. Fled from a newt at 33% HP like it was a dragon? Weak-foe exception: difficulty 2 or less and HP above 10%, finish it. Died to a killer bee while fainted from hunger? That one produced a whole food-management review.

The pattern is always the same. The death names the missing guard, the guard becomes code, and that class of stupid death never happens again. NetHack is a QA engineer with a 35-year-old test suite and no mercy.

## The Dev Team Thinks of Everything

The community has another saying: TDTTOE, The Dev Team Thinks of Everything. Cut a tin with a rusty opener and the food inside may make you sick. Polymorph into a xorn and you can eat your own rings. Thirty-five years of developers anticipating what absurd thing a player might try, and having an answer ready.

Have we scratched that depth? Not even close. My agent still dies to boulders and hunger – the tutorial-level material. The deep interactions, the ones the wiki documents with scholarly footnotes, are somewhere below dungeon level 10, and our depth record is 9. There is something humbling about building a frontier-model agent system and watching it get outsmarted by a rolling rock, in a game where the developers anticipated cannibalism etiquette decades ago.

That's also what makes NetHack such a good testbed. The difficulty curve is effectively infinite, every failure is legible, and the game state is text. It stress-tests exactly the things that matter in agentic systems: state tracking, interrupt handling, long-horizon memory, and knowing when not to act.

## The supervisor plays the developer

The newest layer changed my job description. I no longer improve the skills myself – a supervisor loop does.

The mechanics are deliberately dumb. A script owns the batch: start with a time budget, launch the game in a container, and on every tick print exactly one verdict – HEALTHY, DEGENERATE, IDLE, or DONE. Health checks watch for crash loops, stuck sessions, and degenerate play patterns.

The judgment half runs on a stronger frontier model. Which one has changed as models improve – currently it's a mix of Claude Fable 5 and GPT-5.6 Sol, because frontier capacity is a real constraint these days and the supervisor doesn't care whose badge the fixer wears. On a DEGENERATE verdict the batch is already dead; the supervisor reads the run logs, diagnoses the failure, edits a skill or the strategist's prompt, re-verifies against a 19-scenario test harness, and relaunches. If a run is going unusually deep as the budget runs out, it can extend the deadline – a strong character deserves to finish its story.

Three tiers of intelligence, separated by timescale. Seconds: scripted skills. Minutes: the strategist choosing what matters this game. Hours: the supervisor improving the two tiers below it between games. On a recent 60-minute batch the supervisor found nothing to fix at all – two games, no interventions, and a new depth record. The system is starting to not need either of us.

My role is watching the game in another terminal and occasionally suggesting an improvement the telemetry can't see. I've started thinking of it as commentary privileges.

## A licensing footnote that shaped the architecture

There's a follow-up project brewing: distilling a small local model from all this gameplay, so the minutes-tier can eventually run on hardware I own. The runs already produce labeled training material – journals, decision traces, outcome files per game.

One thing to be careful with: frontier model licenses do not allow using their outputs to train or distill other models. We started generating gameplay with the models at hand – Nova, GPT variants like Sol – before reading the terms properly, and the terms are clear across vendors, Claude included. So the training-material runs now use an open-weights model, self-hosted on a GPU EC2 instance, playing the same game through the same skills. Every run carries a purpose flag, and only the pinned open model is accepted for training-candidate runs. The frontier models still play analysis rounds, and their deaths still drive the skill development – but the dataset for distillation comes only from the open model whose license permits it.

It costs some capability. The open model plays worse than Opus. But it plays honestly, in both senses, and for training material that's the kind of honesty that matters.

## What's next

The small robot in the hero image isn't decoration. The skills have compressed the decision problem enough that "autoexplore, descend, or rest" may not need a frontier model much longer, and the training material is accumulating with every run. Whether a distilled local model can hold the minutes-tier is the next experiment.

Until then, the agent keeps dying in ways the NetHack community would find very familiar, and every death keeps making it need me – and the frontier models – a little less.

---

*The earlier posts on [ai-team](/posts/2026-03-14-building-a-self-coordinating-ai-development-team/) and [teaching agents to distrust each other](/posts/2026-07-07-teaching-ai-agents-to-distrust-each-other/) cover the multi-agent side of the same obsession. Find me on [LinkedIn](https://www.linkedin.com/in/ilkka-anttonen) or [Bluesky](https://bsky.app/profile/sirile.bsky.social).*
