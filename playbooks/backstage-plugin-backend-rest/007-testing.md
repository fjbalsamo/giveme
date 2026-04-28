# Guardrail 007 — Testing

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/007-testing.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backend plugin testing has two distinct layers. The router layer verifies
that HTTP endpoints behave correctly — correct status codes, response
shapes, auth enforcement. The service layer verifies that business logic
is correct in isolation from HTTP and database.

The new backend system provides `startTestBackend` from
`@backstage/backend-test-utils` which boots a real backend instance
in memory for integration tests. This is more realistic than mocking
everything, but slower. For unit tests, `createRouter` is called directly
and tested with `supertest` — no backend instance needed.

The most common mistakes are: testing the router without mocking auth
(tests pass locally but fail in CI with real auth), using a real database
in unit tests (slow, fragile, environment-dependent), and not testing
the error paths (404, 401, 409) which are where most production bugs hide.

## Decision

Router tests use `supertest` directly against the output of `createRouter`,
with `httpAuth` mocked to return controlled credentials. Service tests
are pure unit tests with no HTTP or database context. Integration tests
that need a full backend use `startTestBackend` with an in-memory SQLite
database. Every route has tests for the happy path, the auth failure path,
and at least one error path.

## Rules

- MUST: Router tests use `supertest` against `createRouter` output directly
- MUST: Mock `httpAuth` in router tests — never use real auth in unit tests
- MUST: Service tests are pure — no HTTP, no database, no Backstage context
- MUST: Integration tests that need a database use SQLite via `TestDatabases` from `@backstage/backend-test-utils`
- MUST: Every route has a test for: happy path, unauthenticated request (401), and at least one error path
- MUST: Mock `auth.getPluginRequestToken` in tests that make outbound service calls
- MUST NOT: Connect to a real PostgreSQL database in any unit or integration test
- MUST NOT: Use real `httpAuth` or real `auth` service in router tests — mock them
- MUST NOT: Test migration files directly — test the database class against a migrated test database
- SHOULD: Use `mockCredentials` from `@backstage/backend-test-utils` to generate test credentials
- SHOULD: Test the `allow: ['user']` restriction on mutation routes — verify services are rejected
- SHOULD NOT: Use `jest.setTimeout` to work around slow tests — fix the test setup instead

## Examples

### Correct — router unit test

```typescript
// src/router.test.ts
import express from 'express';
import request from 'supertest';
import { createRouter } from './router';
import { mockCredentials, mockServices } from '@backstage/backend-test-utils';
import { MyToolServiceMock } from './service/MyToolServiceMock';

describe('createRouter', () => {
  let app: express.Express;
  let mockService: MyToolServiceMock;

  beforeEach(async () => {
    mockService = new MyToolServiceMock();

    const router = await createRouter({
      logger: mockServices.logger.mock(),
      config: mockServices.rootConfig.mock(),
      auth: mockServices.auth.mock(),
      httpAuth: mockServices.httpAuth.mock(),   // mocked — returns controlled credentials
      myToolService: mockService,
    });

    app = express().use(router);
  });

  describe('GET /items', () => {
    it('returns items for authenticated request', async () => {
      mockService.items = [{ id: '1', name: 'Item One' }];

      const response = await request(app)
        .get('/items')
        .set('Authorization', mockCredentials.user.header());  // mock user token

      expect(response.status).toBe(200);
      expect(response.body.items).toHaveLength(1);
      expect(response.body.items[0].name).toBe('Item One');
    });

    it('returns 401 for unauthenticated request', async () => {
      const response = await request(app).get('/items');  // no auth header

      expect(response.status).toBe(401);
    });
  });

  describe('POST /items', () => {
    it('creates an item for authenticated user', async () => {
      const response = await request(app)
        .post('/items')
        .set('Authorization', mockCredentials.user.header())
        .send({ name: 'New Item' });

      expect(response.status).toBe(201);
      expect(response.body.name).toBe('New Item');
    });

    it('rejects service-to-service calls on mutation', async () => {
      const response = await request(app)
        .post('/items')
        .set('Authorization', mockCredentials.service.header())  // service token
        .send({ name: 'New Item' });

      expect(response.status).toBe(403);              // allow: ['user'] rejects services
    });

    it('returns 400 for invalid payload', async () => {
      const response = await request(app)
        .post('/items')
        .set('Authorization', mockCredentials.user.header())
        .send({});                                     // missing required name

      expect(response.status).toBe(400);
    });
  });
});
```

### Correct — service unit test

```typescript
// src/service/MyToolService.test.ts — pure unit test
import { MyToolService } from './MyToolService';
import { MyToolDatabaseMock } from '../database/MyToolDatabaseMock';
import { mockServices } from '@backstage/backend-test-utils';

describe('MyToolService', () => {
  let service: MyToolService;
  let dbMock: MyToolDatabaseMock;

  beforeEach(() => {
    dbMock = new MyToolDatabaseMock();
    service = new MyToolService({
      logger: mockServices.logger.mock(),
      db: dbMock,
    });
  });

  it('returns all items', async () => {
    dbMock.items = [{ id: '1', name: 'Item One', created_at: new Date() }];
    const items = await service.getItems();
    expect(items).toHaveLength(1);
  });

  it('throws NotFoundError when item does not exist', async () => {
    dbMock.items = [];
    await expect(service.getItemById('nonexistent')).rejects.toThrow(
      'Item not found',
    );
  });
});
```

### Correct — integration test with SQLite

```typescript
// src/plugin.test.ts — integration test with real backend
import { startTestBackend } from '@backstage/backend-test-utils';
import { myToolPlugin } from './plugin';

describe('myToolPlugin integration', () => {
  it('starts and serves health endpoint', async () => {
    const { server } = await startTestBackend({
      features: [myToolPlugin],
    });

    const response = await request(server).get('/api/my-tool/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('ok');
  });
});
```

### Incorrect

```typescript
// ❌ real httpAuth in router test
const router = await createRouter({
  httpAuth: realHttpAuth,     // unpredictable in tests — use mockServices.httpAuth.mock()
  ...
});

// ❌ real PostgreSQL in test
beforeAll(async () => {
  db = knex({
    client: 'pg',
    connection: process.env.TEST_DATABASE_URL,  // use TestDatabases from backend-test-utils
  });
});

// ❌ no auth failure test
describe('GET /items', () => {
  it('returns items', async () => {     // only happy path — missing 401 test
    const response = await request(app)
      .get('/items')
      .set('Authorization', mockCredentials.user.header());
    expect(response.status).toBe(200);
  });
  // missing: unauthenticated test
});

// ❌ arbitrary timeout
jest.setTimeout(30000);                 // fix the test setup, not the timeout
```

## Verify signal

Reject if:
- A router test file does not import `mockServices` from `@backstage/backend-test-utils`
- A router test connects to a real database — knex config with `client: 'pg'` and a connection URL
- `jest.setTimeout` is called in any test file
- A route with `httpAuth.credentials(req, { allow: ['user'] })` has no test that sends a service credential and expects 403
- A test file imports and uses the real plugin services (`MyToolService`, `MyToolDatabase`) without mock equivalents

Flag for human review if:
- A route has no corresponding test file — all routes must have at least happy path + 401 tests
- `startTestBackend` integration tests don't include migration verification — confirm schema is applied correctly
- A service test has only happy path coverage — error paths are where production bugs hide
- Mock classes (`*Mock`) are not exported from `src/` — consumers cannot use them in their own tests