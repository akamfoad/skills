---
name: submit-review
description: Publish an in-session code review to a GitHub PR as staged (pending) inline comments, or as replies to existing review threads. Terse by default; verbosity set by the argument.
disable-model-invocation: true
---

Publish a review you have **already produced this session** to a GitHub PR. This skill submits findings — it does not generate them; work from the current session's review plus any instructions the user adds.

Two branches:

- **Fresh review** — stage this session's findings as **pending** inline comments, then submit them under one review.
- **Replies** — answer existing review threads on the PR.

The argument sets verbosity (default `very-concise`): `/submit-review [very-concise|concise|relaxed|dense] [PR number or URL]`.

## Every comment is high-signal

Lead with the point. No preamble, no praise-padding, no restating the code back at the author. This holds at every verbosity — the level sets *length*, not whether you ramble.

| level | budget |
|---|---|
| `very-concise` (default) | One line: the point, or point + fix. |
| `concise` | One–two sentences: the point plus a why or a fix. |
| `relaxed` | A short paragraph: point, why it matters, a concrete fix — a ` ```suggestion ` block where it helps. |
| `dense` | Maximum information, tightly packed: rationale + edge case + fix, zero filler. Longer only because it carries more, never because it is chattier. |

## Inline only, verdict COMMENT

By default the review carries **no general comment** and the neutral **verdict** `COMMENT`. Add a review body, or switch the verdict to `APPROVE` / `REQUEST_CHANGES`, only when the user says so.

## Process

### 1. Preflight

Confirm an active login:

```bash
gh auth status --json hosts --jq ".hosts|add|.[].active"
```

Expect `true`. Empty, `false`, or an error → stop and tell the user to run `gh auth login`.

Then resolve the PR's `owner`, `repo`, `pull_number`, and head SHA (the `commit_id` comments anchor to). Default to the current branch's PR; honor a PR number/URL passed in the argument. Commands: [`github-api.md`](github-api.md).

### 2. Draft the inline comments

Turn each finding into exactly one inline comment: a `path`, a `line` (plus `side`, and `start_line`/`start_side` for a range), and a `body` within the verbosity budget. Only lines present in the PR diff can be anchored.

Done when every finding you intend to raise is one drafted comment with a resolved path + line, and none exceeds the budget.

### 3. Stage as one pending review

Send all drafted comments in a single `reviews` call with **no** `event` — the review is created **pending**, invisible to the PR author. Capture the returned review `id`. Payload shape: [`github-api.md`](github-api.md).

Done when the staged comment count equals what you drafted (verify by listing the review's comments). If it differs, fix and re-stage — do not go on.

### 4. Confirm — the publish gate

Show the user every staged comment (`path:line` + body) and the intended verdict. Submitting is what makes them visible to the author; until then the pending review is fully reversible — delete it and re-stage to redo.

Get an explicit go-ahead. Without it, do not call submit.

### 5. Submit

Submit the pending review by `id` with the verdict (default `COMMENT`; add a body only if the user asked). Report the returned review URL. Command: [`github-api.md`](github-api.md).

## Replies to existing threads

When the user is answering review threads rather than opening a fresh review:

1. List the PR's review comments to get the thread/comment `id`s. ([`github-api.md`](github-api.md))
2. Draft each reply within the verbosity budget.
3. Replies publish immediately — there is no pending stage. Show the user each reply with its target and get an explicit go-ahead first.
4. Post each as a reply under its comment `id`.

## Commands

Every GitHub call goes through `gh`. Prefer a native `gh` subcommand; fall back to `gh api` where none exists — staging pending comments, submitting a pending review, and posting replies all require `gh api`. Never hand-roll the API with `curl`. Exact invocations and JSON payloads live in [`github-api.md`](github-api.md).
