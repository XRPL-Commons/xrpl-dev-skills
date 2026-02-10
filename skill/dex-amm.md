# DEX & AMM

## On-Ledger DEX (Order Book)

XRPL has a built-in decentralized exchange. Any two assets can be traded via offers on the order book.

### Creating Offers

```typescript
// Sell 100 USD for XRP
const offer = {
  TransactionType: "OfferCreate",
  Account: traderAddress,
  TakerGets: {
    currency: "USD",
    issuer: issuerAddress,
    value: "100",
  },
  TakerPays: "50000000", // 50 XRP in drops
};
// From the taker's perspective: they "get" USD and "pay" XRP
// From your perspective: you're selling USD for XRP
```

**Offer flags:**
| Flag | Value | Effect |
|------|-------|--------|
| `tfPassive` | 0x00010000 | Don't cross existing offers, only sit on the book |
| `tfImmediateOrCancel` | 0x00020000 | Fill what you can immediately, cancel the rest |
| `tfFillOrKill` | 0x00040000 | Execute fully or not at all |
| `tfSell` | 0x00080000 | Treat as sell order (sell exact TakerGets amount) |

### Cancelling Offers

```typescript
const cancel = {
  TransactionType: "OfferCancel",
  Account: traderAddress,
  OfferSequence: 12345, // sequence number of the OfferCreate transaction
};
```

Offers also auto-cancel when:
- They're fully consumed
- They expire (if `Expiration` was set)
- A new offer with `OfferSequence` replaces them

### Querying the Order Book

```typescript
const book = await client.request({
  command: "book_offers",
  taker_gets: { currency: "USD", issuer: issuerAddress },
  taker_pays: { currency: "XRP" },
  limit: 20,
});
// Returns sorted list of offers (best price first)
```

## Automated Market Maker (AMM)

XRPL has native AMM pools alongside the order book. Trades are automatically routed through whichever provides a better price.

### Creating an AMM Pool

```typescript
const ammCreate = {
  TransactionType: "AMMCreate",
  Account: creatorAddress,
  Amount: "10000000000", // 10,000 XRP in drops
  Amount2: {
    currency: "USD",
    issuer: issuerAddress,
    value: "10000",
  },
  TradingFee: 500, // 0.5% (in basis points, 1-1000 = 0.001%-1%)
};
// Creator must have both assets and a trust line for the LP token
```

**Important:** Creating an AMM costs a special fee (currently higher than normal) and creates an AMM account that holds the pool assets.

### Depositing Liquidity

```typescript
// Dual-asset deposit (proportional)
const deposit = {
  TransactionType: "AMMDeposit",
  Account: depositorAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  Amount: "5000000000",  // 5,000 XRP
  Amount2: {
    currency: "USD",
    issuer: issuerAddress,
    value: "5000",
  },
  Flags: 0x00100000, // tfTwoAsset
};

// Single-asset deposit
const singleDeposit = {
  TransactionType: "AMMDeposit",
  Account: depositorAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  Amount: "1000000000", // deposit only XRP
  Flags: 0x00080000, // tfSingleAsset
};
```

### Withdrawing Liquidity

```typescript
// Withdraw proportionally by LP token amount
const withdraw = {
  TransactionType: "AMMWithdraw",
  Account: depositorAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  LPTokenIn: {
    currency: lpTokenCurrency,
    issuer: ammAccountAddress,
    value: "100",
  },
  Flags: 0x00010000, // tfLPToken
};

// Single-asset withdraw
const singleWithdraw = {
  TransactionType: "AMMWithdraw",
  Account: depositorAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  Amount: "500000000", // withdraw only XRP
  Flags: 0x00080000, // tfSingleAsset
};
```

### AMM Voting (Trading Fee)

LP token holders can vote to change the trading fee:

```typescript
const vote = {
  TransactionType: "AMMVote",
  Account: voterAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  TradingFee: 600, // vote for 0.6%
};
// Fee is a weighted average of up to 8 active votes
```

### AMM Auction (Discounted Trading)

```typescript
const bid = {
  TransactionType: "AMMBid",
  Account: bidderAddress,
  Asset: { currency: "XRP" },
  Asset2: { currency: "USD", issuer: issuerAddress },
  BidMin: { currency: lpTokenCurrency, issuer: ammAccountAddress, value: "10" },
};
// Winner gets discounted trading fee for 24 hours
```

### Querying AMM State

```typescript
const ammInfo = await client.request({
  command: "amm_info",
  asset: { currency: "XRP" },
  asset2: { currency: "USD", issuer: issuerAddress },
});
// Returns: pool balances, LP token info, trading fee, auction slot, vote slots
```

## Cross-Currency Payments

XRPL automatically finds the best path through both the order book and AMM:

```typescript
// Find payment paths
const paths = await client.request({
  command: "ripple_path_find",
  source_account: senderAddress,
  destination_account: recipientAddress,
  destination_amount: {
    currency: "EUR",
    issuer: eurIssuer,
    value: "100",
  },
});

// Send cross-currency payment using discovered paths
const payment = {
  TransactionType: "Payment",
  Account: senderAddress,
  Destination: recipientAddress,
  Amount: {
    currency: "EUR",
    issuer: eurIssuer,
    value: "100",
  },
  SendMax: "60000000", // max XRP willing to spend (in drops)
  Paths: paths.result.alternatives[0].paths_computed,
};
```

## Reserve Implications

- Each open offer: 1 owner reserve (2 XRP)
- AMM LP token trust line: 1 owner reserve (2 XRP)
- Cancel offers you no longer need to free up reserves
