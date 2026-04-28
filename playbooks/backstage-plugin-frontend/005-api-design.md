# Guardrail 005 — API design

<!--
  Source: giveme community playbook (backstage-plugin-frontend/005-api-design.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage's API system is how plugins expose services to each other and
to the host app. An `ApiRef` is a typed token — it identifies a service
without coupling the consumer to the implementation. This is dependency
injection at the plugin boundary.

In the legacy system, APIs were registered in `createPlugin({ apis: [] })`
using `createApiFactory`. In the new system, APIs are `ApiBlueprint`
extensions — they live in the extensions array like any other UI contribution.

The most common mistake is exposing an API implementation directly instead
of through an `ApiRef`. This couples consumers to the concrete class,
makes testing impossible, and breaks the plugin's public contract the
moment the implementation changes.

## Decision

Every service a plugin exposes to other plugins or to the host app is
declared as an `ApiRef` using `createApiRef`. The interface is defined
separately from the implementation. The `ApiRef`, the interface type,
and an `ApiFactory` are exported from `src/index.ts`. The implementation
class is internal — but the factory function is public so that integrators
can register the API globally in the host app's `createApp({ apis: [] })`
when the default extension-based registration is not sufficient.
In the new system, the factory is also registered as an `ApiBlueprint` extension.

## Rules

- MUST: Every plugin API has a corresponding `ApiRef` created with `createApiRef`
- MUST: `ApiRef` is typed with an interface — never with a concrete class
- MUST: The interface, `ApiRef`, and `ApiFactory` are exported from `src/index.ts` — the implementation class is not
- MUST: The exported factory follows the naming convention `create<ApiName>ApiFactory` or `<apiName>ApiFactory`
- MUST: `ApiRef` id follows the pattern `plugin.<plugin-id>.<api-name>` — e.g. `plugin.my-tool.myToolApi`
- MUST: In the new system, APIs are registered as `ApiBlueprint` extensions in `createFrontendPlugin`
- MUST: API implementations use constructor injection for their dependencies — no direct `useApi()` calls inside implementations
- MUST NOT: Export the API implementation class from `src/index.ts` — export the factory instead
- MUST NOT: Register APIs in `createPlugin({ apis: [] })` for new plugins — use `ApiBlueprint`
- MUST NOT: Use `createApiFactory` directly in `createPlugin` — belongs in `ApiBlueprint.make()`
- SHOULD: Keep `ApiRef` and interface in `src/api/<ApiName>.ts`
- SHOULD: Keep implementation in `src/api/<ApiName>Impl.ts`
- SHOULD: Provide a mock implementation in `src/api/<ApiName>Mock.ts` for consumers to use in tests
- SHOULD NOT: Make `ApiRef` depend on Backstage-internal types that are not part of the public API

## Examples

### Correct

```typescript
// src/api/MyToolApi.ts — public contract
import { createApiRef } from '@backstage/frontend-plugin-api';

export interface MyToolApi {
  getItems(): Promise<MyToolItem[]>;
  getItemById(id: string): Promise<MyToolItem | undefined>;
}

// ApiRef id: plugin.<plugin-id>.<api-name>
export const myToolApiRef = createApiRef<MyToolApi>({
  id: 'plugin.my-tool.myToolApi',
});
```

```typescript
// src/api/MyToolApiImpl.ts — internal implementation
import { DiscoveryApi, FetchApi } from '@backstage/core-plugin-api';
import { MyToolApi, MyToolItem } from './MyToolApi';

export class MyToolApiImpl implements MyToolApi {
  constructor(
    private readonly discoveryApi: DiscoveryApi,  // injected, not imported
    private readonly fetchApi: FetchApi,
  ) {}

  async getItems(): Promise<MyToolItem[]> {
    const baseUrl = await this.discoveryApi.getBaseUrl('my-tool');
    const response = await this.fetchApi.fetch(`${baseUrl}/items`);
    return response.json();
  }

  async getItemById(id: string): Promise<MyToolItem | undefined> {
    const baseUrl = await this.discoveryApi.getBaseUrl('my-tool');
    const response = await this.fetchApi.fetch(`${baseUrl}/items/${id}`);
    if (response.status === 404) return undefined;
    return response.json();
  }
}
```

```typescript
// src/api/MyToolApiFactory.ts — exported so integrators can register globally
import { createApiFactory, discoveryApiRef, fetchApiRef } from '@backstage/core-plugin-api';
import { myToolApiRef } from './MyToolApi';
import { MyToolApiImpl } from './MyToolApiImpl';

// Exported factory — integrators use this in createApp({ apis: [myToolApiFactory] })
// when they need to register the API globally instead of relying on the plugin extension
export const myToolApiFactory = createApiFactory({
  api: myToolApiRef,
  deps: {
    discoveryApi: discoveryApiRef,
    fetchApi: fetchApiRef,
  },
  factory: ({ discoveryApi, fetchApi }) =>
    new MyToolApiImpl(discoveryApi, fetchApi),
});
```

```typescript
// src/api/MyToolApiMock.ts — for consumers to use in tests
import { MyToolApi, MyToolItem } from './MyToolApi';

export class MyToolApiMock implements MyToolApi {
  private items: MyToolItem[] = [];

  async getItems(): Promise<MyToolItem[]> {
    return this.items;
  }

  async getItemById(id: string): Promise<MyToolItem | undefined> {
    return this.items.find(item => item.id === id);
  }
}
```

```typescript
// src/extensions/MyToolApi.ts — new system registration
import { ApiBlueprint } from '@backstage/frontend-plugin-api';
import { discoveryApiRef, fetchApiRef } from '@backstage/core-plugin-api';
import { myToolApiRef } from '../api/MyToolApi';
import { MyToolApiImpl } from '../api/MyToolApiImpl';

export const myToolApi = ApiBlueprint.make({
  name: 'my-tool',
  params: defineParams => defineParams({
    api: myToolApiRef,
    deps: {
      discoveryApi: discoveryApiRef,
      fetchApi: fetchApiRef,
    },
    factory: ({ discoveryApi, fetchApi }) =>
      new MyToolApiImpl(discoveryApi, fetchApi),
  }),
});
```

```typescript
// src/index.ts — interface, ApiRef, and factory exported; implementation is not
export { myToolApiRef } from './api/MyToolApi';
export type { MyToolApi, MyToolItem } from './api/MyToolApi';
export { myToolApiFactory } from './api/MyToolApiFactory';  // for global registration
export { MyToolApiMock } from './api/MyToolApiMock';        // for consumer tests
// MyToolApiImpl is NOT exported — it is internal
```

### Incorrect

```typescript
// ❌ ApiRef typed with concrete class instead of interface
export const myToolApiRef = createApiRef<MyToolApiImpl>({  // use interface
  id: 'plugin.my-tool.myToolApi',
});

// ❌ implementation class exported from index.ts
export { MyToolApiImpl } from './api/MyToolApiImpl';  // export the factory instead

// ❌ ApiRef id doesn't follow convention
export const myToolApiRef = createApiRef<MyToolApi>({
  id: 'myToolApi',               // should be 'plugin.my-tool.myToolApi'
});

// ❌ legacy registration in createPlugin
export const myToolPlugin = createPlugin({
  id: 'my-tool',
  apis: [
    createApiFactory({            // belongs in ApiBlueprint for new system
      api: myToolApiRef,
      deps: {},
      factory: () => new MyToolApiImpl(),
    }),
  ],
});

// ❌ useApi inside implementation
export class MyToolApiImpl implements MyToolApi {
  async getItems() {
    const discoveryApi = useApi(discoveryApiRef);  // hooks don't work here
    ...
  }
}
```

## Verify signal

Reject if:
- `createApiRef` is typed with a class instead of an interface — pattern: `createApiRef<ClassName>` where `ClassName` ends in `Impl`
- `ApiRef` id does not follow `plugin.<plugin-id>.<api-name>` pattern
- An implementation class (`*Impl`) is exported from `src/index.ts` — export a factory function instead
- `createApiFactory` appears inside `createPlugin({ apis: [] })` in a new plugin
- `useApi` is called inside an API implementation class

Flag for human review if:
- No factory export exists in `src/index.ts` — integrators cannot register the API globally if needed
- No mock implementation exists in `src/api/` — consumers will struggle to test components that depend on this API
- The `ApiRef` interface has more than 10 methods — may be a sign the API is doing too much
- An implementation class takes more than 4 constructor arguments — may indicate too many responsibilities