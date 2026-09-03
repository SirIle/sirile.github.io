# AGENTS.md – sirile.github.io

Hugo blog (PaperMod theme). Personal site of Ilkka Anttonen.

## Build

- Preview: `hugo server -D --baseURL http://localhost:1313`
- Build check: `hugo -D`
- Config: `hugo.toml`. Theme overrides in `layouts/`, CSS in `assets/css/extended/custom.css`.

## Content conventions

- Posts: `content/posts/YYYY-MM-DD-slug.md`, images under `static/images/<topic>/`.
- Front matter `cover.image` is the primary hero and og:image. Optional `cover.variants` lists extra heroes; `layouts/partials/cover.html` shows one at random per page view.
- Mermaid blocks, the `gallery` shortcode and GLightbox are available; see existing posts for usage.
- Screenshots are scrubbed before publishing. Keep replacement strings the same length as the originals so box-drawing alignment survives.

## Writing rules

- En-dashes (–) or commas, never em-dashes. Straight quotes only. Sentence case in headings.
- First person, conversational, honest about what didn't work. Avoid the patterns listed at [Wikipedia:Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing): no bold-first bullets, no unicode arrows, no signposted conclusions, no marketing vocabulary.
- Label AI-generated images when it matters.

## Deploy

Pushing to `main` triggers GitHub Actions. Local machine policy may block pushes; the maintainer pushes from a separate box. Do not force-push or rewrite history.

Local, untracked agent context lives in `.kiro/` (gitignored).
