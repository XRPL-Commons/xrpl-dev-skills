# Security Checklist

## #1 Vulnerability: Partial Payment Attacks

This is the most common integration vulnerability on XRPL. If you only remember one thing from this file, remember this.

**The attack:** A malicious sender uses `tfPartialPayment` flag to send less than the `Amount` field specifies. If you check `Amount` instead of `delivered_amount`, you'll credit the user for more than they actually sent.

**The fix:** Always check `meta.delivered_amount`:

```typescript
// WRONG — vulnerable to partial payment attack
const amount = tx.Amount;

// CORRECT — always use delivered_amount
const delivered = tx.meta.delivered_amount;
// For very old transactions (pre-2014), delivered_amount might be "unavailable"
// Treat "unavailable" as partial
```

**When processing incoming payments, always:**

1. Check `meta.delivered_amount` (not `Amount`)
2. Verify the `Destination` is your account
3. Check `DestinationTag` matches (if applicable)
4. Confirm the transaction is `validated` (not just submitted)

## Key Management

### Secret Protection

- Never log, store in plain text, or transmit seeds/secrets
- Use environment variables or secret management services
- Never commit secrets to version control
- For frontend: never handle private keys — delegate to wallet via `xrpl-connect`

### Signing Practices

- **Backend/scripts**: sign locally using `Wallet.sign()`, never send secrets to a server
- **Frontend**: always delegate to wallet (`manager.signAndSubmit()`)
- **Production/Backend_infra**: hardware wallets (Ledger) or custodial signing (HSM)

### Regular Keys

```typescript
// Set a regular key (allows signing with an alternate key pair)
const setRegularKey = {
  TransactionType: 'SetRegularKey',
  Account: accountAddress,
  RegularKey: regularKeyAddress,
};
// Use case: rotate signing keys without changing your account address
// The master key can be disabled after setting a regular key
```

### Multi-Signing

```typescript
const signerListSet = {
  TransactionType: 'SignerListSet',
  Account: accountAddress,
  SignerQuorum: 3, // minimum weight needed to authorize
  SignerEntries: [
    { SignerEntry: { Account: signer1, SignerWeight: 2 } },
    { SignerEntry: { Account: signer2, SignerWeight: 1 } },
    { SignerEntry: { Account: signer3, SignerWeight: 1 } },
  ],
};
// signer1 + any other = 3 (passes)
// signer2 + signer3 = 2 (fails quorum)
```

## Transaction Security

### Reliable Submission

- **Always set `LastLedgerSequence`** — without it, a transaction can be stuck in limbo indefinitely. `autofill()` does this automatically.
- **Always wait for validation** — `tesSUCCESS` at submission only means "accepted into the queue."
- **Handle all result codes** — `tec*` means the transaction IS in a validated ledger but failed (fees still consumed).

### Fee Protection

```typescript
// Set a reasonable max fee to avoid fee escalation surprises
const prepared = await client.autofill(tx, {
  maxFeeXRP: '0.01', // never pay more than 0.01 XRP
});
```

### Sequence Number Management

- `autofill()` handles sequence numbers automatically for single transactions.
- For multi-transaction workflows, manually increment sequence numbers:

```typescript
const info = await client.request({
  command: 'account_info',
  account: address,
});
let sequence = info.result.account_data.Sequence;

for (const tx of transactions) {
  tx.Sequence = sequence++;
  // sign and submit each
}
```

- For parallel workflows, use Tickets to avoid sequence conflicts.

## Account Security

### Reserve Awareness

- Always calculate and display available balance (total - reserves) to users
- Warn before operations that increase owner count
- Account deletion: requires Sequence >= 256, costs 2 XRP fee, destination must exist

### Critical Account Flags

| Flag                | Effect                                                 | Warning                                       |
| ------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `asfRequireDestTag` | Reject payments without DestinationTag                 | Set for any exchange/service account          |
| `asfDisallowXRP`    | Advisory: don't send me XRP (not enforced by protocol) | Weak protection only                          |
| `asfRequireAuth`    | Trust lines must be pre-authorized                     | Set BEFORE creating any trust lines           |
| `asfNoFreeze`       | Permanently give up freeze ability                     | **Irreversible**                              |
| `asfDefaultRipple`  | Trust lines ripple by default                          | Required for issuers                          |
| `asfDisableMaster`  | Disable master key                                     | Only after setting regular key or signer list |

## Token Security

### TrustLine Risks

- `NoRipple` on holder side prevents unexpected balance shifts
- Issuers should set `DefaultRipple`; holders generally should NOT
- Freezing: issuers can freeze individual trust lines or all at once (`GlobalFreeze`)

### NFT Security

- Verify `tfTransferable` before listing NFTs for sale
- Transfer fees (royalties) are enforced at protocol level
- Brokered sales: verify broker trust and fee structure before accepting

## Integration Security

### WebSocket Connection

- Always use `wss://` (TLS), never `ws://`
- Handle disconnections gracefully — xrpl.js Client has built-in reconnection
- Don't trust data from non-validated ledgers for financial decisions

### Input Validation

- Validate XRPL addresses (starts with `r`, base58 checksum) before building transactions
- Validate currency codes (3 chars standard, or 40 hex chars for non-standard)
- Validate amounts are positive, reasonable, and string-formatted (not float)
- Check `DestinationTag` format (uint32, 0 to 4294967295)

### Destination Tag Enforcement

```typescript
// Before sending, check if destination requires tags
const info = await client.request({
  command: 'account_info',
  account: destinationAddress,
});
const flags = info.result.account_data.Flags;
const requiresDestTag = (flags & 0x00020000) !== 0;
// If true and you don't include DestinationTag → tecDST_TAG_NEEDED
```

## Amount Handling

```typescript
import { xrpToDrops, dropsToXrp } from 'xrpl';

// ALWAYS use helpers — never manual math
xrpToDrops('10'); // "10000000"
dropsToXrp('10000000'); // "10"

// NEVER use floating point for financial math
// BAD:  10.5 * 1000000 → might give 10499999.999999998
// GOOD: xrpToDrops("10.5") → "10500000"
```

## Common Vulnerabilities Summary

| Vulnerability              | Impact                     | Prevention                               |
| -------------------------- | -------------------------- | ---------------------------------------- |
| Partial payment attack     | Credit more than received  | Check `delivered_amount`, not `Amount`   |
| Missing LastLedgerSequence | Transaction stuck in limbo | Use `autofill()` or set manually         |
| Trusting submission result | Act on unvalidated data    | Wait for `validated: true`               |
| Floating-point amounts     | Rounding errors            | Use string amounts + helpers             |
| Exposed secrets            | Account compromise         | Never handle keys in frontend            |
| Missing DestinationTag     | Lost funds at exchanges    | Check `lsfRequireDestTag` before sending |
| Unchecked reserves         | Failed transactions        | Calculate available balance first        |
