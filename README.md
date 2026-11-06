# PrintResize skill

An [Agent Skill](https://agentskills.io) that lets an AI agent turn an image into a print file with the [PrintResize](https://printresize.com) API.

Ask in plain words, for example "make swatch.png a 120 × 40 cm table runner". The agent picks the tool, runs a free price check, asks you before spending credits, then polls the task and saves the file.

## What it can do

| You ask for | Tool |
|---|---|
| An exact print size in centimetres (10 to 240 cm per side), rectangle, square, round or oval | Print Resize |
| A larger image at 2K, 4K or 8K | Upscale |
| A pattern that tiles without a visible seam | Seamless Pattern |
| A tablecloth or placemat mockup, a new scene for a finished product, or a flat pattern pulled from a product photo | Home textiles |

## How the agent behaves

- Runs a price check with `dryRun=true` before every paid run. Nothing is charged.
- Waits for your yes before submitting, unless you gave it a spending limit that covers the run.
- Sends a `clientRequestId` with each run, so a retry after a timeout returns the same task instead of charging twice.
- Reads your key from an environment variable and never asks you to paste it into the chat.

## Requirements

- A PrintResize account and an API key from [printresize.com/api-keys](https://printresize.com/api-keys).
- Credits on that account. Runs are billed per run; prices are on the [pricing page](https://printresize.com/docs/pricing).

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add ivanvolt-labs/printresize-skill
```

As a Claude Code plugin:

```bash
claude plugin marketplace add ivanvolt-labs/printresize-skill
claude plugin install printresize@printresize
```

Or copy the file by hand:

```bash
mkdir -p ~/.claude/skills/printresize
curl -fsSL https://printresize.com/skills/printresize/SKILL.md \
  -o ~/.claude/skills/printresize/SKILL.md
```

Then set your key in the shell your agent runs in:

```bash
export PRINTRESIZE_API_KEY=pr_your_key_here
```

## Docs

- [Agent skill guide](https://printresize.com/docs/agent-skill)
- [API reference](https://printresize.com/docs)

The skill file here is a copy of [printresize.com/skills/printresize/SKILL.md](https://printresize.com/skills/printresize/SKILL.md), which is the canonical version.

## License

MIT
