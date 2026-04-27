# Guardrail 007 — Observability

## Context

A service that can't tell you if it's healthy is a black box. Kubernetes
needs to know whether to send traffic to a pod or restart it — without
`/health/live` and `/health/ready`, it's guessing. A liveness probe that
passes while the database is unreachable means requests are routed to a
broken service.

Without a `x-request-id` propagated through every log line, debugging a
production incident means correlating timestamps and hoping. With it, you
type one ID and see the entire request lifecycle across every service that
touched it.

## Decision

Every service exposes two health endpoints and one metrics endpoint from
day one — not when someone asks for them. The `x-request-id` header is
generated if absent and propagated to every log line and every error
response. Prometheus metrics cover request count and duration at minimum.

## Rules

- MUST: Expose `/health/live` — liveness probe, returns 200 if the process is running
- MUST: Expose `/health/ready` — readiness probe, returns 200 only if Postgres responds
- MUST: Implement health checks with `@nestjs/terminus`
- MUST: Generate `x-request-id` if not present in the incoming request — use UUID v4
- MUST: Propagate `x-request-id` to every log line in the request lifecycle
- MUST: `x-request-id` value equals the `traceId` in error responses — same value, always
- MUST: Expose `/metrics` in Prometheus format — request count by status code, request duration histogram by endpoint
- MUST NOT: Expose `/metrics` when `NODE_ENV === 'production'` without authentication — metrics can leak internal structure
- SHOULD: Use `@nestjs/terminus` `PrismaHealthIndicator` or equivalent for the readiness DB check
- SHOULD: Return `/health/ready` as 503 — not 500 — when a dependency is unavailable
- SHOULD NOT: Include sensitive data in health check responses — no connection strings, no credentials

## Examples

### Correct

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, PrismaHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prisma: PrismaHealthIndicator,
  ) {}

  @Get('live')
  @HealthCheck()
  live() {
    // liveness — is the process alive?
    return { status: 'ok' };
  }

  @Get('ready')
  @HealthCheck()
  ready() {
    // readiness — can the service handle traffic?
    return this.health.check([
      () => this.prisma.pingCheck('database'),  // real DB check
    ]);
  }
}
```

```typescript
// src/common/middleware/request-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // generate if absent — propagate always
    req.headers['x-request-id'] = req.headers['x-request-id'] ?? uuidv4();
    res.setHeader('x-request-id', req.headers['x-request-id']);
    next();
  }
}
```

```typescript
// src/app.module.ts — apply middleware globally
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(RequestIdMiddleware).forRoutes('*');
  }
}
```

```typescript
// src/common/interceptors/metrics.interceptor.ts
@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    const req = context.switchToHttp().getRequest();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        const status = context.switchToHttp().getResponse().statusCode;

        // increment request counter by status code
        requestCounter.inc({ method: req.method, path: req.route?.path, status });

        // record duration histogram by endpoint
        requestDuration.observe({ method: req.method, path: req.route?.path }, duration / 1000);
      }),
    );
  }
}
```

### Incorrect

```typescript
// ❌ liveness and readiness merged into one endpoint
@Get('health')
health() {
  return { status: 'ok' };               // split into /health/live and /health/ready
}

// ❌ readiness without a real DB check
@Get('ready')
ready() {
  return { status: 'ok' };               // must check Postgres — this always passes
}

// ❌ no x-request-id generation
@Get(':id')
findOne(@Param('id') id: string) {
  this.logger.log(`Finding customer ${id}`);  // no traceId in the log — untraceable
  return this.customersService.findOne(id);
}

// ❌ metrics without auth in production
if (process.env.NODE_ENV === 'production') {
  app.use('/metrics', metricsHandler);       // no authentication — exposes internals
}
```

## Verify signal

Reject if:
- No `GET /health/live` endpoint exists in the codebase
- No `GET /health/ready` endpoint exists in the codebase
- `/health/ready` does not call a Prisma or DB health check — returns static response only
- `@nestjs/terminus` is not in `package.json` dependencies
- `x-request-id` is not generated and propagated in a middleware applied globally
- `/metrics` endpoint is exposed in production without authentication middleware

Flag for human review if:
- `/health/live` and `/health/ready` are the same handler — they must have different logic
- A log line in a service does not include the `x-request-id` value from the request context
- The metrics endpoint exposes labels with high cardinality — e.g. full URLs with IDs instead of route patterns