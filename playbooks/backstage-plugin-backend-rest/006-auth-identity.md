# Guardrail 006 — Auth and identity

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/006-auth-identity.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

The new Backstage backend system replaced the legacy `IdentityService` and
`TokenManager` with three focused services: `httpAuth` for extracting
credentials from HTTP requests, `auth` for creating service tokens for
outbound calls, and `userInfo` for resolving user identity from credentials.

In the legacy system, plugins received an `identity` object that mixed
request parsing with token management. This made it easy to accidentally
use the wrong service for the wrong purpose — extracting a user token
for an outbound call, or creating a service token to identify the requester.

The new system enforces separation. A plugin that needs to know who is
making a request uses `httpAuth`. A plugin that needs to call another
Backstage service uses `auth` to mint a service token. A plugin that needs
the user's profile uses `userInfo`. These are not interchangeable.

## Decision

All credential extraction from incoming HTTP requests uses
`coreServices.httpAuth`. Outbound calls to other Backstage backend services
use tokens minted by `coreServices.auth`. User profile resolution uses
`coreServices.userInfo`. The deprecated `coreServices.identity` and
`coreServices.tokenManager` are never used in new plugins.

## Rules

- MUST: Use `coreServices.httpAuth` to extract credentials from incoming requests
- MUST: Use `coreServices.auth` to mint service tokens for outbound calls to other Backstage services
- MUST: Use `coreServices.userInfo` to resolve user identity from credentials when the plugin needs the user's profile
- MUST: Pass `credentials` from `httpAuth.credentials(req)` to `userInfo.getUserInfo(credentials)` — never construct credentials manually
- MUST: Restrict mutations to user credentials with `allow: ['user']` in `httpAuth.credentials(req, { allow: ['user'] })`
- MUST: Declare public endpoints explicitly via `httpRouter.addAuthPolicy` — never skip auth silently
- MUST NOT: Use `coreServices.identity` — deprecated, replaced by `httpAuth` + `userInfo`
- MUST NOT: Use `coreServices.tokenManager` — deprecated, replaced by `coreServices.auth`
- MUST NOT: Parse `Authorization` header manually — use `httpAuth.credentials(req)`
- MUST NOT: Forward user tokens to outbound service calls — mint a service token via `coreServices.auth`
- MUST NOT: Store credentials or tokens in memory between requests — credentials are per-request
- SHOULD: Log the principal type on authenticated actions — `credentials.principal.type`
- SHOULD: Use `coreServices.auth` with `getPluginRequestToken` for service-to-service calls
- SHOULD NOT: Require user credentials for endpoints called by other backend services — use `allow: ['service']` or omit the `allow` restriction

## Examples

### Correct — extracting user credentials

```typescript
// src/router.ts
router.get('/items', async (req, res) => {
  // extract and validate credentials from the request
  const credentials = await httpAuth.credentials(req);

  logger.debug(
    `GET /items — principal: ${credentials.principal.type}`,
  );

  const items = await myToolService.getItems();
  res.json({ items });
});

// mutation — restrict to real users only
router.post('/items', async (req, res) => {
  const credentials = await httpAuth.credentials(req, {
    allow: ['user'],                // rejects service-to-service calls
  });

  const user = await userInfo.getUserInfo(credentials);
  logger.log(`Item created by ${user.userEntityRef}`);

  const item = await myToolService.createItem(req.body, user.userEntityRef);
  res.status(201).json(item);
});
```

### Correct — outbound service-to-service call

```typescript
// src/service/MyToolService.ts
export class MyToolService {
  constructor(
    private readonly auth: AuthService,
    private readonly discoveryService: DiscoveryService,
    private readonly fetchApi: typeof fetch,
  ) {}

  async getEntityFromCatalog(entityRef: string): Promise<Entity> {
    // mint a service token — do NOT forward user tokens
    const { token } = await this.auth.getPluginRequestToken({
      onBehalfOf: await this.auth.getOwnServiceCredentials(),
      targetPluginId: 'catalog',
    });

    const baseUrl = await this.discoveryService.getBaseUrl('catalog');
    const response = await this.fetchApi(
      `${baseUrl}/entities/by-ref/${entityRef}`,
      {
        headers: { Authorization: `Bearer ${token}` },
      },
    );

    if (!response.ok) {
      throw new Error(`Catalog fetch failed: ${response.statusText}`);
    }
    return response.json();
  }
}
```

### Correct — mixed endpoint accepting both users and services

```typescript
// endpoint called by both users and other backend services
router.get('/status', async (req, res) => {
  // no allow restriction — accepts user and service credentials
  const credentials = await httpAuth.credentials(req);

  const status = await myToolService.getStatus();
  res.json({ status, callerType: credentials.principal.type });
});
```

### Incorrect

```typescript
// ❌ deprecated identity service
import { coreServices } from '@backstage/backend-plugin-api';

env.registerInit({
  deps: {
    identity: coreServices.identity,        // deprecated
    tokenManager: coreServices.tokenManager, // deprecated
  },
});

// ❌ manual Authorization header parsing
router.get('/items', async (req, res) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) {
    res.status(401).json({ error: 'Unauthorized' });
    return;
  }
  // use httpAuth.credentials(req) instead
});

// ❌ forwarding user token to outbound call
router.get('/entity/:ref', async (req, res) => {
  const token = req.headers.authorization;  // extracting raw token
  const baseUrl = await discoveryService.getBaseUrl('catalog');
  const response = await fetch(`${baseUrl}/entities/by-ref/${req.params.ref}`, {
    headers: { Authorization: token },      // forwarding user token — use auth.getPluginRequestToken()
  });
  res.json(await response.json());
});

// ❌ storing credentials between requests
let currentUser: BackstageCredentials;

router.post('/session', async (req, res) => {
  currentUser = await httpAuth.credentials(req);  // credentials are per-request only
  res.json({ ok: true });
});

// ❌ mutation without user restriction
router.delete('/items/:id', async (req, res) => {
  await httpAuth.credentials(req);          // missing allow: ['user']
  await myToolService.deleteItem(req.params.id);
  res.status(204).send();
});
```

## Verify signal

Reject if:
- `coreServices.identity` appears in any `deps` object
- `coreServices.tokenManager` appears in any `deps` object
- `req.headers.authorization` is accessed directly in any route handler
- A `POST`, `PUT`, `PATCH`, or `DELETE` route calls `httpAuth.credentials(req)` without `allow: ['user']`
- A service-to-service outbound call uses a token extracted from `httpAuth.credentials(req)` — must use `auth.getPluginRequestToken`
- Credentials or tokens are stored in module-level or class-level variables

Flag for human review if:
- A mutation route uses `allow: ['service']` — verify why a service is performing mutations on behalf of no user
- `userInfo.getUserInfo` is called on every request regardless of whether the user identity is actually needed — unnecessary overhead
- An endpoint that serves sensitive data has no `allow` restriction — verify that service-to-service access to this data is intentional
- `auth.getPluginRequestToken` `targetPluginId` is hardcoded — consider making it configurable for multi-instance setups