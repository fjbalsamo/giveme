# Guardrail 006 — Testing

<!--
  Source: giveme community playbook (backstage-plugin-frontend/006-testing.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Testing Backstage plugins requires rendering components inside a Backstage
app context — with APIs, routing, themes, and the extension tree all wired
up. A plain React test with `render()` from `@testing-library/react` will
fail or produce misleading results because the component expects to find
`useApi()`, `useRouteRef()`, and other Backstage hooks in context.

The new frontend system also changed how extensions are tested. The legacy
`renderInTestApp` still works for component-level tests, but extension-level
tests now use `createExtensionTester` from `@backstage/frontend-test-utils`
which instantiates only the extension tree needed for the test — not the
entire app.

## Decision

Component tests use `renderInTestApp` from `@backstage/test-utils` with
mocked APIs registered via `TestApiProvider`. Extension tests use
`createExtensionTester` from `@backstage/frontend-test-utils`. API
implementations are tested in isolation without any React context.
The exported `*Mock` class is used by consumers — never the real
implementation — in component tests.

## Rules

- MUST: Component tests use `renderInTestApp` from `@backstage/test-utils` — never plain `render` from `@testing-library/react`
- MUST: Mock APIs using `TestApiProvider` from `@backstage/test-utils` wrapping the component under test
- MUST: Use the plugin's exported `*Mock` class as the API mock in component tests — not `jest.fn()` stubs on the real implementation
- MUST: Extension tests use `createExtensionTester` from `@backstage/frontend-test-utils`
- MUST: API implementation unit tests are pure — no React context, no `renderInTestApp`
- MUST: Every component that calls `useApi()` has a corresponding test with the API mocked
- MUST NOT: Use `jest.mock('@backstage/core-plugin-api')` to mock the entire Backstage API module
- MUST NOT: Call real network endpoints in unit tests — mock `fetchApi` or use `msw`
- SHOULD: Use `msw` for integration-level tests that need realistic API response shapes
- SHOULD: Test the loading state, error state, and success state for every component that fetches data
- SHOULD NOT: Use `waitFor` with arbitrary timeouts — use `findBy*` queries which wait automatically

## Examples

### Correct — component test

```typescript
// src/components/MyToolPage/MyToolPage.test.tsx
import React from 'react';
import { renderInTestApp, TestApiProvider } from '@backstage/test-utils';
import { myToolApiRef } from '../../api/MyToolApi';
import { MyToolApiMock } from '../../api/MyToolApiMock';
import { MyToolPage } from './MyToolPage';

describe('MyToolPage', () => {
  it('renders items from the API', async () => {
    const mockApi = new MyToolApiMock();
    mockApi.items = [{ id: '1', name: 'Item One' }];

    const { getByText } = await renderInTestApp(
      <TestApiProvider apis={[[myToolApiRef, mockApi]]}>
        <MyToolPage />
      </TestApiProvider>,
    );

    expect(getByText('Item One')).toBeInTheDocument();
  });

  it('shows an error when the API fails', async () => {
    const mockApi = new MyToolApiMock();
    mockApi.getItems = jest.fn().mockRejectedValue(new Error('API error'));

    const { getByText } = await renderInTestApp(
      <TestApiProvider apis={[[myToolApiRef, mockApi]]}>
        <MyToolPage />
      </TestApiProvider>,
    );

    expect(getByText(/something went wrong/i)).toBeInTheDocument();
  });
});
```

### Correct — extension test

```typescript
// src/extensions/MyToolPage.test.tsx
import { createExtensionTester } from '@backstage/frontend-test-utils';
import { myToolPage } from './MyToolPage';
import { myToolApiRef } from '../api/MyToolApi';
import { MyToolApiMock } from '../api/MyToolApiMock';

describe('myToolPage extension', () => {
  it('renders without crashing', async () => {
    const tester = createExtensionTester(myToolPage)
      .add(myToolApiRef, new MyToolApiMock());

    const { getByRole } = await renderInTestApp(tester.reactElement());

    expect(getByRole('main')).toBeInTheDocument();
  });
});
```

### Correct — API implementation unit test

```typescript
// src/api/MyToolApiImpl.test.ts — pure unit test, no React
import { MyToolApiImpl } from './MyToolApiImpl';
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.get('http://localhost/api/my-tool/items', (req, res, ctx) =>
    res(ctx.json([{ id: '1', name: 'Item One' }]))
  ),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('MyToolApiImpl', () => {
  const mockDiscoveryApi = {
    getBaseUrl: jest.fn().mockResolvedValue('http://localhost/api/my-tool'),
  };
  const mockFetchApi = { fetch: jest.fn().mockImplementation(fetch) };

  it('returns items from the backend', async () => {
    const api = new MyToolApiImpl(mockDiscoveryApi, mockFetchApi);
    const items = await api.getItems();
    expect(items).toHaveLength(1);
    expect(items[0].name).toBe('Item One');
  });
});
```

### Incorrect

```typescript
// ❌ plain render without Backstage context
import { render } from '@testing-library/react';

const { getByText } = render(<MyToolPage />);  // useApi() will throw

// ❌ mocking the entire Backstage module
jest.mock('@backstage/core-plugin-api', () => ({
  useApi: jest.fn().mockReturnValue({ getItems: jest.fn() }),
}));

// ❌ using real implementation as mock
const realApi = new MyToolApiImpl(discoveryApi, fetchApi);
// use MyToolApiMock instead

// ❌ arbitrary timeout in waitFor
await waitFor(() => expect(getByText('Item One')).toBeInTheDocument(), {
  timeout: 5000,   // use findByText('Item One') instead
});
```

## Verify signal

Reject if:
- `render` from `@testing-library/react` is used directly in a component test instead of `renderInTestApp`
- `jest.mock('@backstage/core-plugin-api')` or `jest.mock('@backstage/frontend-plugin-api')` appears in any test file
- A component test mocks an API with `jest.fn()` stubs directly instead of using the exported `*Mock` class
- `waitFor` is used with an explicit `timeout` option — use `findBy*` queries instead

Flag for human review if:
- A component that calls `useApi()` has no corresponding test file
- An extension has no extension-level test using `createExtensionTester`
- `msw` is not in `devDependencies` but the API implementation makes HTTP calls
- Test files import directly from `src/` paths of other packages instead of the package public API