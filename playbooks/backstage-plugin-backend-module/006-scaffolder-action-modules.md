# Guardrail 006 — ScaffolderAction modules

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/006-scaffolder-action-modules.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Scaffolder actions are the building blocks of Backstage Software Templates.
They are registered as modules that extend the scaffolder via its
`ScaffolderExtensionPoint`. An action receives input from the template,
performs a task (create a repo, send a Slack message, provision a resource),
and optionally produces output that subsequent steps can consume.

Actions have three contracts that must be precise: the input schema
(what the template author provides), the output schema (what subsequent
steps receive), and the action ID (how the template references this action).
A mismatch in any of these three contracts produces a runtime error that
is visible to the template author as a cryptic validation failure.

The most common mistakes are: not validating input with Zod (trusting
template author input), not declaring output schema when outputs are
produced, using unstable action IDs that break existing templates, and
leaking sensitive config values into action outputs.

## Decision

Scaffolder actions are created with `createTemplateAction` from
`@backstage/plugin-scaffolder-node`. Input and output schemas are defined
with Zod. Action IDs follow the `<scope>:<verb>-<resource>` naming
convention. Actions are registered via `scaffolderExtensionPoint.addActions()`.
Sensitive values are never included in action outputs or logs.

## Rules

- MUST: Use `createTemplateAction` from `@backstage/plugin-scaffolder-node`
- MUST: Define input schema with Zod — `schema: { input: zodSchema }`
- MUST: Define output schema with Zod when the action produces outputs — `schema: { output: zodSchema }`
- MUST: Action ID follows `<scope>:<verb>-<resource>` convention — e.g. `myorg:create-repository`, `myorg:send-slack-message`
- MUST: Register via `scaffolderExtensionPoint.addActions(action)` — not `addAction` (singular, legacy)
- MUST: Access input via `ctx.input` — never via raw template variables
- MUST: Log progress via `ctx.logger` — not `console.log`
- MUST: Write workspace files via `ctx.workspacePath` — never via absolute paths
- MUST NOT: Include tokens, passwords, secrets, or API keys in action outputs
- MUST NOT: Include tokens, passwords, secrets, or API keys in `ctx.logger` calls
- MUST NOT: Use an unstable or random action ID — templates reference actions by ID, changing it breaks existing templates
- MUST NOT: Make the action ID match an existing Backstage built-in action — prefix with your org scope
- SHOULD: Use a static factory function that accepts config and services for action instantiation
- SHOULD: Validate that required config values exist at action instantiation time — not at execution time
- SHOULD: Use `ctx.output()` to declare outputs — not by returning values from the `handler`
- SHOULD NOT: Make network calls without timeout handling — a hung action blocks the entire scaffolder task

## Examples

### Correct

```typescript
// src/actions/createRepositoryAction.ts
import { createTemplateAction } from '@backstage/plugin-scaffolder-node';
import { z } from 'zod';
import {
  LoggerService,
  RootConfigService,
} from '@backstage/backend-plugin-api';

export function createCreateRepositoryAction(options: {
  config: RootConfigService;
  logger: LoggerService;
}) {
  const githubToken = options.config.getString('github.token');
  // validate config at instantiation — not at execution time

  return createTemplateAction({
    id: 'myorg:create-repository',           // stable ID with org scope

    schema: {
      input: z.object({
        name: z.string().min(1).max(100).describe('Repository name'),
        description: z.string().max(500).optional().describe('Repository description'),
        private: z.boolean().default(true).describe('Whether the repo is private'),
        owner: z.string().describe('GitHub organization or user'),
      }),
      output: z.object({
        repositoryUrl: z.string().url().describe('URL of the created repository'),
        cloneUrl: z.string().url().describe('Clone URL for the repository'),
      }),
    },

    async handler(ctx) {
      const { name, description, private: isPrivate, owner } = ctx.input;

      ctx.logger.info(`Creating repository ${owner}/${name}`);

      const response = await fetch('https://api.github.com/user/repos', {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${githubToken}`,  // token from config, not from input
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ name, description, private: isPrivate }),
      });

      if (!response.ok) {
        throw new Error(`GitHub API error: ${response.statusText}`);
      }

      const repo = await response.json();

      ctx.logger.info(`Repository created: ${repo.html_url}`);

      // declare outputs via ctx.output — never return them
      ctx.output('repositoryUrl', repo.html_url);
      ctx.output('cloneUrl', repo.clone_url);
    },
  });
}
```

```typescript
// src/module.ts
import { createBackendModule, coreServices } from '@backstage/backend-plugin-api';
import { scaffolderActionsExtensionPoint } from '@backstage/plugin-scaffolder-node/alpha';
import { createCreateRepositoryAction } from './actions/createRepositoryAction';

export const scaffolderModuleGithubActions = createBackendModule({
  pluginId: 'scaffolder',
  moduleId: 'github-actions',
  register(env) {
    env.registerInit({
      deps: {
        scaffolder: scaffolderActionsExtensionPoint,
        logger: coreServices.logger,
        config: coreServices.rootConfig,
      },
      async init({ scaffolder, logger, config }) {
        scaffolder.addActions(
          createCreateRepositoryAction({ config, logger }),
        );
      },
    });
  },
});
```

### Incorrect

```typescript
// ❌ no input schema — trusting raw template input
return createTemplateAction({
  id: 'myorg:create-repository',
  async handler(ctx) {
    const name = ctx.input.name;        // unvalidated — could be anything
    const token = ctx.input.githubToken; // ❌ secret in input — never do this
  },
});

// ❌ unstable action ID
return createTemplateAction({
  id: `myorg:create-repository-${Date.now()}`,  // random ID breaks templates
});

// ❌ secret in output
ctx.output('githubToken', githubToken);  // never output secrets

// ❌ secret in logger
ctx.logger.info(`Using token ${githubToken}`);  // never log secrets

// ❌ returning outputs instead of ctx.output
async handler(ctx) {
  return { repositoryUrl: repo.html_url };  // use ctx.output() instead
},

// ❌ console.log instead of ctx.logger
async handler(ctx) {
  console.log('Creating repository...');   // use ctx.logger.info()
},

// ❌ no timeout on network call
const response = await fetch('https://api.github.com/repos', {
  method: 'POST',
  // no timeout — can hang indefinitely and block the scaffolder
});
```

## Verify signal

Reject if:
- `createTemplateAction` is called without a `schema.input` Zod definition
- `ctx.input` contains any field that accepts a token, password, or secret — secrets come from config not from template input
- Action ID has no org scope prefix — e.g. `create-repository` instead of `myorg:create-repository`
- `ctx.output` is not used but the action handler `return`s an object — outputs must be declared via `ctx.output()`
- `console.log` appears in any action handler — use `ctx.logger`
- A token, password, API key, or secret value is passed to `ctx.output()` or `ctx.logger`

Flag for human review if:
- `schema.output` is missing but `ctx.output()` is called — output schema must match what is produced
- A network call has no timeout — verify the external system's reliability and response time
- The action ID matches or closely resembles a Backstage built-in action name — may cause confusion for template authors
- Config values are read inside the `handler` function instead of at instantiation time — validation happens too late