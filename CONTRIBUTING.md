# Contributing

Thanks for thinking about contributing to Builder Weekly Tutorials. This corpus is small on purpose and the bar is sharp. Read this document before opening an issue or a PR.

## The bar

A tutorial earns a place in this corpus if it teaches **one specific task** where an AI agent measurably beats a human. Every published entry contains three things:

1. **A before-state.** What the task looks like without the agent. Concrete enough that a reader recognizes their own workflow in it.
2. **An after-state.** What the task looks like with the agent. Concrete enough that a reader can evaluate whether the trade-off is worth it for them.
3. **A soul line.** One sentence naming explicitly why an agent beats a human at this specific task. "Measurably" is the load-bearing word — the advantage must be numerically defensible (40x speed, 100% recall across N runs, zero variance over M days).

If you can't write the soul line in one sentence and point at a specific dimension (speed, recall, consistency, cost, availability, throughput, concurrency), the topic isn't ready to be a tutorial yet.

## Proposing a new tutorial

Open an [issue](https://github.com/thebuilderweekly/tutorials/issues/new) and use the tutorial proposal template:

```markdown
### Title
{one-sentence title, declarative, no question marks}

### Pillar
{accountability | memory | visibility | agent-teams | scoping | operational}

### Soul line
{why an agent beats a human at this task, one sentence, with a specific measurable dimension}

### Before-state
{what the task looks like without the agent, 1-2 sentences}

### After-state
{what the task looks like with the agent, 1-2 sentences}

### Tools
{list of tools from the Agent Index or the built-in allowlist}
```

A maintainer replies within a few days with one of: "ship it" (open the PR), "sharpen it" (specific feedback on how the soul line or scope needs to tighten), or "not a fit" (with a reason). Most proposals that pass the bar also need the scope narrowed — the common failure is trying to teach three tasks at once.

## Improving an existing tutorial

Fork the repo, edit the entry folder (`entries/{id}/`), and open a PR.

Good targets for an update PR:

- The tutorial's commands no longer work because a tool's API changed
- A better tool now exists for the same task and the old one should be replaced
- A specific step was unclear and you have a clearer phrasing
- A code block is missing an edge case that now matters

Not-good targets:

- "I thought this paragraph read better this way" (style preference without a defect)
- Swapping out the chosen tool for your favorite when the existing tool still works
- Adding a new section that wasn't part of the original tutorial's scope

When in doubt, open the issue before the PR.

## The merge gate

Every PR goes through a seven-check automated gate before a reviewer sees it. Read [MERGE_GATE.md](./MERGE_GATE.md) for the full spec. The checks are:

1. Schema validation (`metadata.json` matches `schema/tutorial.schema.json`)
2. Soul line completeness (≥ 20 chars, names a specific measurable advantage)
3. Before/after state completeness (both present, describe distinct states)
4. Voice violation scan (no em dashes, no banned phrases — see voice rules below)
5. Tool provenance (every tool in the Agent Index or the built-in allowlist)
6. Code block annotations (every fenced block has a language tag)
7. Freshness stamp (`lastVerifiedAt` is current for updates)

The same gate applies to in-house pipeline-generated entries and external contributions. No exceptions.

## Voice rules

The voice rules apply to **published tutorial prose**: `entries/**/tutorial.md` and the `soulLine`, `beforeState`, and `afterState` fields of each entry's `metadata.json`. They do not apply to contributor-facing documentation like `README.md`, `CONTRIBUTING.md`, `MERGE_GATE.md`, the pillar READMEs, or `llms.txt` — those are technical writing for developers and use standard conventions.

When you are writing a tutorial that will be published, the important rules:

**Use direct prose.**

- Subject-verb-object, short sentences, concrete nouns
- "The agent reads the issue, classifies it, and applies a label." Not "The agent is able to perform classification operations on incoming issues."

**No em dashes.**

Use commas, colons, or periods. Em dashes (`—`) are automatically flagged by the gate. A PR with one em dash is a failing PR.

**No marketing words.**

Banned: cutting-edge, next-generation, AI-powered, seamlessly, revolutionary, game-changer, unleash, pivotal, crucial, leverage (as a verb).

**No scaffolding phrases.**

Banned: "in conclusion", "it's worth noting", "furthermore", "moreover", "delve", "tapestry", "landscape", "realm", "myriad", "navigate the complexities".

**No hedges.**

Banned: "arguably", "perhaps", "in some sense", "one could argue". Make the claim or cut it.

**One voice per entry.**

Don't switch between first-person plural ("we", "our") and second-person imperative ("you", "run this") within the same tutorial. Pick one and stay in it.

## The diagram system

Tutorials use three diagram archetypes, rendered as ASCII inside fenced `text` blocks:

**1. Before/after diagram.** Side-by-side comparison of the task with and without the agent. Used once per entry, usually near the top:

```text
BEFORE                          AFTER

you → listen to recording       audio → agent → tasks[]
you → take notes                              ↓
you → forget action items       {owners, deadlines}
```

**2. Pipeline diagram.** The sequence of steps the agent performs. Used when the tutorial walks through a multi-step flow:

```text
[input]  →  [classify]  →  [route]  →  [act]  →  [verify]
                ↓             ↓           ↓          ↓
             anthropic      rules       tool       cueapi
```

**3. System diagram.** How the components connect. Used for tutorials that involve more than one service (a bus, a queue, an external API, a store):

```text
┌─────────┐    ┌──────────┐    ┌─────────┐
│  agent  │───▶│ inngest  │───▶│ mem0    │
└─────────┘    └──────────┘    └─────────┘
     │              │
     ▼              ▼
┌─────────┐    ┌──────────┐
│ cueapi  │    │  github  │
└─────────┘    └──────────┘
```

Draw the diagram by hand. Don't use a tool that outputs something fancier — the ASCII is deliberate; it renders the same way on GitHub, on the Builder Weekly site, on a terminal, and in every LLM's context window.

## License

CC BY 4.0. Your contributions are licensed under the same terms as the rest of the corpus. You retain attribution credit as `author` or `contributor` in the metadata. Fork freely, attribute always.
