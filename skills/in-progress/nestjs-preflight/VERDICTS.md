# Verdicts

Shared rules for every preflight check.

## Verdict

- **PASS**: the property holds across the scope.
- **FAIL**: at least one place breaks the property. List every place, not the first one.
- **N/A**: the trigger is absent (no `@Cron`, no outbound calls, single-replica by design). Cite the search that proves absence.
- **UNVERIFIED**: the evidence lives outside the repo (cluster manifests, secret store, database settings). Name the value the human must read.

## Evidence rule

Each verdict cites proof:

- PASS cites the code or config that makes the property true, as `file:line`.
- FAIL cites each breaking `file:line`.
- Absence (PASS or N/A) cites the exact search and its hit count, e.g. `rg '@Cron\(' apps/orders/src`: 0 hits.

A check with a sub-item that fails is a FAIL. One row per check; sub-items go in the evidence cell.

## Severity

Each check file sets a default severity. Some checks raise it for specific sub-items.

- **critical**: security exposure, leaked secrets, or data loss or corruption.
- **high**: outage, dropped requests, or duplicate side effects under production conditions (replicas, rolling deploys, load, a slow dependency).
- **medium**: slower diagnosis, wasted capacity, or upgrade debt.

## Row format

`# | check | verdict | severity | evidence | fix`

The fix names the mechanism that fits the stack map and the house conventions, in one line.
