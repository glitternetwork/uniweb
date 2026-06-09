# uniweb Skill

AI-powered integration assistant for the uniweb payment platform. Instead of reading docs, let your coding agent write the integration code.

## Install

### Claude Code

```bash
git clone https://github.com/glitternetwork/uniweb.git
mkdir -p ~/.claude/skills/uniweb
cp uniweb/skills/uniweb/SKILL.md ~/.claude/skills/uniweb/SKILL.md
```

Then ask Claude Code to use the `uniweb` skill:

```
Use the uniweb skill to add a checkout button to my Express app.
```

### Codex

```bash
git clone https://github.com/glitternetwork/uniweb.git
mkdir -p ~/.codex/skills
cp -R uniweb/skills/uniweb ~/.codex/skills/uniweb
```

Then invoke the skill explicitly:

```text
Use $uniweb to add a checkout button to my Express app.
```

## What it can do

- Generate checkout session integration code (Express, Next.js, any Node.js framework)
- Set up webhook handlers with signature verification
- Configure subscription billing with trials and dunning
- Create payment links for no-code payment collection
- Help with CLI commands and REST API calls
- Debug payment issues

## Prerequisites

You need a uniweb merchant wallet:

```bash
npm install -g @uniwebpay/cli
uniweb wallet create
```

## Links

- [uniweb Docs](https://vibecash.dev/docs)
- [API Reference](https://vibecash.dev/docs/api-reference)
- [Dashboard](https://vibecash.dev/dashboard)
