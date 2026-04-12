# AI Building Tutorials

Step-by-step builds for specific AI agent capabilities. Each tutorial shows you how to build something working — not how it could work in theory. You finish a tutorial with a running agent doing the task.

This is the canonical source for [Builder Weekly Tutorials](https://thebuilderweekly.com/tutorials). Every tutorial lives here as a folder containing the markdown content, metadata, and any code samples. The website renders the same content with diagrams and syntax highlighting.

## What each tutorial contains

Every tutorial in this repo follows the same structure:

- **A before-state** describing what the task looks like without an AI agent — slow, error-prone, manual, or impossible at scale
- **An after-state** describing what the task looks like with the agent in place
- **A soul line** — one sentence explaining specifically why an agent beats a human at this task. This is the single thing that has to be true for the tutorial to exist.
- **Step-by-step implementation** with runnable code, real API calls, real environment variables. Every code block is copy-paste ready against publicly available services.
- **A breakage moment** showing the failure you'll hit if you skip the verification or accountability layer
- **The fix** showing how to add the missing piece
- **An architecture diagram** showing the system before, during, and after the breakage

The full editorial bar is in [MERGE_GATE.md](./MERGE_GATE.md).

## Available tutorials

### Accountability

- **[Build an accountability loop for your AI agent](./entries/accountability-loop)** — Verify your agent's work at machine speed across hundreds of tasks. Tools: CueAPI. [Read on the web →](https://thebuilderweekly.com/tutorials/accountability-loop)

### Memory

- **[Give your agent memory that survives a restart](./entries/persistent-memory-mem0)** — Build an agent that improves over time without retraining. Tools: Mem0. [Read on the web →](https://thebuilderweekly.com/tutorials/persistent-memory-mem0)

### Operational

- **[Filter a hundred RSS feeds into a daily brief worth reading](./entries/rss-daily-brief)** — Replace an hour of morning scanning with a 30-second agent run. Tools: Exa, Claude. [Read on the web →](https://thebuilderweekly.com/tutorials/rss-daily-brief)

## How to contribute

Three kinds of contributions are welcome:

### Propose a new tutorial

1. Open an issue with the topic, the pillar it belongs to, and the soul line — the one sentence explaining why an agent beats a human at this task
2. Wait for editorial confirmation that the topic fits the bar (we'll respond on the issue)
3. Fork the repo, create a new folder under `entries/{your-tutorial-id}/`, write the tutorial following the structure above
4. Open a PR. The verifier runs automatically against your draft and posts results as a PR comment.
5. If verification passes, a maintainer reviews and merges

### Improve an existing tutorial

1. Fork the repo
2. Edit the tutorial's `tutorial.md`, `metadata.json`, or any code samples
3. Open a PR that touches one tutorial folder
4. The verifier runs the same eight checks against your changes
5. Maintainer reviews and merges if the verifier passes

PRs that touch more than three tutorial folders or sprawl across the repo are auto-closed. Keep changes focused.

### Suggest a structural change

For changes to the schema, the merge gate, the contribution rules, or any non-content part of the repo, open an issue first to discuss before opening a PR. Structural changes need editorial approval before code review.

### The rules

Every contribution — engine-generated or human — passes through the same eight-check verifier:

1. **Scope** — one tutorial covers exactly one task
2. **Intent** — the content matches the metadata (title, soul line, before/after states all align)
3. **Promotional** — no marketing language, no brand pumping, no tracking parameters in links
4. **On-topic** — every section relates to the task in the soul line
5. **Fact verification** — every tool, API, and command in the tutorial is verifiable against real documentation
6. **Consistency** — the tutorial doesn't contradict itself
7. **Voice** — direct prose, short sentences, no em dashes, no AI marketing clichés
8. **Soul line** — the soul line appears verbatim in the opening thesis

PRs that fail any check get a comment explaining the specific failure and are auto-closed. The full rubric is in [MERGE_GATE.md](./MERGE_GATE.md).

## What "open source" actually means here

Most "open" AI publications mean "free to read." This one means more than that:

- **License is [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)** — including full commercial rights. Fork it, sell it, sublicense it, build a business on top of it. Attribution is the only requirement.
- **The corpus is machine-readable** at [thebuilderweekly.com/tutorials/corpus.json](https://thebuilderweekly.com/tutorials/corpus.json). Pull the entire dataset programmatically. No scraping, no rate limits, no API keys.
- **No tracking, no ads, no paywalls.** The content is fully public, including the source markdown and code samples in this repo.
- **The verification logic is open.** The eight checks are documented in [MERGE_GATE.md](./MERGE_GATE.md), and the same rules apply to engine-generated entries and human contributions. No hidden editor's-choice gate.

The only thing this corpus gates is editorial quality. Everything else is yours.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Fork freely, attribute always.

---

Maintained by [The Builder Weekly](https://thebuilderweekly.com).

<!-- auto-deploy test -->
