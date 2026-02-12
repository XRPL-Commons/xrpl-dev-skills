---
name: xrpl-dev
description: End-to-end XRPL development playbook. Covers XRP Ledger dApp development including project scaffolding (create-xrp), wallet integration (xrpl-connect), client SDKs, transactions, tokens, NFTs, DEX/AMM, cross-chain interoperability (Axelar), and security best practices.
user-invocable: true
---

# XRPL Development Skill

## What this Skill is for
Use this Skill when the user asks for:
- XRPL dApp development (any language/framework)
- Scaffolding a new XRPL project
- Frontend wallet connection (Crossmark, Xaman, GemWallet)
- Account creation, funding, and management
- Transaction building, signing, and submission
- Token operations (TrustLines, issued currencies, MPTs)
- NFT operations (XLS-20 NFTokens)
- DEX interactions (offers, order books, path finding)
- AMM (Automated Market Maker) operations
- Cross-chain interoperability (Axelar bridge, XRPL EVM sidechain)
- Payment channels, escrows, and checks
- Security review of XRPL transactions and integrations

## Default stack decisions (opinionated)

1. **Scaffolding: create-xrp**
   - For new projects, start with `npx create-xrp my-app`.
   - Scaffolds a Turborepo monorepo with Next.js (React) or Nuxt (Vue).
   - Pre-configured: `xrpl-connect` wallet connection, network switching, Tailwind CSS, transaction UI.
   - Smart contract support is experimental — skip it by default.

2. **Client SDK: xrpl.js first (JavaScript/TypeScript)**
   - Use `xrpl` for all new JS/TS client code.
   - For Python projects, use `xrpl-py`.
   - For Java/Kotlin, use `xrpl4j`.

3. **Frontend: xrpl-connect first**
   - Use `xrpl-connect` for all wallet connection and signing UX.
   - Framework-agnostic web component (`<xrpl-wallet-connector>`), works with React, Vue, Next.js, Nuxt, vanilla JS.
   - Built-in adapters for Xaman, Crossmark, GemWallet, WalletConnect, and Ledger.
   - Event-driven `WalletManager` handles connection state, session persistence, and auto-reconnect.

4. **Network: testnet for development**
   - Always default to testnet (`wss://s.altnet.rippletest.net:51233`).
   - Use devnet for experimental/amendment features.
   - Never hardcode mainnet endpoints in development code.

5. **Transaction signing: local or wallet-delegated**
   - For backend/scripts: sign locally, never send secrets over the network.
   - For frontend: delegate signing to the connected wallet.
   - For production backends: prefer custodial signing services or hardware wallets.

6. **Account management: reserves-aware**
   - Always check and communicate reserve requirements before operations.
   - Base reserve (currently 10 XRP) + owner reserve (2 XRP per owned object).
   - Warn users when operations will lock up reserves.

7. **Error handling: explicit**
   - Always check `validated` status, not just submission `tesSUCCESS`.
   - Handle `tec*` codes (claimed-but-failed) differently from `tef*`/`tem*` codes.
   - Implement retry logic for `terQUEUED` and `tefPAST_SEQ`.

## Operating procedure (how to execute tasks)

### 1. Classify the task layer
- **Scaffolding** — new project setup
- **Frontend/wallet layer** — wallet connection, signing UX, displaying state
- **Client/scripts layer** — building or sending transactions, querying ledger state
- **Token/asset layer** — TrustLines, issued currencies, NFTs, MPTs
- **DEX/AMM layer** — offers, order books, AMM pools, path finding
- **Interoperability layer** — Axelar bridge, EVM sidechain
- **Security review** — cross-cutting concern

### 2. Pick the right building blocks
- New projects: `npx create-xrp` to scaffold (Next.js or Nuxt)
- Frontend: `xrpl-connect` (WalletManager + web component) + `xrpl` for state queries
- JS/TS backend/scripts: `xrpl` package
- Python projects: `xrpl-py`
- Cross-chain: Axelar GMP via XRPL Payment memos

### 3. Implement with XRPL-specific correctness
Always be explicit about:
- network (mainnet / testnet / devnet) + WebSocket endpoint
- account reserves (base + owner reserves) and their implications
- transaction fees (auto-filled vs explicit, fee escalation)
- sequence numbers (auto-filled vs explicit, for multi-tx workflows)
- `LastLedgerSequence` (always set for reliable submission)
- flags and field requirements per transaction type
- trust line limits and issuer relationships
- NFT transfer fees and broker mechanics

### 4. Add tests
- Unit test transaction building and serialization.
- Integration test against testnet (use faucet for funding).
- Frontend: test wallet connection and signing flows with mocked providers.

### 5. Deliverables expectations
When you implement changes, provide:
- exact files changed + diffs
- commands to install/build/test
- a short "risk notes" section for anything touching signing, fees, reserves, or asset transfers

## Content Sources
This skill incorporates best practices from:
- The [Official XRPL Documentation](https://xrpl.org/docs) — the authoritative reference for protocol specifications, transaction types, and API details.
- The foundational course materials: [xrpl-training-2026-january](https://github.com/XRPL-Commons/xrpl-training-2026-january).

## Progressive disclosure (read when needed)
- Client SDK patterns (xrpl.js): [client-sdk.md](client-sdk.md)
- Frontend & wallet integration: [frontend.md](frontend.md)
- Tokens & TrustLines: [tokens.md](tokens.md)
- NFTs (XLS-20): [nfts.md](nfts.md)
- DEX & AMM: [dex-amm.md](dex-amm.md)
- Payments, escrows & channels: [payments.md](payments.md)
- Cross-chain interoperability: [interoperability.md](interoperability.md)
- Security checklist: [security.md](security.md)
- Resources & references: [resources.md](resources.md)
- A framework-agnostic wallet connection toolkit for the XRP Ledger [xrpl-connect](https://github.com/XRPL-Commons/xrpl-connect)