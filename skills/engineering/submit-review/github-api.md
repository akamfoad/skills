# GitHub review commands

Every call goes through `gh`. `gh api` uses the active host and auth automatically. Substitute `{owner}`, `{repo}`, `{pull_number}`, `{review_id}`, `{comment_id}`.

## Resolve the PR

```bash
# base repo (owner/repo) — the repo the PR lives in
gh repo view --json nameWithOwner --jq .nameWithOwner

# PR number + head SHA (commit_id) for the current branch,
# or pass a number/URL: gh pr view <number|url> --json ...
gh pr view --json number,headRefOid,url --jq '{number, headRefOid, url}'
```

For a PR in another repo, add `--repo {owner}/{repo}` to both commands.

## Fresh review

### Stage as one pending review

A nested `comments` array can't be expressed with `-f`, so build JSON and pass it with `--input`. Omitting `event` leaves the review **pending** (invisible to the PR author).

```bash
cat > "$SCRATCH/review.json" <<'JSON'
{
  "commit_id": "<HEAD_SHA>",
  "comments": [
    { "path": "src/foo.ts", "line": 42, "side": "RIGHT", "body": "…" },
    { "path": "src/bar.ts", "start_line": 10, "start_side": "RIGHT", "line": 14, "side": "RIGHT", "body": "…" }
  ]
}
JSON

gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews \
  --method POST --input "$SCRATCH/review.json" \
  --jq '{id, state}'          # expect state = PENDING; keep id
```

`$SCRATCH` = the session scratchpad dir. Write the payload there, not in the repo.

Verify every comment landed before the gate:

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/comments --jq 'length'
```

### Submit (after the user's go-ahead)

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/events \
  --method POST -f event=COMMENT \
  --jq '{state, html_url}'
```

`event` ∈ `COMMENT` (default) · `APPROVE` · `REQUEST_CHANGES`. Add a summary only when asked: `-f body='…'`.

### Redo — delete a pending review

If the user rejects at the gate, drop the pending review and re-stage:

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id} --method DELETE
```

## Replies to existing threads

List review comments to find the thread roots and their ids (roots have no `in_reply_to_id`):

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments --paginate \
  --jq '.[] | {id, path, line, in_reply_to_id, user: .user.login, body}'
```

Reply under a comment id (publishes immediately — confirm first):

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments/{comment_id}/replies \
  --method POST -f body='…' --jq '{id, html_url}'
```

## Field notes

- **`line` / `side`** — `line` is the line number as shown in the diff; `side` is `RIGHT` (the new file: added or context lines) or `LEFT` (the old file: removed lines). For a range, add `start_line` + `start_side`. Only lines present in the diff can be anchored — a bad line returns `422`.
- **`commit_id`** — the PR head SHA (`headRefOid`); anchors the comments to that commit.
- Prefer `line` / `side` over the legacy `position` field.
- **One-shot alternative** — adding `"event": "COMMENT"` to the stage payload creates *and* submits in one call, but that skips the pending gate. Use the two-phase flow above so staged comments can be reviewed before they go public.
