# Guardrail 004 — HTTP router

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/004-http-router.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A Backstage backend plugin serves its HTTP routes under
`/api/<plugin-id>/`. The new backend system provides `coreServices.httpRouter`
which mounts the plugin's Express router at that path automatically —
the plugin never has to know its own base URL.

Authentication in the new system is explicit and mandatory. Every route
either requires a credential check via `httpAuth.credentials()` or is
explicitly declared as unauthenticated via `httpRouter.addAuthPolicy`.
There is no implicit "trust everything" mode in production. A plugin that
doesn't handle auth will reject every request by default — including
requests from other Backstage services.

The most common mistakes are: mounting routes at absolute paths instead
of relative paths, forgetting to declare auth policies for health or
public endpoints, and mixing credential extraction with business logic
in the same handler.

## Decision

The plugin's HTTP surface lives in `src/router.ts` as a factory function
`createRouter` that accepts a typed `RouterOptions` object and returns
a Promise of an Express Router. Route handlers extract credentials via
`httpAuth.credentials()` and delegate business logic to a service layer.
Public endpoints are explicitly declared via `httpRouter.addAuthPolicy`.

## Rules

- MUST: HTTP routes live in `src/router.ts` as a `createRouter` function
- MUST: `createRouter` accepts a typed `RouterOptions` interface and returns `Promise<express.Router>`
- MUST: Use `express-promise-router` — not plain `express.Router()` — to handle async errors correctly
- MUST: Routes use relative paths — `/items` not `/api/my-tool/items`
- MUST: Every route that serves user requests calls `httpAuth.credentials(req)` to extract credentials
- MUST: Routes that must be public (health checks, webhooks) are declared via `httpRouter.addAuthPolicy` in `plugin.ts`
- MUST: `RouterOptions` interface is defined in `src/router.ts` and includes all injected services
- MUST NOT: Access `req.headers.authorization` directly — use `httpAuth.credentials(req)`
- MUST NOT: Hardcode the plugin base path `/api/<plugin-id>` — routes are always relative
- MUST NOT: Use `express.Router()` directly — use `express-promise-router` for async safety
- MUST NOT: Put database access or business logic directly in route handlers — delegate to service layer
- SHOULD: Return consistent error responses using `express` status codes and JSON bodies
- SHOULD: Log the request outcome at `debug` level — not every request at `info`
- SHOULD NOT: Catch errors in route handlers unless you need to transform them — let `express-promise-router` propagate to the global error handler

## Examples

### Correct

```typescript
// src/router.ts
import { errorHandler } from '@backstage/backend-defaults/alpha';
import { AuthService, HttpAuthService, LoggerService, RootConfigService } from '@backstage/backend-plugin-api';
import express from 'express';
import Router from 'express-promise-router';
import { MyToolService } from './service/MyToolService';

export interface RouterOptions {
  logger: LoggerService;
  config: RootConfigService;
  auth: AuthService;
  httpAuth: HttpAuthService;
  myToolService: MyToolService;
}

export async function createRouter(
  options: RouterOptions,
): Promise<express.Router> {
  const { logger, httpAuth, myToolService } = options;

  const router = Router();
  router.use(express.json());

  // GET /api/my-tool/items
  router.get('/items', async (req, res) => {
    // extract credentials — rejects if unauthenticated
    const credentials = await httpAuth.credentials(req);

    logger.debug(`Fetching items for ${credentials.principal.type}`);
    const items = await myToolService.getItems();
    res.json({ items });
  });

  // GET /api/my-tool/items/:id
  router.get('/items/:id', async (req, res) => {
    await httpAuth.credentials(req);
    const item = await myToolService.getItemById(req.params.id);
    if (!item) {
      res.status(404).json({ error: 'Item not found' });
      return;
    }
    res.json(item);
  });

  // POST /api/my-tool/items
  router.post('/items', async (req, res) => {
    const credentials = await httpAuth.credentials(req, {
      allow: ['user'],            // only real users, not service-to-service
    });
    const item = await myToolService.createItem(req.body, credentials);
    res.status(201).json(item);
  });

  // Health check — declared as unauthenticated in plugin.ts
  router.get('/health', (_, res) => {
    res.json({ status: 'ok' });
  });

  router.use(errorHandler());     // Backstage error handler last
  return router;
}
```

```typescript
// src/plugin.ts — auth policy for public endpoints
async init({ httpRouter, logger, config, database, auth, httpAuth }) {
  const myToolService = await MyToolService.create({ logger, config, database });

  httpRouter.use(
    await createRouter({ logger, config, auth, httpAuth, myToolService }),
  );

  // explicit declaration — health is unauthenticated
  httpRouter.addAuthPolicy({
    path: '/health',
    allow: 'unauthenticated',
  });
},
```

### Incorrect

```typescript
// ❌ plain express.Router — async errors not caught
const router = express.Router();
router.get('/items', async (req, res) => {
  const items = await myToolService.getItems();  // if this throws, server hangs
  res.json(items);
});

// ❌ absolute path
router.get('/api/my-tool/items', async (req, res) => {  // use /items
  ...
});

// ❌ reading auth header directly
router.get('/items', async (req, res) => {
  const token = req.headers.authorization?.split(' ')[1];  // use httpAuth.credentials()
  ...
});

// ❌ business logic in route handler
router.post('/items', async (req, res) => {
  const client = await database.getClient();          // move to service layer
  const [id] = await client('items').insert(req.body).returning('id');
  res.status(201).json({ id });
});

// ❌ no auth policy for public endpoint
router.get('/health', (_, res) => {
  res.json({ status: 'ok' });
  // missing: httpRouter.addAuthPolicy({ path: '/health', allow: 'unauthenticated' })
  // this will reject health check requests from the load balancer
});
```

## Verify signal

Reject if:
- `express.Router()` is used directly — must use `express-promise-router`
- Route paths start with `/api/` — must be relative
- `req.headers.authorization` is accessed directly in any route handler
- Database queries or business logic appear directly in route handler functions
- A route handler has no `httpAuth.credentials(req)` call and no corresponding `httpRouter.addAuthPolicy` entry in `plugin.ts`

Flag for human review if:
- `errorHandler()` from `@backstage/backend-defaults` is not the last middleware in the router
- A `POST`, `PUT`, or `DELETE` route does not restrict credentials to `allow: ['user']` — service-to-service mutations need explicit justification
- More than 8 routes are defined in a single `createRouter` — consider splitting into sub-routers by resource
- A route logs at `info` level on every request — use `debug` for request-level logging