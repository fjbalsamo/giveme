# Guardrail 006 — Security

## Context

Hardcoded secrets in source code are the most common cause of credential
leaks. They survive copy-paste, travel through git history, and end up in
places no one intended. A single committed API key can compromise an entire
infrastructure.

Open CORS, missing rate limits, and absent security headers are not
theoretical risks — they are the default configuration of every NestJS
project that didn't explicitly think about them. The default is insecure.
This guardrail makes the secure path the only path.

## Decision

Secrets come from environment variables, validated at startup with Zod.
The application fails fast if a required variable is missing — a silent
misconfiguration is worse than a crash. CORS, rate limiting, helmet, and
audit logging are not optional features — they are structural requirements
active from day one.

## Rules

- MUST: Load all secrets via `@nestjs/config` with a Zod-validated schema at startup
- MUST: Application throws and refuses to start if a required environment variable is missing
- MUST: CORS configured explicitly with an allowlist from environment variables — no wildcards
- MUST: Rate limiting via `@nestjs/throttler` on all public endpoints — minimum 100 req/min per IP
- MUST: `helmet` enabled with default configuration on the NestJS app
- MUST: Every mutation on a PII entity writes a record to `audit_logs` table with `actor_id`, `action`, `entity_type`, `entity_id`, `diff`, `request_id`, `timestamp`
- MUST: Sensitive fields in `audit_logs.diff` stored as `[REDACTED]` — never the actual value
- MUST NOT: Hardcode secrets, API keys, passwords, or tokens anywhere in `src/`
- MUST NOT: Commit `.env` files — `.env` must be in `.gitignore`
- MUST NOT: Use `*` as CORS origin in any environment
- MUST NOT: Delete or update records in `audit_logs` from application code — append only
- SHOULD: Every string field in the config schema has `.max(N)` — same rule as HTTP input
- SHOULD: Use `ALLOWED_ORIGINS` as the environment variable name for CORS allowlist

## Examples

### Correct

```typescript
// src/config/config.schema.ts
import { z } from 'zod';

export const ConfigSchema = z.object({
  DATABASE_URL: z.string().url().max(500),
  JWT_SECRET: z.string().min(32).max(256),
  ALLOWED_ORIGINS: z.string().max(1000),     // comma-separated allowlist
  THROTTLE_TTL: z.coerce.number().default(60),
  THROTTLE_LIMIT: z.coerce.number().default(100),
});

export type AppConfig = z.infer<typeof ConfigSchema>;
```

```typescript
// src/main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // validate config at startup — fail fast if anything is missing
  const config = app.get(ConfigService);
  const parsed = ConfigSchema.safeParse(process.env);
  if (!parsed.success) {
    console.error('Invalid environment configuration', parsed.error.format());
    process.exit(1);
  }

  // helmet — security headers
  app.use(helmet());

  // CORS — explicit allowlist, never wildcard
  app.enableCors({
    origin: config.get('ALLOWED_ORIGINS').split(','),
    methods: ['GET', 'POST', 'PATCH', 'PUT', 'DELETE'],
  });

  await app.listen(3000);
}
```

```typescript
// src/customers/customers.service.ts
async update(id: string, dto: UpdateCustomerDto, actor: string, requestId: string) {
  const before = await this.repo.findById(id);
  const updated = await this.repo.update(id, dto);

  // audit log — append only, sensitive fields redacted
  await this.auditRepo.create({
    actorId: actor,
    action: 'UPDATE',
    entityType: 'Customer',
    entityId: id,
    diff: {
      name: { before: before.name, after: updated.name },
      email: '[REDACTED]',                   // email is PII — always redact
    },
    requestId,
    timestamp: new Date(),
  });
}
```

### Incorrect

```typescript
// ❌ hardcoded secret
const jwtSecret = 'my-super-secret-key';     // use ConfigService

// ❌ wildcard CORS
app.enableCors({ origin: '*' });             // use explicit allowlist

// ❌ no helmet
const app = await NestFactory.create(AppModule);
await app.listen(3000);                      // missing app.use(helmet())

// ❌ PII in audit log diff
await this.auditRepo.create({
  diff: {
    email: { before: 'old@example.com', after: 'new@example.com' }, // redact this
    password: { before: 'abc123', after: 'xyz789' },                // never log passwords
  },
});

// ❌ deleting audit logs
await this.prisma.auditLog.delete({ where: { id } }); // audit_logs is append-only
```

## Verify signal

Reject if:
- Any string literal that looks like a secret appears in `src/` — patterns: `secret`, `password`, `api_key`, `token` assigned to a string literal
- `.env` is not listed in `.gitignore`
- `app.enableCors({ origin: '*' })` or `origin: true` appears in `src/`
- `helmet()` is not called in `main.ts`
- `ThrottlerModule` is not imported in `app.module.ts`
- A `*.service.ts` performs a mutation on a PII entity without a corresponding `auditRepo.create` call
- `this.prisma.auditLog.delete` or `this.prisma.auditLog.update` appears anywhere in `src/`
- A field named `password`, `token`, `secret`, or `apiKey` appears in an `audit_logs` diff without `[REDACTED]`

Flag for human review if:
- `ALLOWED_ORIGINS` is set to a single origin that looks like a development URL in a non-local environment
- A new entity is added that contains PII fields but has no audit log calls in its service
- `ThrottlerModule` is configured with a limit above 1000 req/min — may be too permissive