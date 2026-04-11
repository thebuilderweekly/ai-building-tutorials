# Merge Gate

Every PR against this repo passes through a **seven-check automated gate** before a human reviewer sees it. The same gate applies to in-house pipeline-generated entries and community contributions — no exceptions.

The gate fails loud. A failing PR won't be merged, and the failure reason is left as a CI comment so the contributor can fix it and push again.

---

## The scope rule

Before the seven checks run, every PR is sized. A PR is in-scope if **either**:

- It touches **one entry folder** under `entries/` (the folder itself is `entries/{id}/`), or
- It touches **fewer than three files outside any entry folder** (for example: a fix to `CONTRIBUTING.md` + a fix to `README.md` = 2 files, accepted).

Any PR larger than this is rejected at the scope check with an explanation. Large refactors, multi-entry touch-ups, and bulk queue rewrites go through separate RFC-style issues, not a single mega-PR.

---

## The seven checks

### 1. Schema validation

Every entry has a `metadata.json` file. The validator asserts that file parses cleanly against [`schema/tutorial.schema.json`](./schema/tutorial.schema.json):

- Every required field is present.
- No unknown fields (additionalProperties is false).
- Every field matches the declared type and format (slug is kebab-case, dates are ISO, `pillar` is one of the six enum values, `status` is one of the five lifecycle states).

**Common failures:**
- `title` longer than 120 characters
- `id` has underscores instead of hyphens
- Extra field you added for yourself (`internal_note`, `draft_history`) — move it into the entry body or strip it
- `pillar` set to `"accountablility"` or any typo outside the enum

### 2. Soul line completeness

The `soulLine` field must be present, at least 20 characters, and contain at least one reference to a **specific measurable advantage**: speed, recall, consistency, cost, availability, throughput, or concurrency.

A generic statement like "an agent is faster and more reliable" fails. A statement like "an agent classifies a thousand issues per hour with deterministic rules, which a human maintainer gets through maybe thirty before burning out" passes.

**Common failures:**
- "An agent is better at this than a human." — no specific dimension, no number, no explanation
- "AI can help with this task." — marketing voice, no claim
- Soul line is a restatement of the `title` with one adjective changed

### 3. Before/after state completeness

Both `beforeState` and `afterState` must be present, each at least 20 characters, and the two must describe **different states** (not the same state with different adjectives).

The check enforces difference by asserting the two strings share fewer than 60% of their tokens by length.

**Common failures:**
- `afterState` is "The task is automated." — too abstract, no concrete transformation described
- `beforeState` and `afterState` describe the same workflow with "slow" swapped for "fast"
- Either state is a bullet list instead of prose

### 4. Voice violation scan

**Scope:** the voice scan runs only on published tutorial prose. Specifically: `entries/**/tutorial.md` and the `soulLine`, `beforeState`, and `afterState` fields of each entry's `metadata.json`. Contributor-facing documentation (`README.md`, `CONTRIBUTING.md`, `MERGE_GATE.md`, pillar READMEs, `llms.txt`) is technical writing for developers and is not in scope. If you are editing those files, you can use em dashes and connecting phrases without failing the gate. The voice rules exist to keep published tutorials in Builder Weekly's house voice, not to police the contribution docs.

Every in-scope file is scanned against a banned-phrase list. The banned list is identical to Builder Weekly's cross-publication voice rules and covers:

- **Typography**: em dashes (`—`), en dashes (`–`), double hyphens (`--`)
- **Scaffolding phrases**: "in conclusion", "it's worth noting", "furthermore", "moreover", "delve", "tapestry", "landscape", "realm", "myriad", "navigate the complexities"
- **Marketing words**: "unleash", "pivotal", "crucial", "cutting-edge", "next-generation", "AI-powered", "seamlessly", "revolutionary", "game-changer", "leverage" (as a verb)
- **Hedges**: "arguably", "perhaps", "in some sense", "one could argue"

One occurrence is a fail. The message tells you the phrase and the line number.

**Common failures:**
- An em dash you didn't notice in the middle of a paragraph (these are the #1 fail — write with hyphens and commas)
- "delve into" — replace with "examine" or "walk through" or just cut the phrase
- "leverage" used as a verb — replace with "use"

### 5. Tool provenance

Every tool listed in `metadata.json`'s `tools` array must be either:

- An entry in Builder Weekly's [Agent Index](https://github.com/thebuilderweekly/the-agent-index) (matched by case-insensitive slug), or
- An entry in the built-in allowlist of well-known tools (`curl`, `git`, `node`, `python`, `docker`, `bash`, `jq`, `ffmpeg`, and a small set of other standard developer tooling).

An entry can't quietly depend on a tool that hasn't been vetted by the Agent Index's programmatic-path criterion, or one that's been deliberately cut (Twilio, Stripe, Bright Data, etc.).

**Common failures:**
- Tutorial uses Zapier — not in the Agent Index, doesn't pass the programmatic-path test, rejected
- Tutorial uses Twilio SMS — Twilio was cut from the Agent Index for A2P 10DLC gating, rejected
- Tool slug typo (`mem-0` instead of `mem0`)

### 6. Code block annotations

Every fenced code block in the entry's `tutorial.md` must have a language annotation:

````markdown
```bash
curl https://api.example.com/endpoint
```
````

Not:

````markdown
```
curl https://api.example.com/endpoint
```
````

This is a hard rule, not a style preference. Readers who copy-paste rely on the annotation for syntax highlighting and for the site's "copy" button to recognize the block as executable. Unlabeled blocks are skipped by both, which silently degrades the reader experience.

**Common failures:**
- A Claude-generated entry with half the blocks labeled and half bare
- An ASCII diagram inside a `text`-annotated block — that's fine, `text` counts

### 7. Freshness stamp

If the PR is an **update** to an existing entry (not the entry's first appearance), `lastVerifiedAt` in `metadata.json` must be advanced to the date of the PR, and the PR description must include one sentence about what was re-verified.

If the PR is the **creation** of a new entry, both `createdAt` and `lastVerifiedAt` must be set to the same date (the PR's date).

This prevents drift between the metadata's verification timestamp and the reality of whether anything was actually re-checked.

**Common failures:**
- Editing `tutorial.md` without touching `metadata.json`'s `lastVerifiedAt`
- Setting `lastVerifiedAt` forward without the one-sentence PR description explaining what was verified

---

## What the gate does NOT check

Two things that are a human reviewer's job, not the gate's:

- **Correctness of the technical claim.** The gate can check that you have a soul line; it can't check that the soul line is actually true. The reviewer does that, usually by running the tutorial end to end against the listed tools.
- **Pillar placement.** The gate validates that `pillar` is in the enum; it doesn't second-guess whether your accountability tutorial is really a memory tutorial in disguise. The reviewer nudges pillar placement during review.

---

## Running the gate locally

The same checks run on every PR via GitHub Actions. To avoid the round trip, run them locally before you push:

```bash
# Schema validation for every entry
./scripts/check-schema.sh

# Voice scan for every markdown file
./scripts/check-voice.sh

# Tool provenance for every entry
./scripts/check-tools.sh
```

(These scripts ship in Phase 11B once the first entries land. Until then, the CI workflow is the only gate.)

---

## Why the gate exists

A corpus is only as tight as its admission bar. Without the gate, entries drift toward "content marketing about AI" within three PRs and the corpus loses the shape that makes it worth reading in the first place. Every check here is a rule we wrote down after seeing it get violated by default.
