# Payments, Escrows & Channels

## XRP Payments

### Direct Payment

```typescript
import { xrpToDrops } from "xrpl";

const payment = {
  TransactionType: "Payment",
  Account: senderAddress,
  Destination: destinationAddress,
  Amount: xrpToDrops("25"), // always use the helper
  DestinationTag: 12345,    // required if destination has asfRequireDestTag
};
```

### Partial Payments

```typescript
const payment = {
  TransactionType: "Payment",
  Account: senderAddress,
  Destination: destinationAddress,
  Amount: xrpToDrops("100"),    // maximum to deliver
  SendMax: xrpToDrops("100"),
  DeliverMin: xrpToDrops("95"), // minimum acceptable
  Flags: 0x00020000,            // tfPartialPayment
};
```

**Critical security note:** When receiving payments, always check `meta.delivered_amount`, NOT the `Amount` field. The `Amount` field can be misleading with partial payments. See [security.md](security.md).

## Cross-Currency Payments

See [dex-amm.md](dex-amm.md) for path finding and cross-currency payment details.

## Escrows

Escrows lock XRP until a time or condition is met.

### Time-Based Escrow

```typescript
// Create escrow (lock XRP until a specific time)
const escrowCreate = {
  TransactionType: "EscrowCreate",
  Account: senderAddress,
  Destination: recipientAddress,
  Amount: xrpToDrops("100"),
  FinishAfter: rippleTimeFromUnix(Date.now() / 1000 + 86400), // 24 hours from now
  CancelAfter: rippleTimeFromUnix(Date.now() / 1000 + 604800), // 7 days (safety)
};

// Finish escrow (recipient or anyone claims after FinishAfter)
const escrowFinish = {
  TransactionType: "EscrowFinish",
  Account: recipientAddress, // anyone can submit this
  Owner: senderAddress,
  OfferSequence: escrowCreateSequence,
};

// Cancel escrow (sender reclaims after CancelAfter)
const escrowCancel = {
  TransactionType: "EscrowCancel",
  Account: senderAddress,
  Owner: senderAddress,
  OfferSequence: escrowCreateSequence,
};
```

### Conditional Escrow (Crypto-Conditions)

```typescript
const escrowCreate = {
  TransactionType: "EscrowCreate",
  Account: senderAddress,
  Destination: recipientAddress,
  Amount: xrpToDrops("100"),
  Condition: "A0258020...", // PREIMAGE-SHA-256 condition (hex)
  CancelAfter: rippleTimeFromUnix(Date.now() / 1000 + 604800),
};

// Finish with fulfillment
const escrowFinish = {
  TransactionType: "EscrowFinish",
  Account: recipientAddress,
  Owner: senderAddress,
  OfferSequence: escrowCreateSequence,
  Condition: "A0258020...",
  Fulfillment: "A0228020...", // preimage that satisfies the condition
};
```

**Ripple epoch helper:**
```typescript
function rippleTimeFromUnix(unixSeconds: number): number {
  return Math.floor(unixSeconds) - 946684800; // offset from Jan 1, 2000
}
```

## Payment Channels

Payment channels enable fast, off-ledger micropayments settled on-ledger.

### Creating a Channel

```typescript
const channelCreate = {
  TransactionType: "PaymentChannelCreate",
  Account: senderAddress,
  Destination: recipientAddress,
  Amount: xrpToDrops("100"),   // total channel capacity
  SettleDelay: 3600,            // 1 hour dispute window
  PublicKey: senderPublicKey,   // key used to sign claims
};
```

### Off-Ledger Claims

```typescript
// Sender creates signed claims off-ledger (no transaction needed)
// These are just signed messages, not XRPL transactions
const claim = {
  channel: channelId,
  amount: xrpToDrops("5"), // cumulative amount (not incremental)
  // signed by the channel's PublicKey
};
```

### Redeeming Claims

```typescript
const channelClaim = {
  TransactionType: "PaymentChannelClaim",
  Account: recipientAddress,
  Channel: channelId,
  Balance: xrpToDrops("5"),     // new balance to claim
  Amount: xrpToDrops("5"),
  Signature: claimSignature,
  PublicKey: senderPublicKey,
};
```

### Closing a Channel

```typescript
// Request close (starts SettleDelay countdown)
const close = {
  TransactionType: "PaymentChannelClaim",
  Account: senderAddress,
  Channel: channelId,
  Flags: 0x00010000, // tfClose
};
```

## Checks

Checks are like paper checks — the recipient can cash them at their convenience.

### Creating a Check

```typescript
const checkCreate = {
  TransactionType: "CheckCreate",
  Account: senderAddress,
  Destination: recipientAddress,
  SendMax: xrpToDrops("50"), // maximum the check can be cashed for
  Expiration: rippleTimeFromUnix(Date.now() / 1000 + 604800),
};
```

### Cashing a Check

```typescript
// Cash for exact amount
const checkCash = {
  TransactionType: "CheckCash",
  Account: recipientAddress,
  CheckID: "CHECK_ID...",
  Amount: xrpToDrops("50"),
};

// Cash for flexible amount
const checkCashFlex = {
  TransactionType: "CheckCash",
  Account: recipientAddress,
  CheckID: "CHECK_ID...",
  DeliverMin: xrpToDrops("45"), // accept at least this much
};
```

### Cancelling a Check

```typescript
const checkCancel = {
  TransactionType: "CheckCancel",
  Account: senderAddress, // sender or recipient can cancel
  CheckID: "CHECK_ID...",
};
```

## Reserve Implications

- Each escrow: 1 owner reserve (2 XRP)
- Each payment channel: 1 owner reserve (2 XRP) + locked XRP
- Each check: 1 owner reserve (2 XRP)
- Completing/cancelling these objects frees the reserve
