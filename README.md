# build-on-aster

A [Claude Code](https://claude.com/claude-code) **skill** that teaches your AI coding assistant to build and run agents on the [Aster Agents](https://www.asteragents.com) platform — entirely through the public REST API, using your organization's API key. No platform-internal access required.

With this skill installed, you can ask Claude Code (or any agent that reads `SKILL.md`) to:

- **Build agents** — create and update agents with the right tools, models, knowledge bases, and skills wired together
- **Stand up knowledge bases** — create a KB and run the full upload → associate → process pipeline, polling to searchable
- **Run agents** — invoke an agent and read back its answer (the `POST /chat` → poll loop)
- **Package skills** — bundle a `SKILL.md` + files and attach them to agents
- **Manage your org** — invitations, tags, and tool-schema syncs

It knows the platform's real contracts (verified against the implementation, not just the spec) and the gotchas that burn first-time integrators.

## Install

This repo is a Claude Code **plugin marketplace**. From inside Claude Code:

```
/plugin marketplace add asterlabs-ai/build-on-aster
/plugin install build-on-aster@aster
```

Or from the command line:

```bash
claude plugin marketplace add asterlabs-ai/build-on-aster
claude plugin install build-on-aster@aster
```

`/plugin update` pulls new versions as they ship. Claude invokes the skill automatically when a task involves managing or running an Aster org; you can also call it explicitly with `/build-on-aster:build-on-aster`.

> **Prefer a plain skill?** Clone the repo and point any SKILL.md-aware tool at `SKILL.md` (it's plain Markdown with a companion `reference.md`):
> ```bash
> git clone https://github.com/asterlabs-ai/build-on-aster ~/.claude/skills/build-on-aster
> ```

## Get an API key

1. Go to **Control Hub → Settings → API Access** at [asteragents.com](https://www.asteragents.com)
2. Copy your key (a bearer token, scoped to one organization, valid for 6 months)
3. Export it where Claude Code can reach it:

```bash
export ASTER_API_KEY="<your-key>"
```

Never hardcode the key; never commit it.

## Quick smoke test

```bash
curl -s https://www.asteragents.com/api/agents \
  -H "Authorization: Bearer $ASTER_API_KEY" | head
```

A JSON array of your org's agents means you're connected.

## What's in here

| File | Purpose |
|---|---|
| `SKILL.md` | The teaching layer — workflows, behavioral rules, and gotchas |
| `reference.md` | Full endpoint reference — exact request/response shapes for every operation |
| `.claude-plugin/` | Plugin + marketplace manifests (`plugin.json`, `marketplace.json`) |

For agent-building concepts (prompting, model choice, tools, knowledge bases), the skill points your assistant at the live docs: [docs.asteragents.com](https://docs.asteragents.com).

## Support

Questions or issues: [asteragents.com/support](https://www.asteragents.com/support).
