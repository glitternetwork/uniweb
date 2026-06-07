# uniweb Skill for Claude Code

AI-powered integration assistant for the uniweb payment platform. Instead of reading docs, let Claude write your integration code.

## Install

### Option 1: Use as plugin (recommended)

```bash
git clone https://github.com/glitternetwork/uniweb.git
claude --plugin-dir ./uniweb
```

### Option 2: Copy to personal skills

```bash
mkdir -p ~/.claude/skills/uniweb
cp skills/uniweb/SKILL.md ~/.claude/skills/uniweb/SKILL.md
```

### Option 3: Copy from docs

Visit [vibecash.dev/docs/skill](https://vibecash.dev/docs/skill) and copy the SKILL.md content directly.

## Usage

In Claude Code, use the `/uniweb` command:

```
/uniweb "add a checkout button to my Express app"
/uniweb "set up subscription billing with a 14-day trial"
/uniweb "handle webhook events for payment and subscription"
/uniweb "add WeChat Pay and Alipay to my checkout"
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
npm install -g uniweb
uniweb wallet create
```

## Links

- [uniweb Docs](https://vibecash.dev/docs)
- [API Reference](https://vibecash.dev/docs/api-reference)
- [Dashboard](https://vibecash.dev/dashboard)
