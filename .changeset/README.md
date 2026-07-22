# Changesets

This folder is managed by [`@changesets/cli`](https://github.com/changesets/changesets), which versions and tags this repo.

To record a change, run `npx changeset`, describe it, and commit the generated file. On merge to `main`, `.github/workflows/release.yml` opens a version PR that bumps `package.json` (and, kept in sync by hand, `.claude-plugin/plugin.json`) and updates the changelog.
