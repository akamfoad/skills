Skills are organized into bucket folders under `skills/`:

- `engineering/` — daily code work
- `productivity/` — daily non-code workflow tools
- `misc/` — kept around but rarely used, not promoted
- `personal/` — tied to my own setup, not promoted
- `in-progress/` — drafts not yet ready to ship
- `deprecated/` — no longer used

Every skill in `engineering/` or `productivity/` (the **promoted** buckets) must have a reference in the top-level `README.md` and an entry in `.claude-plugin/plugin.json`'s `skills` array — the Claude Code plugin ships exactly the promoted set. Skills in `misc/`, `personal/`, `in-progress/`, and `deprecated/` must not appear in either.

The repo is also its own single-plugin Claude Code marketplace: `.claude-plugin/marketplace.json` lists the one `akamfoad-skills` plugin. When bumping the release version, keep `.claude-plugin/plugin.json`'s `version` in sync with `package.json`'s — Claude uses the plugin `version` to decide when installed users see an update. Run `claude plugin validate . --strict` after touching either manifest.

Each bucket folder has a `README.md` listing its skills with a one-line description, the name linked to its `SKILL.md`. Promoted buckets' READMEs (and the top-level `README.md`) group entries into **User-invoked** and **Model-invoked**; non-promoted bucket READMEs use a flat list.

Every `SKILL.md` is either user-invoked (`disable-model-invocation: true`, plus `policy.allow_implicit_invocation: false` in `agents/openai.yaml` for Codex) or model-invoked (model- or user-reachable, with rich trigger phrasing in its description).

Releases use [Changesets](https://github.com/changesets/changesets): run `npx changeset` to record a change; `.github/workflows/release.yml` opens the version PR on merge to `main`.

To (re)link every skill into the local harness skill directories (`~/.claude/skills`, `~/.agents/skills`), run `scripts/link-skills.sh`. Each entry is a symlink into this repo, so a `git pull` keeps installs current; re-run after adding, removing, or renaming a skill.

Once user-invoked skills pile up past what you can remember, add a router skill that maps them and when to reach for each (see the `writing-great-skills` guidance).
