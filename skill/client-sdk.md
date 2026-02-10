# Client SDK Patterns

## xrpl.js (JavaScript / TypeScript) — Primary

### Connection Management

```typescript
import { Client } from 'xrpl';

// Always specify the network explicitly
const client = new Client('wss://s.altnet.rippletest.net:51233'); // testnet
await client.connect();

// Always disconnect when done
try {
  // ... do work
} finally {
  await client.disconnect();
}
```

**Network endpoints:**
| Network | WebSocket | Use case |
|---------|-----------|----------|
| Mainnet | `wss://xrplcluster.com` | Production |
| Testnet | `wss://s.altnet.rippletest.net:51233` | Development (default) |
| Devnet | `wss://s.devnet.rippletest.net:51233` | Experimental amendments |

### Account Operations

#### Creating and Funding Accounts

```typescript
import { Wallet } from 'xrpl';

// Generate a new wallet (random keypair)
const wallet = Wallet.generate();

// From a seed/secret
const wallet = Wallet.fromSeed('sEdT...');

// From a mnemonic
const wallet = Wallet.fromMnemonic('abandon abandon ...');

// Fund on testnet via faucet
const fundResult = await client.fundWallet(wallet);
// fundResult.balance contains the funded amount
```

**Reserve requirements (always communicate these):**

- Base reserve: 1 XRP (minimum to activate an account)
- Owner reserve: 0.2 XRP per owned ledger object (trust lines, offers, escrows, etc.)
- Available balance = balance - (base reserve + owner reserve \* owned object count)

#### Querying Account State

```typescript
// Account info (balance, sequence, flags)
const info = await client.request({
  command: 'account_info',
  account: address,
  ledger_index: 'validated',
});

// Account objects (trust lines, offers, escrows, etc.)
const objects = await client.request({
  command: 'account_objects',
  account: address,
  ledger_index: 'validated',
});

// Trust lines specifically
const lines = await client.request({
  command: 'account_lines',
  account: address,
});

// Offers specifically
const offers = await client.request({
  command: 'account_offers',
  account: address,
});
```

### Transaction Lifecycle

#### 1. Build the transaction

```typescript
const tx = {
  TransactionType: 'Payment',
  Account: wallet.address,
  Destination: 'rDestination...',
  Amount: xrpToDrops('10'), // Always use helpers for XRP amounts
  // LastLedgerSequence and Fee are auto-filled by autofill()
};
```

#### 2. Auto-fill missing fields

```typescript
const prepared = await client.autofill(tx);
// Fills: Fee, Sequence, LastLedgerSequence
// ALWAYS let autofill set LastLedgerSequence for reliable submission
```

#### 3. Sign locally

```typescript
const signed = wallet.sign(prepared);
// signed.tx_blob — serialized signed transaction
// signed.hash — transaction hash (for tracking)
```

#### 4. Submit and wait for validation

```typescript
// Preferred: submit and wait for final result
const result = await client.submitAndWait(signed.tx_blob);

// Check the VALIDATED result, not just submission
if (result.result.meta.TransactionResult === 'tesSUCCESS') {
  // Transaction succeeded and is in a validated ledger
}
```

**Critical: never trust submission alone.** A `tesSUCCESS` at submission means "accepted into the queue", not "permanently succeeded." Always wait for validation or poll by hash.

#### Alternative: submit + poll

```typescript
const submitResult = await client.submit(signed.tx_blob);

// Then poll for validation
const txResult = await client.request({
  command: 'tx',
  transaction: signed.hash,
});
// Check txResult.result.validated === true
```

### Transaction Result Codes

| Prefix | Meaning                                        | Action                            |
| ------ | ---------------------------------------------- | --------------------------------- |
| `tes`  | Success (when validated)                       | Done                              |
| `tec`  | Claimed cost only — tx is in ledger but failed | Check specific code, do not retry |
| `tef`  | Failed before applying                         | Fix and resubmit                  |
| `tem`  | Malformed                                      | Fix transaction fields            |
| `ter`  | Retry — could succeed later                    | Retry with same or new sequence   |

### Subscriptions (WebSocket Streaming)

```typescript
// Subscribe to account transactions
await client.request({
  command: 'subscribe',
  accounts: [address],
});

client.on('transaction', (event) => {
  // event.transaction — the transaction
  // event.meta — transaction metadata
  // event.validated — whether it's in a validated ledger
});

// Subscribe to ledger closes
await client.request({
  command: 'subscribe',
  streams: ['ledger'],
});

client.on('ledgerClosed', (ledger) => {
  // ledger.ledger_index, ledger.ledger_hash, etc.
});
```

### Amount Handling

```typescript
import { xrpToDrops, dropsToXrp } from 'xrpl';

// XRP is always specified in drops (1 XRP = 1,000,000 drops)
const drops = xrpToDrops('10'); // "10000000"
const xrp = dropsToXrp('10000000'); // "10"

// Issued currency amounts are objects
const tokenAmount = {
  currency: 'USD',
  issuer: 'rIssuerAddress...',
  value: '100.50',
};
```

**Common mistake:** passing a number instead of a string to `xrpToDrops`. Always pass strings to avoid floating-point issues.

## xrpl-py (Python)

```python
from xrpl.clients import JsonRpcClient
from xrpl.wallet import Wallet, generate_faucet_wallet
from xrpl.models import Payment
from xrpl.transaction import submit_and_wait
from xrpl.utils import xrp_to_drops

client = JsonRpcClient("https://s.altnet.rippletest.net:51234")

# Generate and fund on testnet
wallet = generate_faucet_wallet(client)

# Build and submit
payment = Payment(
    account=wallet.address,
    destination="rDestination...",
    amount=xrp_to_drops(10),
)

result = submit_and_wait(payment, client, wallet)
```

## xrpl4j (Java)

```java
XrplClient client = new XrplClient(
    HttpUrl.get("https://s.altnet.rippletest.net:51234")
);

// Build payment
Payment payment = Payment.builder()
    .account(wallet.classicAddress())
    .destination(Address.of("rDestination..."))
    .amount(XrpCurrencyAmount.ofDrops(10_000_000))
    .build();
```

## Anti-patterns

- **Never send secrets over the network.** Always sign locally.
- **Never skip `LastLedgerSequence`.** Without it, a transaction can be stuck in limbo indefinitely.
- **Never treat submission success as final.** Always wait for validation.
- **Never use floating-point for XRP amounts.** Use string-based helpers (`xrpToDrops`).
- **Never hardcode mainnet endpoints in dev code.** Use environment variables or config.
