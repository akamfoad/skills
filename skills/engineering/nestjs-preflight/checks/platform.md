# Platform checks

Checks 1, 2, 3, 10, 16, 17. Evidence lives in `main.ts`, the Dockerfile, deploy manifests, CI, and `package.json`. Apply [`../VERDICTS.md`](../VERDICTS.md).

## 1. Config validated at boot, secrets kept out of the image

`app` · high; critical for a secret in the image or in logs

**Property:** a missing or malformed env var stops the process before it serves a request. Secrets reach the process only at runtime.

**Evidence:**
- `ConfigModule.forRoot` has a `validationSchema` (Nest 12 `@nestjs/config` takes any Standard Schema: Zod, Valibot, ArkType) or a `validate` function (class-validator, Joi).
- Required values use `config.getOrThrow(...)`. Each `config.get(...)` of a required value is a FAIL.
- Dockerfile has no secret in `ARG`/`ENV` and copies no `.env*`. `.dockerignore` lists `.env*`.
- No log of the whole config or `process.env`: search `console.log(config`, `logger.*(config`, `JSON.stringify(process.env`. Logging keys only is a PASS.

**Fix:** schema validation on the config module, `getOrThrow` for required keys, secrets from the platform store (Kubernetes Secrets, AWS Secrets Manager, Vault).

**Why boot-time:** in a rolling deploy, a version that fails at boot never becomes ready, so the old version keeps serving.

## 2. SIGTERM reaches Nest, and the container fits production

`app` · high

**Property:** a deploy, scale-down, or node drain runs Nest's shutdown hooks to the end, and the process dies with a JavaScript error, not a silent kill.

**Evidence:**
- `app.enableShutdownHooks()` in `main.ts`. Without it, `onModuleDestroy` and `onApplicationShutdown` never run: pools stay open, queue workers die mid-job, in-flight requests drop.
- Dockerfile `CMD`/`ENTRYPOINT` uses exec form with `node` first: `CMD ["node", "dist/main.js"]`. Shell form (`CMD node dist/main.js`) and `npm run ...` put a process in front of Node that does not reliably forward signals.
- PID 1: Node as PID 1 ignores the re-sent signal Nest uses to exit. PASS needs one of: `tini` or `--init` as entrypoint, or `enableShutdownHooks(undefined, { useProcessExit: true })`.
- Multi-stage build; dev dependencies and source stay out of the final stage (`npm prune --omit=dev` or a production-only install).
- `ENV NODE_ENV=production` and `USER node` (or another non-root user) in the final stage.
- Heap limit: `NODE_OPTIONS=--max-old-space-size=<n>` at about 75% of the container memory limit (768 for 1 GiB). Without it the kernel OOM-kills the process: exit code 137, no log line. UNVERIFIED when the memory limit lives outside the repo.

**Shutdown order with hooks on:** HTTP adapter marked closing, `onModuleDestroy`, `beforeApplicationShutdown`, HTTP server, gateways, microservices close, then `onApplicationShutdown` (where `@nestjs/bullmq` closes workers and waits for running jobs).

**Fix:** also alert on restarts and on `OOMKilled`.

## 3. Pods drain before they close

`app` · high

**Property:** no request is refused while the load balancers still route to a terminating pod, and the grace period covers the longest in-flight job.

**Evidence:**
- A `preStop` delay on the container (`exec: ["sleep", "5"]`). Endpoint removal and `SIGTERM` start at the same time; the removal takes seconds to reach every load balancer and kube-proxy.
- `terminationGracePeriodSeconds` (default 30) ≥ preStop + longest job + buffer. The preStop time counts against it. After it, `SIGKILL`; BullMQ marks the job stalled and another worker reruns it from the start.
- Nest 12 `return503OnClosing: true` on `NestFactory.create()` is a complement only. During propagation its 503s reach clients, so the preStop delay is still required.
- Manifests outside the repo: UNVERIFIED.

**Fix:** add the preStop delay; raise the grace period or shorten jobs; make every job idempotent (a crash also reruns it).

## 10. Node, Nest, and dependencies are current

`app` · medium; high for an end-of-life Node or a high/critical `npm audit` finding

**Property:** the runtime and packages receive security patches.

**Evidence:**
- Node major in the base image, `.nvmrc`, `engines`: an LTS line in Active or Maintenance. Odd majors never become LTS. Node 20 reached end of life in April 2026.
- Base image pins the major (`node:24-slim`) and CI rebuilds it on a schedule, so patch releases reach production.
- Every `@nestjs/*` package resolves to one matching version in the lockfile. A mismatch between `@nestjs/common` and `@nestjs/core` causes confusing DI and metadata errors.
- Lockfile committed; CI and the image install with `npm ci`.
- Renovate or Dependabot config exists.
- `npm audit --omit=dev` runs in CI. Run it now and cite the result.

**Fix:** small continuous update PRs, not one large upgrade.

## 16. Errors keep their status, their log line, and their source location

`app` · medium; high when a custom filter hides unhandled errors from logs

**Property:** `HttpException` returns its real status, every other error returns an opaque 500 and writes a log line with a stack, and stack traces point at `.ts` lines.

**Evidence:**
- Each `@Catch()` filter either extends `BaseExceptionFilter` and calls `super.catch(...)`, or logs unknown errors with the stack itself. A filter written from scratch drops Nest's `ExceptionsHandler` log line.
- Unhandled promise rejections crash the process (Node default since v15). This is correct. Check that the error is logged with its stack before exit, and that restarts alert someone.
- Source maps: build emits them (`sourceMap: true`, or the bundler equivalent), the maps ship in the image, and Node starts with `--enable-source-maps` (flag or `NODE_OPTIONS`).

**Fix:** delegate to `BaseExceptionFilter`; add `--enable-source-maps` to the start command.

## 17. Logs, health checks, monitoring

`app` · medium; high when liveness depends on an external dependency

**Property:** production failures are visible, and a dependency outage does not restart every pod.

**Evidence:**
- Logs are JSON with production log levels, structured params, and a trace id on every line (trace id propagated via `AsyncLocalStorage` or `nestjs-cls`).
- Liveness and readiness are separate endpoints. Liveness checks only the process itself. Readiness holds the dependency checks (`@nestjs/terminus` database ping). A liveness that pings the database restarts every pod when the database goes down, then adds a reconnect storm when it returns.
- Probe paths in the manifests match the two endpoints (UNVERIFIED when manifests are outside the repo).
- Monitoring covers per-route latency and error rate, traces down to queries, grouped errors with an alert on new ones, and job and cron runs. Health probe routes are excluded from latency metrics.

**Fix:** split `/health/live` and `/health/ready`; move dependency indicators to readiness.
