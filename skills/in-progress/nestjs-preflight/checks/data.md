# Data checks

Checks 4, 11, 12, 13, 14, 15. Evidence lives in cron and queue code, outbound calls, DB config, migrations, and provider scopes. Apply [`../VERDICTS.md`](../VERDICTS.md).

## 4. Scheduled jobs run once across replicas

`module` · high

**Property:** each scheduled tick runs on exactly one replica, runs never overlap, and every job is safe to run twice.

**Evidence:**
- Search `@Cron(`, `@Interval(`, `@Timeout(`, and `SchedulerRegistry` use. `@nestjs/schedule` runs in-process: 3 replicas run the nightly export 3 times. A dev instance never shows it.
- Each hit has a cross-replica lock: `@nestjs/locks` `@OnOneInstance({ key })` (lease renewed every `ttl / 3`, 30 s TTL default, failover after the TTL) and/or `@WithoutOverlapping()` (lock for one run: covers the 5-minute job that sometimes takes 7). Without `@nestjs/locks`: a Postgres advisory lock or Redis lock with a TTL, or the job moved to a single-run platform scheduler (Kubernetes CronJob).
- The lock store is shared (Redis or Postgres). The in-memory store excludes only callers in one process; `LocksModule` refuses it in production unless `allowInMemoryStorage` is set, which is a FAIL.
- Jobs process current state ("export everything not yet exported"), not a time window ("yesterday's rows"). Missed ticks are never replayed.
- Long jobs pass the lock's `signal` to their I/O and use its `fencingToken` to reject writes from a holder whose lease was taken over.
- Queues (BullMQ): `defaultJobOptions` sets `attempts` (BullMQ does not retry by default), `backoff`, `removeOnComplete`, `removeOnFail` (otherwise jobs stay in Redis forever). A worker that dies or blocks the event loop past its lock gets its job marked stalled and rerun, so each processor must be idempotent.

**Fix:** a lock on every scheduled method, a shared lock store, explicit queue job options, idempotent processors.

## 11. Every outbound call has a timeout

`module` · high

**Property:** no request waits on a dependency longer than its budget. A hung dependency holds memory, a socket, and often a DB connection per waiting request, and exhausts them under load.

**Evidence:**
- `fetch`: each call passes `signal: AbortSignal.timeout(<ms>)`. Default wait for headers is 5 minutes.
- `@nestjs/axios` / axios: `timeout` set on `HttpModule.register(...)` or per request. Axios default `timeout` is 0, meaning no timeout.
- `@nestjs/http-client` (Nest 12): `timeout` per client in `HttpClientModule.register({ name, baseUrl, timeout: '5s' })`. The timeout is per attempt, and it retries idempotent methods (`GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`) 3 times with backoff, timeouts included: 5 s can mean 15 s or more.
- Database: `statement_timeout` in the `pg` pool config, whatever ORM sits on top.
- Other clients (Redis, SDKs, gRPC): each has an explicit timeout.
- Sum the worst case per call path: timeout × attempts × retry layers. It fits the caller's own timeout.

**Fix:** a timeout on every client, sized to the caller's budget.

## 12. A dependency outage degrades, not cascades

`module` · medium; high for a dependency on the hot path of a core route

**Property:** a dependency that is down for minutes gets fewer calls, holds a bounded share of capacity, and the caller gets a reasonable answer.

**Evidence:**
- `@nestjs/resilience`: a preset in `ResilienceModule.forRoot({ presets })` with `timeout`, `retry`, `circuitBreaker` (`failureRateThreshold`, `minimumCalls`, `openDuration`). Handlers use `@Resilience('<preset>')`, `@Fallback(...)`, `@Bulkhead()`. Non-handler code (jobs, cron) uses `ResilienceService.preset('<preset>').execute(...)`. An open breaker fails fast with 503 and `Retry-After`. Other libraries (cockatiel, opossum) pass on the same property.
- The injected `signal` (`@Signal()`) reaches the I/O, so a timed-out attempt stops its work.
- One layer owns retries. A resilience retry around an HTTP client that retries itself gives 3 × 3 = 9 attempts.
- `@Retry()` applies to safe methods only, unless the handler is also `@Idempotent()` and the retry has `idempotent: true`.

**Fix:** a breaker and bulkhead per unreliable dependency, a fallback where a degraded answer exists, retries in one layer.

## 13. Migrations run as a separate, backward-compatible deploy step

`app` · critical for schema sync against a shared database; high for the rest

**Property:** production schema changes come only from reviewed migrations, applied once before the new version starts, and the previous image still works on the new schema.

**Evidence:**
- No schema sync outside local dev: TypeORM `synchronize: true`, `drizzle-kit push`, `prisma db push`. A sync applies whatever diff it computes, and a renamed field can become a drop plus an add.
- Reviewed migration files are in version control and applied by `drizzle-kit migrate`, TypeORM migrations, or `prisma migrate deploy`.
- They run as one step before rollout: a Kubernetes Job, a release phase, a CI step. Migration at app startup (`migrationsRun: true` or equivalent) with several replicas makes them race. A DB advisory lock around a startup migration removes the race but still couples boot to the migration.
- Every migration in scope is backward compatible: no drop or rename of a column, table, or enum value that the running version still reads. Breaking changes split into expand (add) and contract (drop) across two deploys.
- Rollback means redeploying the previous image against the current schema. A migration that blocks this is a deliberate, reviewed exception.

**Fix:** migrate in a pre-deploy step; expand/contract for breaking changes.

## 14. Connection pool math fits the database

`app` · high

**Property:** (pool size × peak replicas during a deploy) + workers + migration job + admin tools + headroom ≤ database `max_connections`.

**Evidence:**
- Pool size is explicit (`pg` pool `max`, however the ORM passes it). Defaults differ per ORM.
- Peak replicas include the rolling-deploy overlap (old and new pods at once) and autoscaler max.
- Postgres default `max_connections` is 100: 5 replicas × 20 already reaches it.
- Replica counts and `max_connections` outside the repo: UNVERIFIED, with the formula filled in as far as the repo allows.

**Fix:** explicit pool size; past the limit, add a pooler (PgBouncer) rather than raising `max_connections`.

## 15. No unintended request-scoped chains

`module` · medium

**Property:** providers are singletons unless per-request instances are needed by design.

**Evidence:**
- Search `Scope.REQUEST`, `scope: Scope.REQUEST`, and `REQUEST` injection. Each is request-scoped, and so is everything that injects it, up to the controllers. A request-scoped logger near the bottom of the graph can construct a dozen objects per request.
- For each hit, name the chain it pulls up and whether it needs per-request instances or only per-request data.

**Fix:** per-request data through `AsyncLocalStorage` or `nestjs-cls`; per-tenant instances through durable providers.
