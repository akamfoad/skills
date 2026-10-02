# Edge checks

Checks 5, 6, 7, 8, 9. Evidence lives in guards, controllers, DTOs, pipes, and interceptors. Apply [`../VERDICTS.md`](../VERDICTS.md).

## 5. Authentication is deny-by-default

`module` · critical

**Property:** every route requires an authenticated caller unless it carries an explicit, reviewable opt-out.

**Evidence:**
- The auth guard is global: `{ provide: APP_GUARD, useClass: ... }`, or `@nestjs/authentication`'s `AuthenticationModule` (registers a global guard, 401 for anonymous callers). Per-controller `@UseGuards(AuthGuard)` alone **fails open**: the one controller without the decorator is public, and nothing reports it.
- The guard reads opt-out metadata with `reflector.getAllAndOverride(IS_PUBLIC, [context.getHandler(), context.getClass()])`, so handler and class both work.
- List every `@Public()` (or the house equivalent) in scope. Each one must be a route that is public by design: sign-in, sign-up, password reset, health, webhooks with their own signature check.
- In scope, list every route and its effective auth. A route with neither a role requirement nor an opt-out is a FAIL when the global guard only enforces roles that a decorator declares.

**Fix:** register the guard with `APP_GUARD`; opt routes out with metadata. A review then checks one thing: each `@Public()` in the diff.

## 6. Security headers, CORS, rate limits, trusted proxy

`app` · high

**Property:** browsers get standard security headers, only known origins get CORS, brute-forceable routes are rate-limited per real client IP, and `req.ip` / `req.protocol` reflect the client.

**Evidence:**
- Security headers: Nest 12 `app.useSecurityHeaders()` (Helmet 8 defaults: CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`), called once and before `app.init()`/`app.listen()` (else it throws). Older Nest: `app.use(helmet())`.
- CORS: `app.enableCors()` with no argument allows every origin, a FAIL for a browser-facing API. With `credentials: true`, `origin` lists exact origins.
- Rate limits: `ThrottlerModule.forRoot([...])` plus `{ provide: APP_GUARD, useClass: ThrottlerGuard }`, and a tighter `@Throttle(...)` on login, password reset, and anything that sends email or SMS. Counters are per process unless storage is shared: N replicas give an N times higher effective limit.
- Trusted proxy: behind a load balancer, Express needs `app.set('trust proxy', <hops>)`; Fastify needs `trustProxy` on the adapter. Without it every client shares the balancer's IP (one rate-limit bucket, wrong audit logs) and `req.protocol` is `http` (secure cookies break). The value is the real hop count. `true` lets any client choose its IP through `X-Forwarded-For`: FAIL.

**Fix:** headers on, explicit CORS origins, throttler with shared storage, exact proxy hop count.

## 7. Every input is validated

`module` · critical; high for unbounded list or bulk endpoints

**Property:** a handler receives only declared, typed, bounded input.

**Evidence:**
- A global validation pipe exists. class-validator: `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`. `whitelist` is what stops `{ "email": "...", "role": "admin" }` from reaching `repository.save(dto)`. Standard Schema (Nest 12): `StandardSchemaValidationPipe`, with the schema on the parameter (`@Body({ schema })`); a Zod `z.object()` strips unknown keys, `z.strictObject()` rejects them.
- The pipe reaches e2e tests too: `useGlobalPipes()` binds to one app instance, so prefer `{ provide: APP_PIPE, ... }` or repeat the call in test setup.
- Every `@Body`, `@Query`, `@Param` in scope has a DTO or schema. A raw `@Body() body: any` or a plain interface type is a FAIL (interfaces vanish at runtime, so nothing validates).
- Every list endpoint has a `limit`/`take` with a maximum and a default (`@Max(100)` plus a default, or `z.coerce.number().int().min(1).max(100).default(20)`). Array inputs (bulk ids) have a maximum size.
- Body size limit unchanged globally (Express JSON default 100 KB). A global `app.useBodyParser('json', { limit: '50mb' })` is a FAIL: parsing a large body blocks the event loop for every request. Large payloads stream on their own route or upload to object storage.

**Fix:** global pipe with whitelist, DTO or schema on every input, caps and defaults on every list.

## 8. Output is controlled

`module` · critical

**Property:** a response contains only fields chosen for it, and the API map is private in production.

**Evidence:**
- No controller returns an ORM entity or a raw query row. Each returns a response DTO, or a class instance with `@Exclude()` under a global `ClassSerializerInterceptor`. The interceptor skips plain objects: raw query results and spreads (`{ ...user, extra }`) pass through as-is.
- `SwaggerModule.setup()` is skipped when `NODE_ENV === 'production'`, or sits behind auth.

**Fix:** map to response DTOs at the controller boundary; gate Swagger on environment.

## 9. Writes are safe to repeat

`module` · high; critical for money movement

**Property:** a client retry of a non-idempotent write (double click, flaky network, SDK retry) runs the operation once.

**Evidence:**
- List every `POST`/`PATCH` in scope with an external side effect (charge, transfer, order, email, SMS). Each needs dedupe by an `Idempotency-Key` header, or a natural unique key in the database.
- `@nestjs/idempotency`: `IdempotencyModule.forRoot({ scope: (req) => req.user?.id, ... })` and `@Idempotent({ required: true })` on the route. Behaviour: a repeat replays the stored response with `Idempotent-Replayed: true`; a duplicate in flight gets 409 with `Retry-After`; the same key with a different body gets 422; 5xx responses are not stored; `required: true` rejects requests without a key.
- The idempotency store is shared (Postgres or Redis). An in-memory store reruns the handler when the retry lands on another replica or after a deploy. Default `ttl` is 24 h, and the whole response body is stored.
- With `ResilienceModule` too: `IdempotencyModule` comes first in `imports` (import order sets global interceptor order).

**Fix:** required idempotency key with a shared store on every side-effect write. Without `@nestjs/idempotency`: an interceptor keyed by user + key in a table with a unique constraint.
