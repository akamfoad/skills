---
name: nestjs-preflight
description: Go/no-go preflight of a NestJS app or module before a deploy to dev, stage, or prod. 17 checks from the NestJS production checklist.
disable-model-invocation: true
---

# NestJS preflight

A **preflight** is a go/no-go gate. Each check names a **property** that must be true in production (many replicas, rolling deploys, a shared database, dependencies that hang). The verdict is on the property, never on the package: any mechanism that makes the property true is a PASS.

Source: [NestJS Production Checklist: 17 Checks Before You Deploy](https://dev.to/nestjs/nestjs-production-checklist-17-checks-before-you-deploy-1d0j) (Kamil Mysliwiec, 2026-09-30). Check numbers follow the article.

## 1. Fix the target

Get three facts from the user's request. Ask only for what is missing.

- **App**: the deployable app (in a monorepo, one app, e.g. `apps/<name>`).
- **Scope**: the whole app, named modules, or a diff (`git diff <base>...HEAD`). Checks tagged `module` inspect only the scope. Checks tagged `app` always inspect the host app, because the module ships inside it.
- **Environment**: `dev`, `stage`, or `prod`. It sets the gate in step 4.

## 2. Map the stack

Record these facts with the file that proves each one. Every check reads them.

- Nest major (`@nestjs/core` in the lockfile) and HTTP adapter (express or fastify). Nest 12 has `useSecurityHeaders()`, `return503OnClosing`, `StandardSchemaValidationPipe`. Older majors need the fallback mechanism each check names.
- ORM and migration tool, queue library, `@nestjs/schedule` use, outbound HTTP client.
- Where deploy config lives: Dockerfile, Kubernetes manifests or Helm chart, CI pipeline. Config outside the repo makes its checks UNVERIFIED, never PASS.
- House conventions in `CLAUDE.md` / `AGENTS.md` / `docs/`. They choose the mechanism (custom auth guard, env validation library, error model). The property stays the same.

## 3. Run the checks

Dispatch one subagent per group, in parallel. Each subagent gets the target, the stack map, its group file, and [`VERDICTS.md`](VERDICTS.md), and returns one row per check.

| Group | File | Checks | Where the evidence lives |
|---|---|---|---|
| Platform | [`checks/platform.md`](checks/platform.md) | 1, 2, 3, 10, 16, 17 | `main.ts`, Dockerfile, manifests, CI, `package.json` |
| Edge | [`checks/edge.md`](checks/edge.md) | 5, 6, 7, 8, 9 | guards, controllers, DTOs, pipes, interceptors |
| Data | [`checks/data.md`](checks/data.md) | 4, 11, 12, 13, 14, 15 | cron, queues, outbound calls, DB config, migrations, DI scopes |

With no subagent tool, run the groups in sequence and finish every row of a group before you open the next file.

Done when all 17 rows exist, and each one meets the evidence rule in `VERDICTS.md`.

## 4. Gate and report

Apply the gate for the environment:

| Environment | NO-GO when |
|---|---|
| `dev`, `stage` | any `critical` FAIL |
| `prod` | any `critical` or `high` FAIL, or any `critical`/`high` check left UNVERIFIED |

Report in this order:

1. **GO** or **NO-GO**, with the rows that decide it.
2. The 17 rows as one table: `# | check | verdict | severity | evidence | fix`.
3. **Confirm outside the repo**: each UNVERIFIED row as one question with the exact value to read (for example "`terminationGracePeriodSeconds` on the `orders-api` Deployment: is it ≥ preStop + longest job?").

Fixes stay as proposals in the report. Change code only when the user asks.
