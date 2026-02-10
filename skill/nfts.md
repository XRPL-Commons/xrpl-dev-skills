# NFTs

## Minting

### NFTokenMint

```typescript
const mint = {
  TransactionType: 'NFTokenMint',
  Account: minterAddress,
  URI: convertStringToHex('https://example.com/metadata.json'), // hex-encoded
  NFTokenTaxon: 0, // arbitrary collection grouping
  TransferFee: 5000, // 5% royalty on secondary sales (basis points, max 50000)
  Flags: 8, // tfTransferable (see flags below)
};
```

**Flags:**
| Flag | Value | Effect |
|------|-------|--------|
| `tfBurnable` | 1 | Issuer can burn even after transfer |
| `tfOnlyXRP` | 2 | Can only be traded for XRP (no issued currencies) |
| `tfTrustLine` | 4 | Auto-create trust line for non-XRP offers (rarely needed) |
| `tfTransferable` | 8 | Can be transferred/sold to others (almost always set) |

**Important:** Without `tfTransferable`, the NFT can only be sent back to the issuer.

### Batch Minting

NFTs are stored in pages of up to 32. When a page splits, it costs an additional reserve. For efficient minting:

- Mint in batches to the same account
- Be aware that minting the 33rd NFT triggers a page split (extra reserve)

## Trading

### Creating Sell Offers

```typescript
const sellOffer = {
  TransactionType: 'NFTokenCreateOffer',
  Account: ownerAddress,
  NFTokenID: '000800...',
  Amount: '10000000', // 10 XRP in drops
  Flags: 1, // tfSellNFToken
  Destination: buyerAddress, // optional: restrict to specific buyer
  Expiration: Math.floor(Date.now() / 1000) + 86400 - 946684800, // Ripple epoch
};
```

### Creating Buy Offers

```typescript
const buyOffer = {
  TransactionType: 'NFTokenCreateOffer',
  Account: buyerAddress,
  NFTokenID: '000800...',
  Owner: currentOwnerAddress, // required for buy offers
  Amount: '10000000',
};
```

### Accepting Offers

```typescript
// Accept a sell offer (buyer calls this)
const accept = {
  TransactionType: 'NFTokenAcceptOffer',
  Account: buyerAddress,
  NFTokenSellOffer: 'OFFER_ID...',
};

// Accept a buy offer (owner calls this)
const accept = {
  TransactionType: 'NFTokenAcceptOffer',
  Account: ownerAddress,
  NFTokenBuyOffer: 'OFFER_ID...',
};
```

### Brokered Sales

A broker matches a buy offer and sell offer, potentially taking a fee:

```typescript
const brokered = {
  TransactionType: 'NFTokenAcceptOffer',
  Account: brokerAddress,
  NFTokenSellOffer: 'SELL_OFFER_ID...',
  NFTokenBuyOffer: 'BUY_OFFER_ID...',
  NFTokenBrokerFee: '500000', // broker's cut (must be less than price difference)
};
```

**Rules:**

- Buy offer amount >= sell offer amount + broker fee
- The NFT must have `tfTransferable` flag
- Transfer fee (royalty) is deducted from the seller's proceeds automatically

## Burning

```typescript
const burn = {
  TransactionType: 'NFTokenBurn',
  Account: ownerAddress, // owner can always burn
  NFTokenID: '000800...',
};
// Issuer can also burn if tfBurnable was set at mint
```

## Querying

```typescript
// All NFTs owned by an account
const nfts = await client.request({
  command: 'account_nfts',
  account: address,
  limit: 100,
});
// Paginate with `marker` if more than limit

// Buy offers for an NFT
const buyOffers = await client.request({
  command: 'nft_buy_offers',
  nft_id: '000800...',
});

// Sell offers for an NFT
const sellOffers = await client.request({
  command: 'nft_sell_offers',
  nft_id: '000800...',
});
```

## Reserve Implications

- NFTs are stored in NFTokenPages (up to 32 per page)
- Each NFTokenPage costs 1 owner reserve (2 XRP)
- First NFT minted creates first page (2 XRP reserve)
- Reserve cost is shared across NFTs on the same page
- Burning all NFTs on a page releases that reserve

## Anti-patterns

- **Don't forget `tfTransferable`** — without it, NFTs can only go back to the issuer.
- **Don't set transfer fees too high** — they're capped at 50% and can discourage trading.
- **Don't ignore Ripple epoch** — XRPL expiration uses seconds since Jan 1, 2000 (not Unix epoch). Offset: `946684800`.
- **Don't assume offer IDs are predictable** — always query for existing offers before accepting.
