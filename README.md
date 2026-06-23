# Error discovery skill

This skill makes an AI agent run error analysis on a dataset. Point it at a
JSONL/CSV/JSON file of LLM outputs or traces, and it builds a review app, picks
diverse samples, and watches your annotations as you go.

You read and leave free-text notes. The agent sorts them into failure modes,
tracks coverage, and picks new samples to fill gaps.

The full instructions are in `SKILL.md`.

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
