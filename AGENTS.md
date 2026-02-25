# Tonkeeper Web - Agent Guidelines

This document provides guidance for coding agents working on the Tonkeeper Web monorepo.

## Project Structure

```
tonkeeper-web/
├── packages/
│   ├── core/          # Core business logic, services, types, API clients
│   ├── uikit/         # Shared React components, hooks, state management
│   └── locales/       # i18n translation files
├── apps/
│   ├── web/           # Web wallet (vite + react)
│   ├── extension/     # Browser extension
│   ├── desktop/       # Electron desktop app
│   ├── twa/           # Telegram Mini App
│   ├── mobile/        # iPad app (Capacitor + Ionic)
│   └── web-swap-widget/  # Swap widget
└── tests/
    └── playwright/    # E2E tests
```

## Build Commands

```bash
# Install dependencies
yarn install

# Build all packages
yarn build

# Build specific app
yarn build:web          # Web wallet
yarn build:extension    # Browser extension
yarn build:desktop      # Desktop app
yarn build:twa          # Telegram Mini App
yarn build:ipad         # iPad app
yarn build:swap-widget  # Swap widget

# Build packages only (used by apps)
yarn build:pkg

# Start development server (in app directory)
cd apps/web && yarn start
cd apps/desktop && yarn start
```

## Lint Commands

```bash
# Lint desktop app
cd apps/desktop && yarn lint

# Lint extension
cd apps/extension && npx eslint --ext .ts,.tsx .

# TypeScript check (run in package/app directory)
npx tsc --noEmit
```

## Test Commands

```bash
# Run all Playwright tests
cd tests/playwright && yarn test

# Run specific test file
cd tests/playwright && npx playwright test tests/path/to/test.spec.ts

# Run tests with specific browser
cd tests/playwright && npx playwright test --project=chromium

# Run tests in headed mode
cd tests/playwright && npx playwright test --headed

# Install Playwright browsers
cd tests/playwright && yarn playwright
```

## Code Style Guidelines

### Formatting (Prettier)

Configuration in `.prettierrc.json`:

-   Single quotes
-   4-space indentation
-   Print width: 100
-   No trailing commas
-   Arrow parens: avoid
-   Bracket spacing: true
-   Semicolons: true

### Imports

```typescript
// External imports first (alphabetically)
import { useCallback, useMemo, useState } from 'react';
import { useMutation, useQuery } from '@tanstack/react-query';
import styled from 'styled-components';

// Internal imports (relative paths)
import { useAppContext } from '../hooks/appContext';
import { QueryKey } from '../libs/queryKey';
```

-   No file extensions for TS/TSX imports
-   Extensions required for: `.scss`, `.css`, `.json`, `.svg`
-   Use `unused-imports/no-unused-imports` - remove unused imports automatically

### TypeScript

-   Strict mode enabled
-   No explicit `any` - use `unknown` or proper types
-   No inferrable types (let TS infer when obvious)
-   Enum members: UPPER_CASE
-   Use Zod schemas for runtime validation and type inference

```typescript
// Good - type inference
const [value, setValue] = useState('');

// Good - explicit when needed
const [user, setUser] = useState<User | null>(null);

// Zod pattern
const schema = z.object({ name: z.string() });
type MyType = z.infer<typeof schema>;
```

### Naming Conventions

-   **Components**: PascalCase (`ErrorBoundary.tsx`, `Link.tsx`)
-   **Hooks**: camelCase with `use` prefix (`useCopyToClipboard.ts`)
-   **Services/Utils**: camelCase (`configService.ts`, `nft.ts`)
-   **Types/Interfaces**: PascalCase (`TonConnectAccount`, `FallbackProps`)
-   **Constants**: UPPER_SNAKE_CASE or camelCase for private
-   **Enum members**: UPPER_CASE

```typescript
export enum CONNECT_EVENT_ERROR_CODES {
    UNKNOWN_ERROR = 0,
    BAD_REQUEST_ERROR = 1
}
```

### React Components

-   Functional components only
-   Use `FC` type with `ComponentProps` for typed props
-   Destructure props in function signature

```typescript
export const Link: FC<ComponentProps<typeof RouterLink> & { contents?: boolean }> = props => {
    const env = useAppTargetEnv();
    return <RouterLinkStyled {...props} />;
};
```

### React Query Patterns

```typescript
export const useWalletNftList = () => {
    const wallet = useActiveWallet();
    const { tonApiV2 } = useActiveApi();

    return useQuery<NFT[], Error>([wallet.rawAddress, QueryKey.nft], async () => {
        const { nftItems } = await new AccountsApi(tonApiV2).getAccountNftItems({
            accountId: wallet.rawAddress
        });
        return nftItems;
    });
};

export const useMarkNftAsSpam = () => {
    return useMutation<void, Error, NftWithCollectionId>(async nft => {
        // mutation logic
    });
};
```

### Styled Components

```typescript
const RouterLinkStyled = styled(RouterLink)<{ contents?: boolean }>`
    ${p => p.contents && 'display: contents'};
    color: inherit;
`;
```

### Error Handling

-   Use `Error` class for rejections
-   Try-catch for async operations
-   Console.error for logging (console.log is warned)

```typescript
try {
    await someAsyncOperation();
} catch (e) {
    console.error(e);
    throw new Error('Operation failed');
}
```

### ESLint Rules to Follow

-   `@typescript-eslint/no-explicit-any`: error
-   `@typescript-eslint/no-inferrable-types`: error
-   `eqeqeq`: smart (use === except when comparing null/undefined)
-   `no-console`: warn (allowed: debug, error, info)
-   `unused-imports/no-unused-imports`: error
-   `unused-imports/no-unused-vars`: error (ignore vars/args with `_` prefix)
-   `i18next/no-literal-string`: warn (no hardcoded UI strings)

## Key Libraries

-   **State/Data**: @tanstack/react-query v4
-   **Routing**: react-router-dom v5
-   **Styling**: styled-components v6
-   **Validation**: zod
-   **Blockchain**: @ton/core, @ton/ton
-   **UI Framework**: @ionic/react (for mobile apps)
-   **Build**: Vite, Turbo, TypeScript

## Translation (i18n)

All user-facing strings must use i18n. Do not hardcode UI strings.

```typescript
const { t } = useTranslation();
return <span>{t('settings_title')}</span>;
```

## Package Manager

This project uses **Yarn 4** with workspaces. Always use `yarn` commands.

## Notes

-   The `packages/core/src/tonApiV2/` directory is auto-generated from OpenAPI specs
-   Do not manually edit generated API code
-   Use existing hooks and services from `@tonkeeper/uikit` and `@tonkeeper/core`

<!-- nx configuration start-->
<!-- Leave the start & end comments to automatically receive updates. -->

## General Guidelines for working with Nx

-   For navigating/exploring the workspace, invoke the `nx-workspace` skill first - it has patterns
    for querying projects, targets, and dependencies
-   When running tasks (for example build, lint, test, e2e, etc.), always prefer running the task
    through `nx` (i.e. `nx run`, `nx run-many`, `nx affected`) instead of using the underlying
    tooling directly
-   Prefix nx commands with the workspace's package manager (e.g., `pnpm nx build`,
    `npm exec nx test`) - avoids using globally installed CLI
-   You have access to the Nx MCP server and its tools, use them to help the user
-   For Nx plugin best practices, check `node_modules/@nx/<plugin>/PLUGIN.md`. Not all plugins have
    this file - proceed without it if unavailable.
-   NEVER guess CLI flags - always check nx_docs or `--help` first when unsure

## Scaffolding & Generators

-   For scaffolding tasks (creating apps, libs, project structure, setup), ALWAYS invoke the
    `nx-generate` skill FIRST before exploring or calling MCP tools

## When to use nx_docs

-   USE for: advanced config options, unfamiliar flags, migration guides, plugin configuration, edge
    cases
-   DON'T USE for: basic generator syntax (`nx g @nx/react:app`), standard commands, things you
    already know
-   The `nx-generate` skill handles generator discovery internally - don't call nx_docs just to look
    up generator syntax

<!-- nx configuration end-->
