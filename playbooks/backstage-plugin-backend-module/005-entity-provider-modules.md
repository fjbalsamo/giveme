# Guardrail 005 — EntityProvider modules

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/005-entity-provider-modules.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

An EntityProvider is the standard way to ingest entities into the Backstage
catalog from an external system — GitHub organizations, AWS resources, PagerDuty
services, or any custom data source. The provider connects to the external
system, fetches data, and emits entities that the catalog stores and indexes.

EntityProviders run on a schedule managed by `coreServices.scheduler`. The
catalog's mutation API (`EntityProviderConnection`) has two modes: full
replacement (`applyMutation` with `fullReplace`) and incremental updates
(`applyMutation` with `delta`). Choosing the wrong mutation mode for the
data volume causes either excessive catalog churn (fullReplace on large
datasets) or stale data accumulation (delta without periodic full sync).

The most common mistakes are: not implementing `getProviderName()` with a
stable unique name (causes duplicate entities on restart), not handling
the `connect()` lifecycle correctly, and running full replacement on every
tick instead of using delta mutations for incremental updates.

## Decision

EntityProvider modules implement the `EntityProvider` interface from
`@backstage/plugin-catalog-node`. They register via
`catalogProcessingExtensionPoint.addEntityProvider()`. Scheduling uses
`coreServices.scheduler` with a configurable interval. Provider names
are stable, unique, and derived from config to support multi-instance
deployments. Mutation mode is chosen based on data volume and freshness
requirements.

## Rules

- MUST: Implement `EntityProvider` interface from `@backstage/plugin-catalog-node`
- MUST: Implement `getProviderName()` returning a stable, unique string — e.g. `GithubOrgEntityProvider:my-org`
- MUST: Implement `connect(connection: EntityProviderConnection)` storing the connection for later use
- MUST: Register via `catalogProcessingExtensionPoint.addEntityProvider(provider)` — not `addProcessor`
- MUST: Schedule refresh via `coreServices.scheduler` — never use `setInterval` or `setTimeout`
- MUST: Make the schedule configurable via `coreServices.rootConfig`
- MUST: Use `applyMutation` with `type: 'full'` for small datasets or initial sync
- MUST: Use `applyMutation` with `type: 'delta'` for incremental updates on large datasets
- MUST: Wrap `applyMutation` calls in try/catch and log errors — a provider failure must not crash the catalog
- MUST NOT: Call `applyMutation` before `connect()` has been called — store connection and guard against null
- MUST NOT: Hardcode the provider name — include config-derived identifiers for multi-instance support
- MUST NOT: Use `setInterval` or `setTimeout` for scheduling — use `coreServices.scheduler`
- SHOULD: Use a static `fromConfig` factory method for provider instantiation
- SHOULD: Emit a `location` entity type for discovered resources when possible — not raw component entities
- SHOULD NOT: Fetch all data on every tick for large external systems — implement delta or pagination

## Examples

### Correct

```typescript
// src/providers/GithubOrgEntityProvider.ts
import {
  EntityProvider,
  EntityProviderConnection,
} from '@backstage/plugin-catalog-node';
import {
  LoggerService,
  RootConfigService,
  SchedulerService,
  SchedulerServiceTaskScheduleDefinition,
} from '@backstage/backend-plugin-api';

export class GithubOrgEntityProvider implements EntityProvider {
  private connection?: EntityProviderConnection;
  private readonly orgUrl: string;
  private readonly providerName: string;

  static fromConfig(
    config: RootConfigService,
    options: {
      logger: LoggerService;
      scheduler: SchedulerService;
      schedule?: SchedulerServiceTaskScheduleDefinition;
    },
  ): GithubOrgEntityProvider {
    const orgUrl = config.getString('githubOrg.url');
    const orgName = config.getString('githubOrg.name');

    const provider = new GithubOrgEntityProvider(orgUrl, orgName, options.logger);

    // schedule is configurable — operator controls frequency
    const schedule = options.schedule ?? {
      frequency: { minutes: 60 },
      timeout: { minutes: 10 },
    };

    options.scheduler.scheduleTask({
      id: `github-org-discovery-${orgName}`,
      frequency: schedule.frequency,
      timeout: schedule.timeout,
      fn: async () => {
        await provider.refresh();
      },
    });

    return provider;
  }

  constructor(
    orgUrl: string,
    private readonly orgName: string,
    private readonly logger: LoggerService,
  ) {
    this.orgUrl = orgUrl;
    // stable unique name — includes orgName for multi-instance support
    this.providerName = `GithubOrgEntityProvider:${orgName}`;
  }

  getProviderName(): string {
    return this.providerName;           // stable — same across restarts
  }

  async connect(connection: EntityProviderConnection): Promise<void> {
    this.connection = connection;
    await this.refresh();               // initial sync on connect
  }

  async refresh(): Promise<void> {
    if (!this.connection) {
      this.logger.warn(`${this.providerName}: refresh called before connect`);
      return;
    }

    try {
      this.logger.debug(`${this.providerName}: refreshing entities`);
      const entities = await this.fetchEntitiesFromGithub();

      await this.connection.applyMutation({
        type: 'full',                   // full replacement — small dataset
        entities: entities.map(entity => ({
          entity,
          locationKey: this.providerName,
        })),
      });

      this.logger.log(`${this.providerName}: emitted ${entities.length} entities`);
    } catch (error) {
      // never crash the catalog — log and continue
      this.logger.error(`${this.providerName}: refresh failed`, error);
    }
  }

  private async fetchEntitiesFromGithub() {
    // fetch from GitHub API — implementation omitted for brevity
    return [];
  }
}
```

```typescript
// src/module.ts
import { createBackendModule, coreServices } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { GithubOrgEntityProvider } from './providers/GithubOrgEntityProvider';

export const catalogModuleGithubOrgDiscovery = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'github-org-discovery',
  register(env) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, logger, config, scheduler }) {
        const provider = GithubOrgEntityProvider.fromConfig(config, {
          logger,
          scheduler,
        });
        catalog.addEntityProvider(provider);
      },
    });
  },
});
```

### Incorrect

```typescript
// ❌ hardcoded provider name — breaks multi-instance
getProviderName(): string {
  return 'GithubOrgEntityProvider';    // add org identifier for uniqueness
}

// ❌ setInterval instead of scheduler
connect(connection: EntityProviderConnection) {
  this.connection = connection;
  setInterval(() => this.refresh(), 60_000);  // use coreServices.scheduler
}

// ❌ applyMutation before connect
async refresh() {
  await this.connection!.applyMutation({ ... });  // connection may be undefined
}

// ❌ no error handling in refresh
async refresh() {
  const entities = await this.fetchEntities();
  await this.connection!.applyMutation({         // unhandled rejection crashes catalog
    type: 'full',
    entities,
  });
}

// ❌ registering as processor instead of provider
async init({ catalog }) {
  catalog.addProcessor(provider);               // should be addEntityProvider
}
```

## Verify signal

Reject if:
- The provider class does not explicitly `implements EntityProvider`
- `getProviderName()` returns a hardcoded string without any config-derived identifier
- `setInterval` or `setTimeout` appears in any provider file
- `applyMutation` is called without a null check on `this.connection`
- The `refresh()` method has no try/catch around `applyMutation`
- The module uses `catalog.addProcessor()` instead of `catalog.addEntityProvider()`

Flag for human review if:
- `applyMutation` uses `type: 'full'` and the external system has more than 1000 records — consider delta mutations
- The schedule interval is less than 5 minutes — verify the external system can handle the polling frequency
- `connect()` does not trigger an initial `refresh()` — entities won't appear until the first scheduled tick
- The provider name does not include a config-derived identifier — will conflict in multi-instance deployments