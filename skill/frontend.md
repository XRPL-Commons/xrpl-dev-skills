# Frontend & Wallet Integration

## Scaffolding: create-xrp

For new projects, always start here:

```bash
npx create-xrp my-app
```

The CLI prompts for:
1. **Project name** (validated npm-compatible name)
2. **Framework**: Next.js (React) or Nuxt (Vue)
3. **Package manager**: pnpm (recommended), npm, or yarn

By default (no smart contract option), this scaffolds a **flat project** with:
- `xrpl-connect` wallet connection pre-wired
- Network switching (testnet / devnet / mainnet)
- Tailwind CSS styling
- Transaction form component
- Account info display
- WalletProvider (React Context) or useWallet composable (Nuxt)

### Generated Structure

```
my-app/
├── components/
│   ├── Header.js               # Nav + wallet display
│   ├── AccountInfo.js          # Connected account details
│   ├── WalletConnector.js      # xrpl-connect web component
│   └── TransactionForm.js      # Send XRP payments
├── hooks/                      # (Next.js) or composables/ (Nuxt)
│   ├── useWalletManager.js     # Initialize WalletManager
│   └── useWalletConnector.js
├── lib/
│   └── networks.js             # Network endpoint configs
├── app/                        # (Next.js App Router) or pages/ (Nuxt)
│   ├── layout.js
│   └── page.js
├── package.json
└── ...
```

### Environment Variables

**Next.js** (`.env.local`):
```env
NEXT_PUBLIC_XAMAN_API_KEY=<your_key>
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=<your_id>
NEXT_PUBLIC_DEFAULT_NETWORK=testnet
```

**Nuxt** (`.env`):
```env
VITE_XAMAN_API_KEY=<your_key>
VITE_WALLETCONNECT_PROJECT_ID=<your_id>
```

### Dev Commands

```bash
pnpm dev          # Start dev server
pnpm build        # Production build
pnpm lint         # Lint
```

If you already have an existing project, install manually:

```bash
npm install xrpl-connect xrpl
```

---

## xrpl-connect — the default

Use `xrpl-connect` for all frontend wallet integration. It provides a framework-agnostic web component and an event-driven WalletManager that handles multiple wallets, session persistence, and auto-reconnection.

### Supported Wallets

| Wallet | Adapter | Config | Notes |
|--------|---------|--------|-------|
| Xaman | `XamanAdapter` | `apiKey` required (from https://apps.xumm.dev) | QR codes, push notifications, mobile-first |
| Crossmark | `CrossmarkAdapter` | No config needed | Browser extension |
| GemWallet | `GemWalletAdapter` | No config needed | Browser extension |
| WalletConnect | `WalletConnectAdapter` | `projectId` required (from https://cloud.walletconnect.com) | Mobile wallet support |
| Ledger | `LedgerAdapter` | Optional `accountIndex`, `timeout` | Hardware wallet, requires HTTPS + Chrome/Edge/Opera |

## Core Setup

### 1. Create the WalletManager

```typescript
import {
  WalletManager,
  XamanAdapter,
  CrossmarkAdapter,
  GemWalletAdapter,
} from "xrpl-connect";

const manager = new WalletManager({
  adapters: [
    new XamanAdapter({ apiKey: "YOUR_XAMAN_API_KEY" }),
    new CrossmarkAdapter(),
    new GemWalletAdapter(),
  ],
  network: "testnet", // 'mainnet' | 'testnet' | 'devnet'
  autoConnect: true,   // restore previous session
});
```

### 2. Attach the Web Component

```html
<xrpl-wallet-connector
  id="wallet-connector"
  wallets="xaman,crossmark,gemwallet"
  style="--xc-primary-color: #3b99fc;"
></xrpl-wallet-connector>
```

```typescript
const connector = document.getElementById("wallet-connector");
connector.setWalletManager(manager);
```

### 3. Listen to Events

```typescript
manager.on("connect", (account) => {
  console.log("Connected:", account.address);
});

manager.on("disconnect", () => {
  console.log("Disconnected");
});

manager.on("accountChange", (account) => {
  console.log("Account switched:", account.address);
});

manager.on("networkChange", (network) => {
  console.log("Network switched:", network);
});

manager.on("error", (error) => {
  console.error("Wallet error:", error);
});
```

### 4. Sign and Submit Transactions

```typescript
// xrpl-connect handles signing via the connected wallet
// No private keys touch your application
const result = await manager.signAndSubmit({
  TransactionType: "Payment",
  Account: manager.account.address,
  Destination: "rDestination...",
  Amount: "10000000", // 10 XRP in drops
});

// Sign a message (for authentication / proof of ownership)
const signature = await manager.signMessage("Hello XRPL");
```

## Framework Integration

### React / Next.js

```tsx
import { useEffect, useRef, useState } from "react";
import {
  WalletManager,
  XamanAdapter,
  CrossmarkAdapter,
  GemWalletAdapter,
} from "xrpl-connect";

function WalletConnector() {
  const connectorRef = useRef<HTMLElement>(null);
  const [manager] = useState(
    () =>
      new WalletManager({
        adapters: [
          new XamanAdapter({ apiKey: process.env.NEXT_PUBLIC_XAMAN_KEY! }),
          new CrossmarkAdapter(),
          new GemWalletAdapter(),
        ],
        network: "testnet",
        autoConnect: true,
      })
  );
  const [account, setAccount] = useState<string | null>(null);

  useEffect(() => {
    manager.on("connect", (acc) => setAccount(acc.address));
    manager.on("disconnect", () => setAccount(null));

    if (connectorRef.current) {
      connectorRef.current.setWalletManager(manager);
    }

    return () => {
      manager.disconnect();
    };
  }, [manager]);

  return (
    <div>
      {account ? (
        <p>Connected: {account}</p>
      ) : (
        <xrpl-wallet-connector ref={connectorRef} wallets="xaman,crossmark,gemwallet" />
      )}
    </div>
  );
}
```

**Next.js note:** The web component requires browser APIs. Use `"use client"` directive or dynamic import with `ssr: false`.

### Vue 3

```vue
<template>
  <xrpl-wallet-connector ref="connector" wallets="xaman,crossmark,gemwallet" />
  <p v-if="account">Connected: {{ account }}</p>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { WalletManager, XamanAdapter, CrossmarkAdapter, GemWalletAdapter } from "xrpl-connect";

const connector = ref(null);
const account = ref(null);

const manager = new WalletManager({
  adapters: [
    new XamanAdapter({ apiKey: import.meta.env.VITE_XAMAN_KEY }),
    new CrossmarkAdapter(),
    new GemWalletAdapter(),
  ],
  network: "testnet",
  autoConnect: true,
});

onMounted(() => {
  manager.on("connect", (acc) => (account.value = acc.address));
  manager.on("disconnect", () => (account.value = null));
  connector.value?.setWalletManager(manager);
});

onUnmounted(() => {
  manager.disconnect();
});
</script>
```

## Styling / Customization

The web component is fully customizable via CSS variables:

```css
xrpl-wallet-connector {
  /* Colors */
  --xc-primary-color: #3b99fc;
  --xc-background-color: #000637;
  --xc-background-secondary: #1a1a3e;
  --xc-text-color: #f5f4e7;
  --xc-success-color: #10b981;
  --xc-warning-color: #f59e0b;
  --xc-danger-color: #ef4444;

  /* Layout */
  --xc-border-radius: 12px;
  --xc-overlay-background: rgba(0, 0, 0, 0.7);
}
```

## Error Handling

| Error Code | Meaning | Recovery |
|------------|---------|----------|
| `WALLET_NOT_FOUND` | Wallet extension not installed | Prompt user to install |
| `CONNECTION_FAILED` | Connection rejected by user | Retry or suggest another wallet |
| `SIGN_FAILED` | User rejected the signing request | Inform user, no retry |
| `INVALID_PARAMS` | Bad transaction format | Validate transaction before submitting |
| `NETWORK_ERROR` | Communication failure | Check connection, retry |

```typescript
manager.on("error", (error) => {
  switch (error.code) {
    case "WALLET_NOT_FOUND":
      showInstallPrompt(error.walletName);
      break;
    case "SIGN_FAILED":
      showMessage("Transaction cancelled by user");
      break;
    default:
      showMessage(`Error: ${error.message}`);
  }
});
```

## Transaction Signing UX

Follow this flow for any user-initiated transaction:

1. **Build** — prepare the transaction, show the user what they're approving
2. **Confirm** — display amount, destination, fees, and reserve impact in your UI
3. **Sign** — call `manager.signAndSubmit(tx)` (opens wallet popup / QR)
4. **Wait** — show pending spinner while wallet processes
5. **Result** — show success with explorer link, or error with actionable message

## UX Checklist

- [ ] Show network indicator (mainnet vs testnet) prominently
- [ ] Display reserve-adjusted available balance, not raw balance
- [ ] Show fee estimate before signing
- [ ] Disable submit buttons during pending transactions
- [ ] Show transaction hash immediately after submission (link to explorer)
- [ ] Distinguish between "submitted" and "validated" states
- [ ] Handle wallet not installed gracefully (link to install page)
- [ ] Show clear error messages for common failures (insufficient balance, no trust line)
- [ ] For Xaman: handle QR code expiry and timeout
- [ ] Use `autoConnect: true` to restore sessions across page reloads
