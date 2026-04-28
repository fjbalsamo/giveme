# Guardrail 005 — Database access

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/005-database-access.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage provides a managed database service via `coreServices.database`
that gives each plugin an isolated Knex client scoped to its own database
or schema. Plugins never manage connection pools, credentials, or database
names directly — the backend operator configures these centrally in
`app-config.yaml`.

Migrations are the contract between the plugin and its database schema.
A plugin that ships without migrations forces operators to run raw SQL
manually. A plugin that runs migrations unconditionally on every startup
blocks startup if the database is unavailable. A plugin whose migration
files are not included in the published package fails on first install.

The three cases — no database, database without migrations, database with
migrations — have distinct patterns. Using the wrong pattern for the
situation creates operational burden for every team that installs the plugin.

## Decision

Plugins that need persistence use `coreServices.database` to get a Knex
client. All database access is encapsulated in a dedicated database class
in `src/database/`. Migrations live in a `migrations/` folder at the
package root and run via `client.migrate.latest()` on startup, respecting
the `database.migrations.skip` flag. Plugins that don't need a database
do not declare `coreServices.database` in their deps.

## Rules

- MUST: Declare `coreServices.database` in `deps` only if the plugin actually persists data
- MUST: All database access is encapsulated in a class in `src/database/<PluginName>Database.ts`
- MUST: Database class is instantiated with a Knex client — never calls `database.getClient()` internally
- MUST: Migrations live in `migrations/` at the package root — not in `src/`
- MUST: Migration files follow knex naming convention — timestamp prefix, descriptive name: `20240101000000_init.js`
- MUST: Migrations run via `client.migrate.latest({ directory: migrationsDir })` using `resolvePackagePath`
- MUST: Respect `database.migrations?.skip` flag before running migrations
- MUST: `migrations/` is included in `package.json` `files` array
- MUST NOT: Access database credentials or connection strings directly — use `coreServices.database`
- MUST NOT: Share a Knex client between plugins — each plugin gets its own isolated client
- MUST NOT: Run raw SQL strings with `client.raw()` for schema changes — use migration files
- MUST NOT: Call `client.migrate.latest()` without checking `database.migrations?.skip`
- SHOULD: Wrap all database operations in a typed interface defined in `src/database/`
- SHOULD: Use transactions for operations that modify multiple tables
- SHOULD NOT: Put knex query logic directly in service classes — delegate to the database class

## Examples

### Correct — no database

```typescript
// src/plugin.ts — no database dep
env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,
    logger: coreServices.logger,
    config: coreServices.rootConfig,
    // no coreServices.database — this plugin doesn't persist data
  },
  async init({ httpRouter, logger, config }) {
    httpRouter.use(await createRouter({ logger, config }));
  },
});
```

### Correct — database with migrations

```typescript
// src/plugin.ts
import { resolvePackagePath } from '@backstage/backend-plugin-api';

env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,
    logger: coreServices.logger,
    database: coreServices.database,
    auth: coreServices.auth,
    httpAuth: coreServices.httpAuth,
  },
  async init({ httpRouter, logger, database, auth, httpAuth }) {
    const client = await database.getClient();

    // run migrations — respect skip flag for operators who manage schema externally
    const migrationsDir = resolvePackagePath(
      '@scope/backstage-plugin-my-tool-backend',
      'migrations',
    );
    if (!database.migrations?.skip) {
      await client.migrate.latest({ directory: migrationsDir });
    }

    const db = new MyToolDatabase(client);
    const myToolService = new MyToolService({ logger, db });
    httpRouter.use(await createRouter({ logger, auth, httpAuth, myToolService }));
  },
});
```

```typescript
// src/database/MyToolDatabase.ts
import { Knex } from 'knex';

export interface MyToolItem {
  id: string;
  name: string;
  created_at: Date;
}

export class MyToolDatabase {
  constructor(private readonly db: Knex) {}

  async getItems(): Promise<MyToolItem[]> {
    return this.db<MyToolItem>('my_tool_items').select('*');
  }

  async getItemById(id: string): Promise<MyToolItem | undefined> {
    return this.db<MyToolItem>('my_tool_items').where({ id }).first();
  }

  async createItem(data: Omit<MyToolItem, 'id' | 'created_at'>): Promise<MyToolItem> {
    const [item] = await this.db<MyToolItem>('my_tool_items')
      .insert({ ...data, id: generateUuid(), created_at: new Date() })
      .returning('*');
    return item;
  }

  async deleteItem(id: string): Promise<void> {
    await this.db<MyToolItem>('my_tool_items').where({ id }).delete();
  }
}
```

```javascript
// migrations/20240101000000_init.js
/**
 * @param {import('knex').Knex} knex
 */
exports.up = async function up(knex) {
  await knex.schema.createTable('my_tool_items', table => {
    table.uuid('id').primary();
    table.string('name').notNullable();
    table.timestamp('created_at').defaultTo(knex.fn.now());
  });
};

/**
 * @param {import('knex').Knex} knex
 */
exports.down = async function down(knex) {
  await knex.schema.dropTable('my_tool_items');
};
```

### Incorrect

```typescript
// ❌ database access directly in service class
export class MyToolService {
  constructor(private readonly database: DatabaseService) {}

  async getItems() {
    const client = await this.database.getClient();  // getClient() belongs in plugin.ts init
    return client('my_tool_items').select('*');
  }
}

// ❌ migration without skip check
async init({ database }) {
  const client = await database.getClient();
  await client.migrate.latest({ directory: migrationsDir }); // missing skip check
}

// ❌ raw SQL for schema changes
async init({ database }) {
  const client = await database.getClient();
  await client.raw(`
    CREATE TABLE IF NOT EXISTS my_tool_items (
      id UUID PRIMARY KEY
    )
  `);  // use migration files instead
}

// ❌ knex query logic in service class
export class MyToolService {
  async getItems() {
    return this.db('my_tool_items')    // this belongs in MyToolDatabase
      .where({ active: true })
      .orderBy('created_at', 'desc')
      .select('*');
  }
}
```

```javascript
// ❌ migration without down function
exports.up = async function(knex) {
  await knex.schema.createTable('my_tool_items', ...);
};
// missing exports.down — migration is not reversible
```

## Verify signal

Reject if:
- `coreServices.database` is in `deps` but the plugin has no `migrations/` folder and no database class in `src/database/`
- `client.migrate.latest()` is called without checking `database.migrations?.skip`
- `client.raw()` is used for schema changes — table creation, column addition, index creation
- A migration file is missing `exports.down` — all migrations must be reversible
- `resolvePackagePath` is not used to locate the migrations directory — hardcoded paths break when the package is installed
- Database query logic (`.where()`, `.select()`, `.insert()`, `.update()`, `.delete()`) appears in `src/service/` files

Flag for human review if:
- A migration file modifies an existing table without a corresponding down migration that reverses the change
- `migrations/` is not in `package.json` `files` array but `coreServices.database` is used — migrations won't be published
- A database class method performs more than one table join — consider whether a dedicated query method is cleaner
- No transaction is used for a service operation that writes to more than one table