# Tonkeeper Web - Comprehensive Onboarding Guide

## Executive Summary

Tonkeeper Web is a **multi-platform cryptocurrency wallet** for the TON blockchain (and TRON USDT).
It's built as a **monorepo** with shared business logic (`packages/core`), shared UI components
(`packages/uikit`), and platform-specific apps (`apps/web`, `apps/mobile`, etc.).

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         APPS LAYER                              │
│  ┌─────────┐ ┌───────────┐ ┌──────────┐ ┌─────┐ ┌────────────┐ │
│  │   Web   │ │ Extension │ │ Desktop  │ │ TWA │ │   Mobile   │ │
│  │ (Vite)  │ │  (Vite)   │ │(Electron)│ │(Vite)│ │(Capacitor) │ │
│  └────┬────┘ └─────┬─────┘ └────┬─────┘ └──┬──┘ └─────┬──────┘ │
└───────┼────────────┼────────────┼──────────┼──────────┼────────┘
        │            │            │          │          │
        └────────────┴────────────┼──────────┴──────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────┐
│                         UIKIT LAYER                            │
│  ┌──────────────┐ ┌────────────┐ ┌─────────┐ ┌───────────────┐ │
│  │  Components  │ │   Hooks    │ │  State  │ │    Pages      │ │
│  │  (React/TSX) │ │ (Data/UX)  │ │(RQuery) │ │  (Routes)     │ │
│  └──────────────┘ └────────────┘ └─────────┘ └───────────────┘ │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────┐
│                          CORE LAYER                            │
│  ┌──────────────┐ ┌────────────┐ ┌─────────┐ ┌───────────────┐ │
│  │   Services   │ │   Entries  │ │  APIs   │ │    Utils      │ │
│  │ (Business)   │ │  (Types)   │ │(TonAPI) │ │  (Helpers)    │ │
│  └──────────────┘ └────────────┘ └─────────┘ └───────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Key Technologies

| Technology               | Purpose                      | Location                            |
| ------------------------ | ---------------------------- | ----------------------------------- |
| **React 18**             | UI Framework                 | Everywhere                          |
| **React Query v4**       | Server state & caching       | `packages/uikit/src/state/`         |
| **React Router v5**      | Navigation                   | `packages/uikit/src/libs/routes.ts` |
| **Styled Components v6** | CSS-in-JS styling            | All components                      |
| **Zod**                  | Runtime validation           | `packages/core/src/entries/`        |
| **@ton/core**            | TON blockchain               | `packages/core/`                    |
| **Ionic React**          | Mobile UI primitives         | `apps/mobile/`                      |
| **Vite**                 | Build tool                   | Web, Mobile, TWA                    |
| **Turbo**                | Monorepo build orchestration | Root `turbo.json`                   |

---

## 3. Data Flow Architecture

### 3.1 State Management Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    React Query Cache                         │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ Account/    │ │ Blockchain   │ │ UI State             │ │
│  │ Wallet Data │ │ Data (NFTs,  │ │ (Settings, Theme,    │ │
│  │             │ │ Jettons, etc)│ │ Preferences)         │ │
│  └──────┬──────┘ └──────┬───────┘ └──────────┬───────────┘ │
└─────────┼───────────────┼────────────────────┼──────────────┘
          │               │                    │
          ▼               ▼                    ▼
    ┌─────────┐     ┌─────────┐         ┌──────────┐
    │ Local   │     │  TonAPI │         │ Storage  │
    │ Storage │     │  (HTTP) │         │ (Async)  │
    └─────────┘     └─────────┘         └──────────┘
```

### 3.2 Key Query Keys

Located in `packages/uikit/src/libs/queryKey.ts`:

```typescript
export const QueryKey = {
    account: 'account',
    wallet: 'wallet',
    activity: 'activity',
    nft: 'nft',
    nftCollection: 'nft-collection',
    jettons: 'jettons',
    tonConnectConnection: 'ton-connect-connection',
    pro: 'pro',
    dashboardData: 'dashboard-data'
    // ... more keys
};
```

### 3.3 Data Fetching Pattern

```typescript
// Example: packages/uikit/src/state/wallet.ts
export const useActiveAccountQuery = () => {
    const storage = useAccountsStorage();
    return useQuery<Account | null, Error>(
        [QueryKey.account, QueryKey.wallet], // Query key
        () => storage.getActiveAccount(), // Fetcher function
        { keepPreviousData: true } // Options
    );
};
```

---

## 4. App Initialization Flow (Web)

```
apps/web/src/index.tsx
        │
        ▼
apps/web/src/App.tsx:App
        │
        ├── BrowserRouter
        │       │
        │       ▼
        │   QueryClientProvider (React Query)
        │       │
        │       ▼
        │   AppSdkContext.Provider (BrowserAppSdk instance)
        │       │
        │       ▼
        │   TranslationContext.Provider (i18n)
        │       │
        │       ▼
        │   StorageContext.Provider (sdk.storage)
        │       │
        │       ▼
        │   UserThemeProvider
        │       │
        │       ▼
        │   AppContext.Provider (API configs, features)
        │       │
        │       ▼
        │   Content → MobileView or DesktopView
```

**Key initialization steps:**

1. Load active account (`useActiveAccountQuery`)
2. Load all accounts (`useAccountsStateQuery`)
3. Load user language (`useUserLanguage`)
4. Load fiat currency (`useUserFiatQuery`)
5. Check app lock status (`useLock`)
6. Fetch server config (`useTonenpointConfig`)
7. Setup analytics (`useAnalytics`)

---

## 5. Platform SDK Abstraction

The `IAppSdk` interface (`packages/core/src/AppSdk.ts`) abstracts platform differences:

```typescript
interface IAppSdk {
    storage: IStorage; // Persistent storage
    keychain?: IKeychainService; // Secure key storage (mobile)
    notifications?: NotificationService;
    biometry?: BiometryService;

    // UI operations
    copyToClipboard(value: string): void;
    openPage(url: string): Promise<void>;
    confirm(options: ConfirmOptions): Promise<boolean>;
    topMessage(message: string): void;
    alert(message: string): void;

    // Events for cross-component communication
    uiEvents: IEventEmitter<UIEvents>;

    // Platform info
    targetEnv: 'web' | 'mobile' | 'tablet' | 'extension' | 'twa';
    isIOs(): boolean;
    isStandalone(): boolean;
}
```

**Implementations:**

-   **Web**: `apps/web/src/libs/appSdk.ts` → `BrowserAppSdk`
-   **Mobile**: `apps/mobile/src/libs/appSdk.ts` → `CapacitorAppSdk`

---

## 6. User Flows

### 6.1 First-Time User / Wallet Creation

```
User opens app
        │
        ▼
┌───────────────────┐
│ Initialize.tsx    │  ← "Welcome to Tonkeeper" screen
│ (onboarding page) │
└─────────┬─────────┘
          │ User clicks "Continue"
          ▼
┌───────────────────┐
│ AddWalletNotif.   │  ← Modal with options
│ (controlled)      │
└─────────┬─────────┘
          │
          ├─→ Create New Wallet
          │       │
          │       ▼
          │   Generate mnemonic → Set password → Done
          │
          ├─→ Import Wallet (Mnemonic)
          │       │
          │       ▼
          │   Enter 24 words → Set password → Done
          │
          ├─→ Watch Only
          │       │
          │       ▼
          │   Enter address → Done
          │
          ├─→ Ledger
          │       │
          │       ▼
          │   Connect device → Select accounts → Done
          │
          └─→ Keystone
                  │
                  ▼
              Scan QR → Done
```

**Key file**: `packages/uikit/src/state/wallet.ts` (`useCreateAccountMnemonic`)

### 6.2 Send Transaction Flow

```
User clicks "Send"
        │
        ▼
┌───────────────────┐
│ SendNotifications │  packages/uikit/src/components/transfer/
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ RecipientView     │  Enter address / Scan QR / Select contact
└─────────┬─────────┘
          │ Valid address entered
          ▼
┌───────────────────┐
│ AmountView        │  Enter amount, select asset (TON/Jetton)
└─────────┬─────────┘
          │ Amount confirmed
          ▼
┌───────────────────┐
│ ConfirmTransfer   │  Review: recipient, amount, fees
│ View              │  Choose sender type: Regular/Gasless/Battery
└─────────┬─────────┘
          │ User confirms
          ▼
┌───────────────────┐
│ useSendTransfer() │  Execute transaction
│                   │  1. Get signer (mnemonic/ledger)
│                   │  2. Encode message
│                   │  3. Sign
│                   │  4. Broadcast via API
└───────────────────┘
```

**Key files:**

-   `packages/uikit/src/components/transfer/SendNotifications.tsx`
-   `packages/core/src/service/ton-blockchain/ton-asset-transaction.service.ts`

### 6.3 View Assets Flow (Home Page)

```
Home.tsx loads
        │
        ├── useAllChainsAssets()  ← Fetch TON balance + Jettons + TRON USDT
        │       │
        │       └── Parallel fetch from TonAPI and TronAPI
        │
        ├── useWalletFilteredNftList()  ← Fetch NFTs, filter spam
        │       │
        │       └── AccountsApi.getAccountNftItems()
        │
        └── usePreFetchRates()  ← Fetch token prices
                │
                ▼
        Render:
        ┌─────────────────────────────┐
        │ Balance (total fiat value)  │
        ├─────────────────────────────┤
        │ Actions: Send / Receive     │
        ├─────────────────────────────┤
        │ Assets: TON, Jettons, NFTs  │
        │ (CompactView or TabsView)   │
        └─────────────────────────────┘
```

**Key file**: `packages/uikit/src/pages/home/Home.tsx`

---

## 7. Routing Structure

### 7.1 Route Definitions

Located in `packages/uikit/src/libs/routes.ts`:

```typescript
export enum AppRoute {
    home: '/',
    import: '/import',
    settings: '/settings',
    activity: '/activity',
    purchases: '/purchases',  // NFTs
    coins: '/coins',
    swap: '/swap',
    browser: '/browser',
    // ...
}
```

### 7.2 Web vs Mobile Layout

**Web** (`apps/web/src/AppMobile.tsx`):

```tsx
// MobileView for narrow screens
<>
    <Header /> // Top bar with account selector
    <Switch>
        {' '}
        // Route switch
        <Route path={AppRoute.home} component={Home} />
        <Route path={AppRoute.activity} component={Activity} />
        ...
    </Switch>
    <Footer /> // Bottom navigation tabs
</>
```

**Mobile** (`apps/mobile/src/app-content/NarrowContent.tsx`):

-   Uses Ionic's `IonRouterOutlet` for native-feeling transitions
-   Same routes, but wrapped in Ionic components

---

## 8. Account Types & Data Models

### 8.1 Account Types

Located in `packages/core/src/entries/account.ts`:

```typescript
type Account =
    | AccountTonMnemonic // Standard wallet (24-word mnemonic)
    | AccountMAM // HD wallet (multiple derivations)
    | AccountTonTestnet // Testnet wallet
    | AccountLedger // Hardware wallet
    | AccountKeystone // QR hardware wallet
    | AccountTonWatchOnly // View-only wallet
    | AccountTonMultisig; // Multisig wallet
```

### 8.2 Key Data Structures

```typescript
// packages/core/src/entries/wallet.ts
interface TonWalletStandard {
    id: string;              // Wallet ID (raw address)
    publicKey: string;       // Ed25519 public key
    version: WalletVersion;  // V4R2, V5R1, etc.
    rawAddress: string;      // 0:... format
}

// packages/core/src/service/wallet/configService.ts
interface TonWalletConfig {
    pinnedNfts: string[];
    hiddenNfts: string[];
    pinnedTokens: string[];
    hiddenTokens: string[];
    trustedNfts: string[];
    spamNfts: string[];
    batterySettings: { ... };
}
```

---

## 9. API Layer

### 9.1 TonAPI

Located in `packages/core/src/tonApiV2/` - auto-generated from OpenAPI specs.

Key APIs:

```typescript
// accounts
AccountsApi.getAccount({ accountId });
AccountsApi.getAccountNftItems({ accountId, limit, offset });
AccountsApi.getAccountEvents({ accountId });

// jettons
JettonsApi.getAccountJettons({ accountId });

// blockchain
BlockchainApi.execGetMethodForBlockchainAccount({ accountId, methodName });
```

### 9.2 Using APIs in Components

```typescript
// packages/uikit/src/state/nft.ts
export const useWalletNftList = () => {
    const wallet = useActiveWallet();
    const { tonApiV2 } = useActiveApi();

    return useQuery<NFT[], Error>([wallet.rawAddress, QueryKey.nft], async () => {
        const { nftItems } = await new AccountsApi(tonApiV2).getAccountNftItems({
            accountId: wallet.rawAddress,
            offset: 0,
            limit: 1000
        });
        return nftItems;
    });
};
```

---

## 10. Component Structure

### 10.1 Component Types

| Type                   | Location                                   | Examples                         |
| ---------------------- | ------------------------------------------ | -------------------------------- |
| **Pages**              | `packages/uikit/src/pages/`                | `Home.tsx`, `Activity.tsx`       |
| **UI Components**      | `packages/uikit/src/components/`           | `Button.tsx`, `Notification.tsx` |
| **Feature Components** | `packages/uikit/src/components/{feature}/` | `transfer/`, `nft/`, `settings/` |
| **Shared**             | `packages/uikit/src/components/shared/`    | `Link.tsx`, `ErrorBoundary.tsx`  |

### 10.2 Component Pattern

```tsx
// packages/uikit/src/components/shared/Link.tsx
import { FC, ComponentProps } from 'react';
import { Link as RouterLink } from 'react-router-dom';
import styled from 'styled-components';

export const Link: FC<ComponentProps<typeof RouterLink> & { contents?: boolean }> = props => {
    const env = useAppTargetEnv();

    if (env === 'mobile') {
        return <MobileRouterLinkStyled {...props} />;
    }
    return <RouterLinkStyled {...props} />;
};

const RouterLinkStyled = styled(RouterLink)<{ contents?: boolean }>`
    ${p => p.contents && 'display: contents'};
    color: inherit;
`;
```

---

## 11. TonConnect (DApp Integration)

### 11.1 Connection Flow

```
DApp generates TonConnect URL
        │
        ▼
User scans QR / clicks deep link
        │
        ▼
parseTonConnect() extracts parameters
        │
        ▼
getManifest() fetches DApp metadata
        │
        ▼
User sees: "Connect to [DApp Name]?"
        │
        ▼ (Approve)
        │
┌───────────────────────────┐
│ Generate ton_addr +       │
│ ton_proof (signature)     │
└───────────┬───────────────┘
            │
            ▼
Store connection in account
DApp receives wallet address
```

### 11.2 Transaction Signing Flow

```
DApp sends transaction request
        │
        ▼
User sees transaction details
        │
        ▼
┌───────────────────────────┐
│ ConfirmTransactionModal   │
│ - Recipient addresses     │
│ - Amounts                 │
│ - Estimated fees          │
│ - Sender type choice      │
└───────────┬───────────────┘
            │ (Confirm)
            ▼
Sign & broadcast transaction
        │
        ▼
Return result to DApp
```

**Key files**: `packages/uikit/src/components/connect/`

---

## 12. How to Tinker

### 12.1 Running the Web App

```bash
# 1. Install dependencies
yarn install

# 2. Build shared packages first
yarn build:pkg

# 3. Start web development server
cd apps/web && yarn start
# Opens at http://localhost:5173
```

### 12.2 Making Changes

| To change...     | Edit files in...                                 |
| ---------------- | ------------------------------------------------ |
| Home page layout | `packages/uikit/src/pages/home/Home.tsx`         |
| Balance display  | `packages/uikit/src/components/home/Balance.tsx` |
| Send flow        | `packages/uikit/src/components/transfer/`        |
| Wallet creation  | `packages/uikit/src/state/wallet.ts`             |
| API calls        | `packages/uikit/src/state/*.ts`                  |
| Routes           | `packages/uikit/src/libs/routes.ts`              |
| Translations     | `packages/locales/`                              |

### 12.3 Common Tasks

**Add a new hook:**

```typescript
// packages/uikit/src/hooks/useMyFeature.ts
import { useQuery } from '@tanstack/react-query';
import { useActiveWallet } from '../state/wallet';
import { QueryKey } from '../libs/queryKey';

export const useMyFeature = () => {
    const wallet = useActiveWallet();

    return useQuery([wallet.id, QueryKey.myFeature], async () => {
        // fetch data
    });
};
```

**Add a new route:**

```typescript
// 1. Add to packages/uikit/src/libs/routes.ts
export enum AppRoute {
    // ...
    myNewPage: '/my-new-page'
}

// 2. Add route in app routing (e.g., AppMobile.tsx)
<Route path={AppRoute.myNewPage} component={MyNewPage} />
```

---

## 13. Key Files Reference

| Purpose               | File Path                                    |
| --------------------- | -------------------------------------------- |
| App entry (web)       | `apps/web/src/App.tsx`                       |
| App entry (mobile)    | `apps/mobile/src/app/App.tsx`                |
| Wallet state hooks    | `packages/uikit/src/state/wallet.ts`         |
| NFT state             | `packages/uikit/src/state/nft.ts`            |
| Activity/transactions | `packages/uikit/src/state/activity.ts`       |
| API configuration     | `packages/core/src/entries/apis.ts`          |
| Account types         | `packages/core/src/entries/account.ts`       |
| Wallet service        | `packages/core/src/service/walletService.ts` |
| SDK interface         | `packages/core/src/AppSdk.ts`                |
| Routes                | `packages/uikit/src/libs/routes.ts`          |
| Query keys            | `packages/uikit/src/libs/queryKey.ts`        |

---

## 14. Tips for React Beginners

1. **Start with the Home page** (`packages/uikit/src/pages/home/Home.tsx`) - it's the simplest entry
   point
2. **Trace data flow**: Find where `useQuery` is called, then trace back to the API
3. **Use React DevTools** to see component hierarchy and props
4. **Check the Network tab** to see API calls to TonAPI
5. **State is mostly in React Query** - look for `useQuery` and `useMutation`
6. **Components are in `packages/uikit/src/components/`** organized by feature

---

## 15. Debugging Tips

### React Query DevTools

The project uses React Query v4. To debug queries:

```typescript
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Add inside QueryClientProvider
<ReactQueryDevtools initialIsOpen={false} />;
```

### Common Issues

1. **"No active account" error**: Check if wallet is created/imported
2. **API errors**: Check network tab, verify API endpoint in `AppContext`
3. **Stale data**: Call `queryClient.invalidateQueries()` after mutations
4. **Build errors**: Run `yarn build:pkg` before starting apps

### Useful Console Commands

```javascript
// In browser console, access React Query cache
window.__REACT_QUERY_DEVTOOLS_GLOBAL_QUERY_CLIENT__;

// Check stored accounts
localStorage.getItem('tonkeeper_accounts');
```
