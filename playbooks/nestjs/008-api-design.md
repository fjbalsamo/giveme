# Guardrail 008 — API design

## Context

An API without versioning is a promise you can't keep. The moment you
need to change a contract — and you will — you either break existing
clients or you freeze the old behavior forever. Neither is acceptable.

Offset-based pagination sounds simple until your table has a million
rows and `OFFSET 990000` takes 30 seconds. Cursor-based pagination is
not a premature optimization — it's the only pagination strategy that
stays fast as data grows.

OpenAPI generated from code drifts. OpenAPI generated from Zod schemas
can't drift — the schema is the code. One source of truth means the
docs are always correct.

## Decision

Every endpoint lives under a version prefix. Pagination uses cursors,
not offsets. OpenAPI is generated automatically from Zod schemas via
`@nestjs/swagger` — never written by hand. Breaking changes get a new
version, not a flag or a workaround.

## Rules

- MUST: All endpoints versioned in the URL — `/v1/customers`, `/v2/customers`
- MUST: OpenAPI spec generated automatically via `@nestjs/swagger` integrated with `nestjs-zod`
- MUST: OpenAPI served at `/docs` — disabled when `NODE_ENV === 'production'`
- MUST: List endpoints use cursor-based pagination — response includes `nextCursor` and `hasMore`
- MUST: Every module has a `README.md` describing its purpose and HTTP contract
- MUST: Module `README.md` updated whenever the HTTP contract changes
- MUST NOT: Use offset-based pagination (`skip`/`take` with page numbers exposed to the client)
- MUST NOT: Write OpenAPI decorators by hand — schemas drive the spec
- MUST NOT: Introduce breaking changes in an existing version — bump the version prefix
- SHOULD: Return `nextCursor: null` and `hasMore: false` when there are no more results
- SHOULD: Accept cursor as a query parameter — `GET /v1/customers?cursor=<opaque-string>`
- SHOULD NOT: Expose internal database IDs or row numbers as cursor values — encode them

## Examples

### Correct

```typescript
// src/customers/customers.controller.ts
@Controller('v1/customers')                  // version in the URL prefix
export class CustomersController {

  @Get()
  async findAll(
    @Query('cursor') cursor?: string,
    @Query('limit') limit = 20,
  ): Promise<CustomerListResponseDto> {
    return this.customersService.findAll({ cursor, limit });
  }
}
```

```typescript
// src/customers/customers.schema.ts
export const CustomerListResponseSchema = z.object({
  data: z.array(CustomerResponseSchema),
  nextCursor: z.string().nullable(),        // opaque cursor — not a raw ID
  hasMore: z.boolean(),
});

export type CustomerListResponseDto = z.infer<typeof CustomerListResponseSchema>;
```

```typescript
// src/customers/customers.service.ts
async findAll({ cursor, limit }: { cursor?: string; limit: number }) {
  const decodedCursor = cursor ? decodeCursor(cursor) : undefined;

  const items = await this.repo.findMany({
    take: limit + 1,                        // fetch one extra to know if there's more
    cursor: decodedCursor ? { id: decodedCursor } : undefined,
    orderBy: { createdAt: 'desc' },
  });

  const hasMore = items.length > limit;
  const data = hasMore ? items.slice(0, limit) : items;
  const nextCursor = hasMore ? encodeCursor(data[data.length - 1].id) : null;

  return { data: data.map(toResponseDto), nextCursor, hasMore };
}
```

```typescript
// src/main.ts
if (process.env.NODE_ENV !== 'production') {
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document); // /docs only in non-production
}
```

### Incorrect

```typescript
// ❌ no version in URL
@Controller('customers')                    // use v1/customers

// ❌ offset pagination
@Get()
findAll(@Query('page') page = 1, @Query('limit') limit = 20) {
  return this.repo.findMany({
    skip: (page - 1) * limit,              // use cursor instead
    take: limit,
  });
}

// ❌ OpenAPI docs in production
const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document); // gate this behind NODE_ENV check

// ❌ hand-written OpenAPI decorator
@ApiProperty({ type: String, description: 'Customer name' })  // let Zod drive this
name: string;

// ❌ breaking change in existing version
@Controller('v1/customers')
async findAll() {
  // removed the `email` field from the response — this is a breaking change
  // create v2/customers instead
}
```

## Verify signal

Reject if:
- A `@Controller()` decorator has a path without a version prefix — no `v1/`, `v2/`, etc.
- A list endpoint returns a response with `page`, `pageNumber`, or `offset` fields
- A list endpoint uses `skip` and `take` with a page number exposed to the client
- `SwaggerModule.setup` is called without a `NODE_ENV !== 'production'` guard
- `@ApiProperty()` decorators appear in `src/` — Zod schemas drive OpenAPI, not manual decorators
- A module folder has no `README.md` file

Flag for human review if:
- A cursor value looks like a raw UUID or integer — cursors should be opaque encoded strings
- A `v1/` endpoint response schema differs from a previous version in a breaking way without a corresponding `v2/` controller
- A module `README.md` was not updated in the same commit that changed the HTTP contract