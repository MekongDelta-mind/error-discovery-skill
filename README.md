# Error discovery skill

This skill makes an AI agent run interactive error analysis on any dataset of
LLM outputs or traces. The full instructions are in `SKILL.md`.

The agent studies the data, builds a review UI, selects diverse samples, and
monitors the human's annotations live. It categorizes free-text notes into
failure modes, tracks coverage, and proposes new samples to fill gaps.

## What it does

1. Reads the dataset and figures out the content type (articles, agent traces,
   code, structured output, etc.).
2. Designs visual encoding based on what varies in the data. Uses Gestalt
   principles (color for categories, spacing for hierarchy, opacity for
   importance).
3. Builds a single-file HTML review app served by a Python stdlib server. No
   dependencies.
4. Clusters the data and picks a diverse initial sample (cluster reps + random
   picks).
5. Runs an interactive loop: monitors annotations, categorizes failure modes,
   proposes new samples to increase coverage.

The human's job is to read and leave free-text notes. The agent organizes those
notes into a failure mode taxonomy.

## What is in here

- `SKILL.md`: the skill, with all phases and rules.

## How to use it

`SKILL.md` is a plain markdown file. Any agent that can read a file can follow
it.

- Claude Code reads skills from `~/.claude/skills`. Clone the repo into a folder
  named `error-discovery`:

```
git clone https://github.com/shreyashankar/error-discovery-skill ~/.claude/skills/error-discovery
```

- Other agents can use the rules too. Paste the contents of `SKILL.md` into
  whatever instructions that agent reads.

Then point the agent at a dataset:

```
Can you help me do error analysis on traces.jsonl?
```

The agent builds the review app, opens it in the browser, and starts the
interactive loop.
