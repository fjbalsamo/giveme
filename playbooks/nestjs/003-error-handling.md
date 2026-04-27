# Guardrail 003 — Error handling and logging

## Context

Plain `throw new Error('something went wrong')` gives the client nothing
useful and gives the developer nothing traceable. Without a consistent
error format, every endpoint fails differently — different shapes,
different status codes, different levels of detail. Debugging becomes
archaeology.

`console.log` is equally dangerous in production: it bypasses log levels,
has no module context, and can silently leak sensitive data. When something
breaks at 3am, you want structured logs with trace IDs — not scattered
console output that disappears into the void.

## Decision

All errors thrown to the client use NestJS built-in exceptions. All error
responses follow RFC 7807 (`application/problem+json`) with a mandatory
`traceId` that matches the `x-request-id` propagated through logs. All
logging goes through NestJS `Logger` with module context. `console.*` is
banned everywhere in `src/`.

## Rules

- MUST: Use NestJS exceptions — `NotFoundException`, `BadRequestException`, `ConflictException`, `UnauthorizedException`, `ForbiddenException`, `UnprocessableEntityException`
- MUST: Error responses follow RFC 7807 with fields `type`, `title`, `status`, `detail`, `instance`, `timestamp`, `traceId`
- MUST: `traceId` in error responses equals the `x-request-id` from the request — same value, no exceptions
- MUST: Use `new Logger('ModuleName')` for all logging — instantiated with the class name as context
- MUST: A global `HttpExceptionFilter` converts all NestJS exceptions to RFC 7807 format
- MUST NOT: `throw new Error()` with a plain message for errors destined to the client
- MUST NOT: `console.log`, `console.error`, `console.warn`, `console.info`, `console.debug` anywhere in `src/`
- MUST NOT: Log tokens, passwords, secrets, API keys, or PII — use entity IDs to reference users in logs
- SHOULD: Use `logger.error` for unexpected failures, `logger.warn` for recoverable situations, `logger.log` for significant business events, `logger.debug` for diagnostics
- SHOULD NOT: Enable `logger.debug` in production — gate it behind `NODE_ENV`

## Examples

### Correct

```typescript
// src/customers/customers.service.ts
import { Injectable, NotFoundException, Logger } from '@nestjs/common';

@Injectable()
export class CustomersService {
  private readonly logger = new Logger(CustomersService.name);

  async findOne(id: string): Promise<CustomerResponseDto> {
    const customer = await this.repo.findById(id);

    if (!customer) {
      // NestJS exception — converted to RFC 7807 by the global filter
      throw new NotFoundException(`Customer ${id} not found`);
    }

    this.logger.log(`Customer retrieved — id: ${id}`);  // ID only, never PII
    return toResponseDto(customer);
  }

  async deactivate(id: string): Promise<void> {
    try {
      await this.repo.deactivate(id);
      this.logger.log(`Customer deactivated — id: ${id}`);
    } catch (error) {
      this.logger.error(`Failed to deactivate customer — id: ${id}`, error.stack);
      throw new UnprocessableEntityException('Could not deactivate customer');
    }
  }
}
```

```typescript
// src/common/filters/http-exception.filter.ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      type: `https://your-service.com/errors/${slugify(exception.message)}`,
      title: exception.message,
      status,
      detail: exception.getResponse(),
      instance: request.url,
      timestamp: new Date().toISOString(),
      traceId: request.headers['x-request-id'],  // same value as propagated in logs
    });
  }
}
```

### Incorrect

```typescript
// ❌ plain Error thrown to the client
async findOne(id: string) {
  const customer = await this.repo.findById(id);
  if (!customer) {
    throw new Error('not found');              // use NotFoundException
  }
}

// ❌ console anywhere in src/
async create(dto: CreateCustomerDto) {
  console.log('creating customer', dto);       // use this.logger.log — and never log dto directly, it may contain PII
  return this.repo.create(dto);
}

// ❌ logging PII
this.logger.log(`Customer email: ${customer.email}`);  // log ID only

// ❌ Logger without context
const logger = new Logger();                   // always pass the class name
logger.log('something happened');
```

## Verify signal

Reject if:
- `console.log`, `console.error`, `console.warn`, `console.info` or `console.debug` appears anywhere in `src/`
- `throw new Error(` appears in `src/` outside of `*.spec.ts` files
- `new Logger()` is instantiated without a context argument in `src/`
- A `*.filter.ts` response object is missing any of `type`, `title`, `status`, `detail`, `timestamp`, `traceId`
- `traceId` in an error response is not sourced from `x-request-id` header

Flag for human review if:
- A `catch` block logs `error.message` instead of `error.stack` — stack traces are needed for debugging
- A service method has more than 2 `try/catch` blocks — may indicate the error strategy needs rethinking
- Any log statement includes a field named `email`, `phone`, `password`, `token`, or `secret`