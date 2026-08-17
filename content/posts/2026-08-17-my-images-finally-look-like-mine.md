---
title: "My images finally look like mine"
date: 2026-08-17
draft: false
tags: ["ai", "images", "lora", "self-hosting", "krea", "dnd"]
ShowToc: true
TocOpen: false
cover:
    image: "images/krea-lora/krea2-judges.png"
    variants: ["images/krea-lora/krea2-judges.png", "images/krea-lora/bedrock-judges.png"]
    alt: "Three robot judges at a courtroom bench, the middle one raising a gavel"
    caption: "Three robot judges, one prompt. Reload the page and a different artist may have drawn them – that trick is part of this story."
---

Everyone who has generated AI images knows the feeling. Each image is fine on its own, but put ten of them on your blog and they look like you hired ten different freelancers who never met each other. The problem isn't quality anymore. The problem is that the images don't look like *yours*.

This is the story of how I fixed that with a rented GPU, an evening of training, and a few euros – and what broke along the way.

## Two artists, one brief

I run my blog heroes through a fixed prompt pattern: golden amber, deep teal, near-black background, flat digital illustration. That pattern was doing all the brand work, so I tested how much of it actually survives the model. Same three prompts, two backends: Stability Ultra on Amazon Bedrock, and Krea 2 Turbo running self-hosted on a GPU instance I start and stop as needed.

![Nine images in a three-by-three grid: each row is one prompt, each column one backend. The left column varies wildly in style, the middle and right columns look like one consistent artist.](/images/krea-lora/contact-sheet.png)

*Left column: Stability, one prompt per row. Middle and right: Krea 2, two separate sessions days apart.*

Look at the columns, not the individual images. The left column is three different artists who never met: one drew flat retro art, one drew a photograph, one drew a 3D render. The middle and right columns are the same illustrator on different days – and those two columns come from separate sessions, on different model launches. Krea held the style on every prompt. Stability held it on one of three, dropped the second robot from one brief entirely, and added garbled text to a screen that the prompt explicitly said should have none.

A single great image is luck. A column that holds together is an identity, and for anything you publish regularly, the column is what matters.

The plumbing, kept to one paragraph as promised: the model runs on a g6e.xlarge EC2 instance that costs about as much per hour as a coffee, and roughly a pizza per month while stopped. A small daemon polls a queue in S3, so anything on my machine – or any of my agents – can submit a job and fetch the result. First image of a session pays a few minutes of boot and model load; after that, an image takes under half a minute. The instance shuts itself down when the queue stays empty. That's it. No subscription, no rate limits, and the weights sit on my own disk.

## The harder problem: the same face twice

Style consistency turned out to be the easy half. My D&D campaign project (the same multi-agent setup from the [ai-team posts](/posts/2026-03-14-building-a-self-coordinating-ai-development-team/)) has recurring heroes, and they need to look like themselves across every scene: portraits, battle maps, standees. Prompting cannot do that. You can pin the wardrobe in words – silver braids, blue dress, white cloak – but the face underneath changes actors every time.

Here is what sixty attempts at the same elf look like:

![A grid of sixty AI-generated portraits of an elven woman. Every portrait wears the same blue dress and white cloak, but the faces clearly belong to different women.](/images/krea-lora/curate-lyra.png)

*The curation view for Lyra: same costume in every tile, a different woman in most of them.*

The fix is a LoRA – a small adapter file trained on top of the base model that teaches it one specific thing. In this case: who Lyra is. The recipe is short. Generate about sixty candidates from a canonical description, click through them in a little keyboard-driven picker (space to keep, arrows to move – the agents built it, I judged), keep the twenty or so where the character is recognizably the same person, caption each with a made-up trigger word, and train overnight on the same GPU. About four hours per character.

The curation step is where the humanity lives, and sometimes it catches things you didn't know to look for. Jonathan is a human paladin, not an elf – but the model kept painting his hair over the exact spot where you'd check for pointed ears. The project ordered a targeted re-shoot, hair pulled back, before his dataset passed:

![Another curation grid, this time of an armored knight with auburn hair. The bottom rows show him with hair pulled back, revealing round human ears.](/images/krea-lora/curate-jonathan.png)

*The ear verification round, bottom rows: round ears confirmed, canon preserved.*

## What a trained adapter changes

Everything. Here is the same prompt – "bust portrait, gentle smile, golden hour sunlight, on a castle balcony" – with the character's trigger word, adapter off on the left, adapter on the right:

![Two images side by side. Left: a generic elf woman who is not Lyra. Right: Lyra, unmistakably the same character as the training set, in a new scene.](/images/krea-lora/composite-lyra-pair.png)

*Lyra without and with her adapter. The base model has never heard of her.*

To the base model the trigger word is noise, so it invents someone. With the 90-megabyte adapter loaded, the same words produce *the* character, in a scene and light that appear nowhere in the training data.

Jonathan's pair made the point even better, by accident:

![Two images side by side. Left: a literal stone bust sculpture of a young woman on a pedestal. Right: Jonathan the paladin, smiling in armor in a castle courtyard.](/images/krea-lora/composite-jona-pair.png)

*The base model read "bust portrait" and delivered exactly that: a bust. Made of stone. Of someone else.*

Without the adapter, the model took "bust portrait" as literally as language allows and sculpted a stranger. With it, Jonathan walked out of the keeper set into a sunny courtyard. If you ever need to explain to someone what a character LoRA does, this pair is the whole lecture.

And because the adapter only pins identity, everything else stays negotiable. Lyra as a tarot card, action pose, magic and all:

![A tarot card in art nouveau style showing the elf Lyra in a dynamic pose, silver magic swirling from her raised hand.](/images/krea-lora/blog-lyra-tarot.png)

*Same person, completely different assignment. The card titled itself "SSTAR" despite the prompt saying no text – identity is solved, spelling is not.*

## Stacking artists

The tarot card was still one adapter plus a persuasive prompt. The real trick is loading two adapters at once: one that knows *who*, one that decides *how*. Krea publishes official style adapters – watercolor, ink wash, a children's-drawing style – as separate downloads, each a file you drop next to the model, and the community makes many more (the anime panel below comes from one such collection). Stack Lyra's identity adapter with one of them and you get this:

![Five portraits of the same elf woman in a row: painterly house style, Art Deco watercolor, monochrome ink wash, a children's book sketch style, and anime.](/images/krea-lora/strip-lyra-five-artists.png)

*One character, five artists. Left to right: the house look, Art Deco watercolor, monochrome ink wash, naive sketch, anime. Same braids, same pendant, same person.*

Every portrait is the same woman – the ink wash even keeps her pendant as the only spot of color. This is the moment the setup stops being a tool and becomes a studio: the character adapters are the cast, the style adapters are the artists, and any job can pair one of each by name. Adding a new artist to the shelf costs a download, or an evening of training if nobody has made the style you want.

## Where it breaks

Honesty section. Each adapter teaches the model one person, and it applies that person generously. Ask for both heroes in the same frame:

![A two-by-two grid of couple portraits on a castle rampart. Top left: two generic anime characters. Top right: two identical elf women. Bottom left: two identical knights. Bottom right: two knights, one with silver hair.](/images/krea-lora/composite-duo-grid.png)

*One prompt, four attempts. Clockwise from top left: no adapter (two strangers, and the house style gone too), Lyra's adapter (two Lyras), both adapters stacked (Jonathan and a silver-haired Jonathan), Jonathan's adapter (two Jonathans).*

With one adapter loaded, everyone in the frame becomes that character – Lyra's adapter produces twins, Jonathan's produces two of him. Stack both at reduced strength and you get a blend: Jonathan's armor won the wardrobe for both figures, Lyra survived only as one of them's hair color. The project's plan saw this coming and calls for composing group scenes from per-character renders instead. Knowing where the technique stops is part of owning it.

## Read the license before the pipeline exists

This is the third time the same rule has hit me across projects, so it has earned its own section. My [NetHack agent](/posts/2026-08-07-yet-another-stupid-death-ai-plays-nethack/) had to switch to an open-weights model for training data, because frontier model licenses forbid using outputs to train other models. The exact same clause exists in Stability's license: outputs from the Bedrock side cannot be used to train other generative models. So every image in these training sets was generated with Krea 2 itself – which its community license permits, and which is technically better anyway, since the model learns from in-distribution data. Even the D&D token art we already owned was ruled out, because its own license doesn't extend to training.

Once is a gotcha, three times is a rule: where your training data may come from is a design input, not an afterthought. Retrofitting provenance means regenerating everything – I know, because none of my previously published hero images qualified, and the style dataset has to be rebuilt from scratch.

Two more clauses worth knowing if you follow this path: the trained adapters are derivatives of the model, mine to use but not to distribute freely, so they stay private. And images you publish should be labeled as AI-generated – in the EU that's not just politeness anymore. Consider this post its own label.

## The studio ledger

What all of this cost, in round numbers: the comparison session was about an hour of GPU time, so roughly two coffees. Each character's dataset batch and overnight training run came to a few euros. The adapters themselves are 90-megabyte files sitting next to the model weights, loaded per job by name. One machine now serves every project I have – blog heroes in the house style, campaign portraits with persistent faces – and adding a new specialist costs an evening and the price of a lunch.

The obvious next hire is a style adapter trained on the blog's own look, so the amber-and-teal identity joins the shelf as a named artist instead of a prompt incantation. The two-Lyras problem also isn't going anywhere; per-character compositing is the practical answer today, but it's an itch.

The part I keep coming back to isn't the cost, though. It's that every keeper in those training sets passed through my hands – space bar, next, space bar. The adapter is a compressed record of a few hundred small judgments about what my characters and my pages should look like. That's why the results feel like mine: because the taste in them is. The machine just made it repeatable.

---

*The multi-agent system that built the picker, ordered the ear re-shoot, and ran the training is the same one from [Building a self-coordinating AI development team](/posts/2026-03-14-building-a-self-coordinating-ai-development-team/). Find me on [LinkedIn](https://www.linkedin.com/in/ilkka-anttonen) or [Bluesky](https://bsky.app/profile/sirile.bsky.social).*
