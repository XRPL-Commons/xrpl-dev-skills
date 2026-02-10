# Tokens & TrustLines

## Issued Currencies (Fungible Tokens)

On XRPL, fungible tokens are "issued currencies." They require a **trust line** between the holder and the issuer — the holder explicitly opts in to holding that token up to a specified limit.

**Key concepts:**
- **Issuer**: the account that creates/issues the token
- **Trust line**: a bilateral relationship between holder and issuer (created via `TrustSet`)
- **Balance**: held on the trust line object itself (not in the account)
- **Rippling**: the ability for balances to shift through intermediary accounts

### Step 1: Issuer Account Setup

```typescript
// Enable DefaultRipple on issuer (required for token to flow through paths)
const accountSet = {
  TransactionType: "AccountSet",
  Account: issuerAddress,
  SetFlag: 8, // asfDefaultRipple
};
```

Other issuer flags:
| Flag | Value | Effect |
|------|-------|--------|
| `asfDefaultRipple` | 8 | Allow token to flow through payment paths (issuer should enable) |
| `asfRequireAuth` | 2 | Trust lines must be authorized before holder can receive tokens |
| `asfGlobalFreeze` | 7 | Freeze all trust lines (emergency stop) |
| `asfNoFreeze` | 6 | Permanently give up ability to freeze (irreversible) |
| `asfAllowTrustLineClawback` | 16 | Enable clawback (must set before any trust lines exist) |

### Step 2: Holder Creates Trust Line

```typescript
const trustSet = {
  TransactionType: "TrustSet",
  Account: holderAddress,
  LimitAmount: {
    currency: "USD",
    issuer: issuerAddress,
    value: "1000000", // max amount willing to hold
  },
};
// Costs 1 owner reserve (2 XRP) — always warn the user
```

### Step 3: Issuer Sends Tokens

```typescript
// Tokens are "created" by the issuer sending a Payment
const payment = {
  TransactionType: "Payment",
  Account: issuerAddress,
  Destination: holderAddress,
  Amount: {
    currency: "USD",
    issuer: issuerAddress,
    value: "100",
  },
};
```

### Authorized Trust Lines

If the issuer has `asfRequireAuth`:

```typescript
const trustSet = {
  TransactionType: "TrustSet",
  Account: issuerAddress,
  LimitAmount: {
    currency: "USD",
    issuer: holderAddress, // counterparty is the holder
    value: "0",
  },
  Flags: 0x00010000, // tfSetfAuth
};
```

### Freezing

```typescript
// Freeze a specific trust line
const trustSet = {
  TransactionType: "TrustSet",
  Account: issuerAddress,
  LimitAmount: {
    currency: "USD",
    issuer: holderAddress,
    value: "0",
  },
  Flags: 0x00100000, // tfSetFreeze
};

// Global freeze (all trust lines)
const accountSet = {
  TransactionType: "AccountSet",
  Account: issuerAddress,
  SetFlag: 7, // asfGlobalFreeze
};
```

## Multi-Purpose Tokens (MPTs)

MPTs are a newer token standard that don't require trust lines.

### Creating an MPT Issuance

```typescript
const mptCreate = {
  TransactionType: "MPTokenIssuanceCreate",
  Account: issuerAddress,
  MaximumAmount: "1000000",
  AssetScale: 2,            // decimal places
  TransferFee: 100,          // 0.1% (basis points, max 50000)
};
```

### Authorizing Holders

```typescript
// Holder opts in
const holderAuth = {
  TransactionType: "MPTokenAuthorize",
  Account: holderAddress,
  MPTokenIssuanceID: "0000...",
};

// Issuer authorizes (if RequireAuth flag set)
const issuerAuth = {
  TransactionType: "MPTokenAuthorize",
  Account: issuerAddress,
  MPTokenIssuanceID: "0000...",
  Holder: holderAddress,
};
```

### Transferring MPTs

```typescript
const payment = {
  TransactionType: "Payment",
  Account: senderAddress,
  Destination: recipientAddress,
  Amount: {
    mpt_issuance_id: "0000...",
    value: "50",
  },
};
```

## Querying Token State

```typescript
// All trust lines for an account
const lines = await client.request({
  command: "account_lines",
  account: address,
});

// Issuer perspective: who holds my tokens
const balances = await client.request({
  command: "gateway_balances",
  account: issuerAddress,
  hotwallet: [hotWalletAddress],
});
```

## Reserve Implications

- Each trust line: 1 owner reserve (2 XRP)
- Each MPT holding: 1 owner reserve (2 XRP)
- To recover: set trust line limit to 0 with zero balance (deletes the object)
- Always inform users of reserve cost before creating trust lines

## Anti-patterns

- **Don't skip DefaultRipple on the issuer** — tokens won't flow through payment paths.
- **Don't create trust lines without warning about reserves** — users may not have enough XRP.
- **Don't use 3-character codes for non-standard currencies** — use the 160-bit hex format for custom names.
- **Don't confuse issuer-perspective vs holder-perspective** in `TrustSet` — the `issuer` field in `LimitAmount` is always the counterparty.
