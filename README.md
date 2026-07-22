# skills

My personal collection of [agent skills](https://docs.claude.com/en/docs/claude-code/skills).

[![skills.sh](https://skills.sh/b/akamfoad/skills)](https://skills.sh/akamfoad/skills)

## Install

Two ways, two philosophies.

### skills.sh — copy & hack

Copies editable skill files into your project. Works on Claude Code, Codex, and other Agent-Skills harnesses.

```bash
npx skills@latest add akamfoad/skills
```

Then pick the skills and agents you want.

### Claude Code plugin — subscribe

Installs the promoted skill set as a managed, versioned bundle that updates when I ship a new version — you don't edit it.

```bash
claude plugin marketplace add akamfoad/skills
claude plugin install akamfoad-skills@akamfoad
```

Or, inside Claude Code:

```
/plugin marketplace add akamfoad/skills
/plugin install akamfoad-skills@akamfoad
```

## Skills

### Engineering

**User-invoked**

- **[submit-review](./skills/engineering/submit-review/SKILL.md)** — Publish an in-session code review to a GitHub PR as staged (pending) inline comments, or as replies to existing review threads. Terse by default; verbosity set by the argument.

## Development

Skills live under `skills/<bucket>/<name>/SKILL.md`. `engineering/` and `productivity/` are **promoted** — they ship via the plugin and are listed above; `misc/`, `personal/`, `in-progress/`, and `deprecated/` are staging buckets that don't. Full conventions are in [CLAUDE.md](./CLAUDE.md).

Link every skill into your local harness directories for development:

```bash
./scripts/link-skills.sh
```

## License

[MIT](./LICENSE)
