# XRPL Development Skill for Claude Code

A comprehensive Claude Code skill for modern XRP Ledger development.

## Overview

This skill provides Claude Code with deep knowledge of the XRPL development ecosystem:

- **Client SDK**: `xrpl.js` (v4.x), `xrpl-py`, `xrpl4j`
- **Frontend**: `xrpl-connect` wallet toolkit (Xaman, Crossmark, GemWallet, WalletConnect, Ledger)
- **Tokens**: Issued currencies, TrustLines, Multi-Purpose Tokens (MPTs)
- **NFTs**: XLS-20 NFTokens — minting, trading, brokered sales
- **DEX & AMM**: Order book offers, AMM pools, cross-currency routing
- **Payments**: XRP and cross-currency payments, escrows, payment channels, checks
- **Interoperability**: Axelar bridge, XRPL EVM sidechain
- **Security**: Partial payment attacks, key management, reserve awareness, common vulnerabilities

## Installation

### Quick Install

```bash
npx skills add https://github.com/xrpl-commons/xrpl-dev-skill
```

### Manual Install

```bash
git clone https://github.com/xrpl-commons/xrpl-dev-skill
cd xrpl-dev-skill
./install.sh
```

## Skill Structure

```
skill/
├── SKILL.md                 # Main skill definition (required)
├── client-sdk.md            # Connection, accounts, tx lifecycle, signing, querying
├── frontend.md              # Wallet connection, React patterns, tx signing UX
├── tokens.md                # Issued currencies, TrustLines, MPTs
├── nfts.md                  # XLS-20 NFTokens
├── dex-amm.md               # Order book + AMM
├── payments.md              # XRP payments, cross-currency, escrows, channels, checks
├── interoperability.md      # Axelar bridge, XRPL EVM sidechain interop
├── security.md              # Vulnerability patterns and prevention
└── resources.md             # Curated reference links
```

## Usage

Once installed, Claude Code will automatically use this skill when you ask about:

- XRPL dApp development
- Wallet connection and signing flows
- Transaction building, signing, and submission
- Token issuance and management
- NFT minting and trading
- DEX/AMM interactions
- Cross-chain interoperability
- Security reviews

### Example Prompts

```
"Help me set up a Next.js app with XRPL wallet connection"
"Create a token issuance flow with TrustLines"
"Build an NFT marketplace with brokered sales"
"How do I swap tokens using the AMM?"
"Bridge assets from XRPL to EVM sidechain via Axelar"
"Review this integration for partial payment vulnerabilities"
```

## Progressive Disclosure

The skill uses Claude Code's progressive disclosure pattern. The main `SKILL.md` provides core guidance, and Claude reads specialized markdown files only when needed for specific tasks.

## Contributing

Contributions are welcome! Please ensure any updates reflect current XRPL ecosystem best practices.

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT License - see [LICENSE](LICENSE) for details.
