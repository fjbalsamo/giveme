# Guardrail 007 — Testing

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/007-testing.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Testing backend modules requires a different approach for each of the
three module types. EntityProvider tests need to verify that the provider
emits the right entities and calls `applyMutation` correctly. Scaffolder
action tests need to verify input validation, output declaration, and
error handling. ExtensionPoint modules need to verify that they register
correctly with the target plugin.

The new backend system's `startTestBackend` from
`@backstage/backend-test-utils` is the right tool for integration tests
— it boots a real backend with both the target plugin and the module
wired together. For unit tests, providers and actions are tested in
isolation without any backend context.

The most common mistake is testing the module registration itself instead
of the behavior it adds. A module test that only verifies "the module
registered without error" provides no value. Tests must verify the actual
capability the module contributes.

## Decision

Unit tests cover provider and action logic in isolation — no backend,
no real services. Integration tests use `startTestBackend` with the
target plugin and the module together. EntityProvider tests mock
`EntityProviderConnection`. Scaffolder action tests use
`createMockActionContext` from `@backstage/plugin-scaffolder-node/testUtils`.
All external HTTP calls are intercepted with `msw`.

## Rules

- MUST: Unit tests for providers and actions use no real backend services
- MUST: Mock `EntityProviderConnection` in provider unit tests
- MUST: Use `createMockActionContext` from `@backstage/plugin-scaffolder-node/testUtils` for action tests
- MUST: Integration tests use `startTestBackend` with both the target plugin and the module as features
- MUST: Intercept all external HTTP calls with `msw` — no real network in tests
- MUST: Test input validation — verify that invalid input throws with a meaningful error
- MUST: Test that secrets never appear in action outputs or logger calls
- MUST: Test error paths — provider refresh failure, action handler failure, network timeout
- MUST NOT: Use real `EntityProviderConnection` in unit tests
- MUST NOT: Make real network calls in any test
- MUST NOT: Test only the happy path — error paths are required for providers and actions
- SHOULD: Test that `getProviderName()` returns a stable unique name across different config values
- SHOULD: Test that `ctx.output()` is called with the correct keys and value types
- SHOULD NOT: Use `jest.setTimeout` to work around slow tests — fix the test setup

## Examples

### Correct — EntityProvider unit test

```typescript
// src/providers/GithubOrgEntityProvider.test.ts
import { GithubOrgEntityProvider } from './GithubOrgEntityProvider';
import { mockServices } from '@backstage/backend-test-utils';
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.get('https://api.github.com/orgs/my-org/members', (req, res, ctx) =>
    res(ctx.json([{ login: 'user1', name: 'User One' }]))
  ),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('GithubOrgEntityProvider', () => {
  const mockConnection = {
    applyMutation: jest.fn(),
    refresh: jest.fn(),
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('returns stable unique provider name', () => {
    const provider = new GithubOrgEntityProvider(
      'https://github.com',
      'my-org',
      mockServices.logger.mock(),
    );
    expect(provider.getProviderName()).toBe('GithubOrgEntityProvider:my-org');
  });

  it('different orgs produce different provider names', () => {
    const providerA = new GithubOrgEntityProvider('https://github.com', 'org-a', mockServices.logger.mock());
    const providerB = new GithubOrgEntityProvider('https://github.com', 'org-b', mockServices.logger.mock());
    expect(providerA.getProviderName()).not.toBe(providerB.getProviderName());
  });

  it('emits entities via applyMutation on refresh', async () => {
    const provider = new GithubOrgEntityProvider(
      'https://github.com',
      'my-org',
      mockServices.logger.mock(),
    );
    await provider.connect(mockConnection as any);

    expect(mockConnection.applyMutation).toHaveBeenCalledWith(
      expect.objectContaining({
        type: 'full',
        entities: expect.arrayContaining([
          expect.objectContaining({
            entity: expect.objectContaining({ kind: 'User' }),
          }),
        ]),
      }),
    );
  });

  it('does not crash catalog when refresh fails', async () => {
    server.use(
      rest.get('https://api.github.com/orgs/my-org/members', (req, res, ctx) =>
        res(ctx.status(500))
      ),
    );

    const provider = new GithubOrgEntityProvider(
      'https://github.com',
      'my-org',
      mockServices.logger.mock(),
    );
    await provider.connect(mockConnection as any);

    // refresh failed but connection.applyMutation was not called with garbage
    expect(mockConnection.applyMutation).not.toHaveBeenCalled();
  });
});
```

### Correct — ScaffolderAction unit test

```typescript
// src/actions/createRepositoryAction.test.ts
import { createMockActionContext } from '@backstage/plugin-scaffolder-node/testUtils';
import { createCreateRepositoryAction } from './createRepositoryAction';
import { mockServices } from '@backstage/backend-test-utils';
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.post('https://api.github.com/user/repos', (req, res, ctx) =>
    res(ctx.json({
      html_url: 'https://github.com/my-org/my-repo',
      clone_url: 'https://github.com/my-org/my-repo.git',
    }))
  ),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('myorg:create-repository', () => {
  const mockConfig = mockServices.rootConfig.mock({
    data: { github: { token: 'test-token' } },
  });

  const action = createCreateRepositoryAction({
    config: mockConfig,
    logger: mockServices.logger.mock(),
  });

  it('creates a repository and declares outputs', async () => {
    const ctx = createMockActionContext({
      input: {
        name: 'my-repo',
        owner: 'my-org',
        private: true,
      },
    });

    await action.handler(ctx);

    expect(ctx.output).toHaveBeenCalledWith(
      'repositoryUrl',
      'https://github.com/my-org/my-repo',
    );
    expect(ctx.output).toHaveBeenCalledWith(
      'cloneUrl',
      'https://github.com/my-org/my-repo.git',
    );
  });

  it('throws when GitHub API returns an error', async () => {
    server.use(
      rest.post('https://api.github.com/user/repos', (req, res, ctx) =>
        res(ctx.status(422), ctx.json({ message: 'Repository name already exists' }))
      ),
    );

    const ctx = createMockActionContext({
      input: { name: 'existing-repo', owner: 'my-org', private: true },
    });

    await expect(action.handler(ctx)).rejects.toThrow('GitHub API error');
  });

  it('never outputs the GitHub token', async () => {
    const ctx = createMockActionContext({
      input: { name: 'my-repo', owner: 'my-org', private: true },
    });

    await action.handler(ctx);

    const outputCalls = (ctx.output as jest.Mock).mock.calls;
    for (const [, value] of outputCalls) {
      expect(String(value)).not.toContain('test-token');
    }
  });

  it('throws on invalid input', async () => {
    const ctx = createMockActionContext({
      input: { name: '', owner: 'my-org', private: true }, // empty name
    });

    await expect(action.handler(ctx)).rejects.toThrow();
  });
});
```

### Correct — integration test

```typescript
// src/plugin.test.ts
import { startTestBackend } from '@backstage/backend-test-utils';
import { catalogPlugin } from '@backstage/plugin-catalog-backend';
import { catalogModuleGithubOrgDiscovery } from './module';

describe('catalogModuleGithubOrgDiscovery integration', () => {
  it('starts with catalog plugin without error', async () => {
    await expect(
      startTestBackend({
        features: [
          catalogPlugin,                       // target plugin
          catalogModuleGithubOrgDiscovery,     // module under test
        ],
      }),
    ).resolves.not.toThrow();
  });
});
```

### Incorrect

```typescript
// ❌ testing only registration — no behavioral test
it('registers without error', async () => {
  await startTestBackend({
    features: [catalogPlugin, catalogModuleGithubOrgDiscovery],
  });
  // test passes but verifies nothing meaningful
});

// ❌ real network in provider test
it('fetches entities', async () => {
  const provider = new GithubOrgEntityProvider(
    'https://api.github.com',     // real URL — test depends on network
    'my-org',
    logger,
  );
});

// ❌ no secret leak test for actions
it('creates repository', async () => {
  await action.handler(ctx);
  expect(ctx.output).toHaveBeenCalledWith('repositoryUrl', expect.any(String));
  // missing: verify token is not in any output
});
```

## Verify signal

Reject if:
- A provider test does not mock `EntityProviderConnection.applyMutation`
- A scaffolder action test does not use `createMockActionContext`
- Any test file makes real HTTP calls — `msw` must intercept all network requests
- A provider test has no test for the failure path — refresh failure must not throw
- A scaffolder action test has no test verifying secrets do not appear in outputs
- `jest.setTimeout` is called in any test file

Flag for human review if:
- The integration test with `startTestBackend` does not include the target plugin as a feature — module cannot register without its target
- A provider test does not verify `getProviderName()` uniqueness across different config values
- An action test has no invalid input test — input schema validation must be verified
- `msw` is not in `devDependencies` but the provider or action makes HTTP calls