# Guardrail 004 — ExtensionPoint modules

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/004-extension-point-modules.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

An ExtensionPoint is a typed contract that a plugin publishes to allow
modules to add capabilities without modifying the plugin itself. The
plugin owns the ExtensionPoint definition — it publishes it via its
`-node` package. Modules consume it via `deps` and call its methods
inside `registerInit`.

The critical discipline is understanding what an ExtensionPoint can and
cannot do. An ExtensionPoint is not a backdoor into the plugin's internals
— it is a deliberate, versioned API surface. If the ExtensionPoint doesn't
expose what the module needs, the correct answer is to contribute a new
method to the ExtensionPoint, not to work around it by importing plugin
internals.

A module that imports directly from the plugin's `-backend` package to
access non-ExtensionPoint APIs creates a tight coupling that breaks on
every minor version bump of the target plugin.

## Decision

Modules interact with the target plugin exclusively via its published
ExtensionPoint from the `-node` package. The module calls only the
methods documented in the ExtensionPoint's interface. If the required
capability is not available on the ExtensionPoint, the module contributor
opens an issue or PR on the target plugin rather than bypassing the
ExtensionPoint boundary.

## Rules

- MUST: Access the target plugin only via its published ExtensionPoint ref from the `-node` package
- MUST: Call only methods defined on the ExtensionPoint interface — no internal plugin APIs
- MUST: Verify that the ExtensionPoint method being called exists in the target plugin version declared in `package.json`
- MUST: The class or function registered via the ExtensionPoint implements the interface the plugin expects
- MUST: Read the target plugin's `-node` package TypeScript types to understand the exact interface contract
- MUST NOT: Import from the target plugin's `-backend` package in module source files
- MUST NOT: Cast or coerce the ExtensionPoint type to access undocumented methods
- MUST NOT: Use `(extensionPoint as any)` or type assertions to bypass the ExtensionPoint interface
- SHOULD: Implement the registered class as a separate file in `src/providers/` or `src/actions/`
- SHOULD: Use a static factory method (`fromConfig`) on provider/action classes for clean instantiation
- SHOULD NOT: Register more than one provider or action per module — create separate modules for separate capabilities

## Examples

### Correct

```typescript
// src/module.ts
import { createBackendModule, coreServices } from '@backstage/backend-plugin-api';
import { myToolExtensionPoint } from '@scope/backstage-plugin-my-tool-node';
import { SlackNotificationProvider } from './providers/SlackNotificationProvider';

export const myToolModuleSlackNotifications = createBackendModule({
  pluginId: 'my-tool',
  moduleId: 'slack-notifications',
  register(env) {
    env.registerInit({
      deps: {
        myTool: myToolExtensionPoint,          // from -node package only
        logger: coreServices.logger,
        config: coreServices.rootConfig,
      },
      async init({ myTool, logger, config }) {
        // verify method exists on the ExtensionPoint interface before calling
        const provider = SlackNotificationProvider.fromConfig(config, { logger });
        myTool.addNotificationProvider(provider);  // documented ExtensionPoint method
      },
    });
  },
});
```

```typescript
// src/providers/SlackNotificationProvider.ts
import { NotificationProvider } from '@scope/backstage-plugin-my-tool-node';

// implements the interface the plugin expects — not the plugin's internal class
export class SlackNotificationProvider implements NotificationProvider {
  static fromConfig(
    config: RootConfigService,
    options: { logger: LoggerService },
  ): SlackNotificationProvider {
    const webhookUrl = config.getString('myTool.slack.webhookUrl');
    return new SlackNotificationProvider(webhookUrl, options.logger);
  }

  constructor(
    private readonly webhookUrl: string,
    private readonly logger: LoggerService,
  ) {}

  async notify(message: string): Promise<void> {
    this.logger.debug(`Sending Slack notification: ${message}`);
    await fetch(this.webhookUrl, {
      method: 'POST',
      body: JSON.stringify({ text: message }),
    });
  }
}
```

### Incorrect

```typescript
// ❌ importing from -backend to bypass ExtensionPoint
import { MyToolPlugin } from '@scope/backstage-plugin-my-tool-backend';

async init({ config }) {
  const plugin = new MyToolPlugin();          // bypasses ExtensionPoint entirely
  plugin.internalRegisterProvider(provider); // not a published API
}

// ❌ type assertion to access undocumented methods
async init({ myTool }) {
  (myTool as any).internalMethod(provider);  // ExtensionPoint boundary violated
}

// ❌ multiple capabilities in one module
async init({ myTool, logger, config }) {
  myTool.addNotificationProvider(slackProvider);   // capability 1
  myTool.addNotificationProvider(emailProvider);   // capability 2 — separate module
  myTool.addAuditLogger(auditLogger);              // capability 3 — separate module
}

// ❌ provider class not implementing the expected interface
export class SlackNotificationProvider {
  // missing: implements NotificationProvider from -node package
  async sendMessage(text: string) {   // wrong method name — plugin expects notify()
    ...
  }
}
```

## Verify signal

Reject if:
- Any `src/` file imports from the target plugin's `-backend` package
- `(extensionPoint as any)` or similar type assertions appear in `src/module.ts`
- A provider or action class does not explicitly `implements` the interface from the `-node` package
- `registerInit` calls more than one distinct ExtensionPoint registration method — each capability is a separate module

Flag for human review if:
- The ExtensionPoint method being called is not present in the `-node` package version declared in `package.json` — may be a version mismatch
- The provider or action class has no static `fromConfig` factory method — instantiation with config becomes harder to test
- The module registers a class that is also exported from `src/index.ts` — verify whether consumers actually need to import it directly
- The `-node` package type for the registered class has more than 5 required methods — the interface may be too broad for a single module