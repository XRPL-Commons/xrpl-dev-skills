# Cross-Chain Interoperability

## Key Constraint

XRPL does not support smart contracts. This means:
- **No Axelar Gateway contract** deployed on XRPL (unlike EVM chains)
- **No ability to receive GMP messages** from other chains — you can only **send** messages out
- The XRPL integration uses **XRPL Multisig Signing** + Axelar's **Amplifier Protocol** instead of a gateway contract
- The gateway is a multisig account controlled by the Axelar Verifier set

## Axelar Gateway Addresses

| Network | Gateway Address |
|---------|----------------|
| Mainnet | `rfmS3zqrQrka8wVyhXifEeyTwe8AMz2Yhw` |
| Testnet | `rNrjh1KGZk2jBR3wPfAQnoidtFFYQKbQn2` |

## Sending GMP Messages (XRPL → Other Chains)

A GMP message is an XRPL `Payment` transaction to the gateway with specially structured `Memos`.

### Message Structure

```typescript
import { convertStringToHex } from "xrpl";

const hex = (str: string) => convertStringToHex(str);

const gmpPayment = {
  TransactionType: "Payment",
  Account: senderAddress,
  Amount: "1000000", // 1 XRP — covers cross-chain gas fees
  Destination: "rNrjh1KGZk2jBR3wPfAQnoidtFFYQKbQn2", // testnet gateway
  Memos: [
    {
      Memo: {
        MemoType: hex("type"),
        MemoData: hex("call_contract"),
      },
    },
    {
      Memo: {
        MemoType: hex("destination_address"),
        MemoData: hex("0A90c0Af1B07f6AC34f3520348Dbfae73BDa358E"), // no 0x prefix
      },
    },
    {
      Memo: {
        MemoType: hex("destination_chain"),
        MemoData: hex("xrpl-evm-devnet"), // target chain name
      },
    },
    {
      Memo: {
        MemoType: hex("payload"),
        MemoData: abiEncodedPayload, // ABI-encoded data for the destination contract
      },
    },
  ],
};
```

**All memo fields must be hex-encoded.**

### Memo Fields

| MemoType | MemoData | Description |
|----------|----------|-------------|
| `type` | `call_contract` | Message type (always `call_contract` for GMP) |
| `destination_address` | Contract address (no `0x` prefix) | Target contract on destination chain |
| `destination_chain` | Chain name (e.g., `xrpl-evm-devnet`) | Axelar chain identifier |
| `payload` | ABI-encoded bytes | Data payload for the destination contract |

### Gas Fees

The `Amount` field in the Payment covers cross-chain gas. You can pay with:

```typescript
// XRP (in drops)
Amount: "1000000" // 1 XRP

// Or an IOU token
Amount: {
  currency: "ABC",
  issuer: "r4DVHyEisbgQRAXCiMtP2xuz5h3dDkwqf1",
  value: "1",
}
```

### Complete Example

```typescript
import { Client, Wallet, convertStringToHex } from "xrpl";

const hex = (str: string) => convertStringToHex(str);

async function sendGmpMessage(
  client: Client,
  wallet: Wallet,
  destinationChain: string,
  destinationAddress: string,  // no 0x prefix
  payload: string,             // ABI-encoded hex
  gasFeeDrops: string = "1000000",
) {
  const gatewayAddress = "rNrjh1KGZk2jBR3wPfAQnoidtFFYQKbQn2"; // testnet

  const tx = {
    TransactionType: "Payment",
    Account: wallet.address,
    Amount: gasFeeDrops,
    Destination: gatewayAddress,
    Memos: [
      { Memo: { MemoType: hex("type"), MemoData: hex("call_contract") } },
      { Memo: { MemoType: hex("destination_address"), MemoData: hex(destinationAddress) } },
      { Memo: { MemoType: hex("destination_chain"), MemoData: hex(destinationChain) } },
      { Memo: { MemoType: hex("payload"), MemoData: payload } },
    ],
  };

  const prepared = await client.autofill(tx);
  const signed = wallet.sign(prepared);
  const result = await client.submitAndWait(signed.tx_blob);

  return result;
  // Track on Axelarscan using the transaction hash
}
```

### Frontend with xrpl-connect

```typescript
// Using xrpl-connect's WalletManager for signing
const result = await manager.signAndSubmit({
  TransactionType: "Payment",
  Account: manager.account.address,
  Amount: "1000000",
  Destination: gatewayAddress,
  Memos: [
    { Memo: { MemoType: hex("type"), MemoData: hex("call_contract") } },
    { Memo: { MemoType: hex("destination_address"), MemoData: hex(destinationAddress) } },
    { Memo: { MemoType: hex("destination_chain"), MemoData: hex(destinationChain) } },
    { Memo: { MemoType: hex("payload"), MemoData: payload } },
  ],
});
```

## Adding Gas (Topping Up Underfunded Transactions)

If a cross-chain transaction gets stuck because gas was insufficient (e.g., destination chain gas price spiked), send an `add_gas` message.

### Add Gas Structure

```typescript
const addGas = {
  TransactionType: "Payment",
  Account: senderAddress,
  Amount: "1000000", // additional gas (1 XRP)
  Destination: gatewayAddress,
  Memos: [
    {
      Memo: {
        MemoType: hex("type"),
        MemoData: hex("add_gas"),
      },
    },
    {
      Memo: {
        MemoType: hex("msg_id"),
        MemoData: hex(transactionHash.toLowerCase().replace("0x", "")),
        // The original tx hash, hex-encoded
      },
    },
  ],
};
```

**Important:** The `msg_id` is the original XRPL transaction hash, **hex-encoded as a string** (not raw bytes). For example, hash `c7c653d2...` becomes `hex("c7c653d2...")`.

## XRPL EVM Sidechain

The XRPL EVM sidechain is an EVM-compatible chain connected to the XRP Ledger.

### Architecture
- Separate chain with its own validators and EVM execution
- Connected to XRPL via Axelar bridge
- Uses XRP as the native gas token (bridged from XRPL)
- Standard EVM tooling works (ethers.js, viem, Hardhat, Foundry)

### Destination Chain Names
| Chain | Axelar Name |
|-------|-------------|
| XRPL EVM Devnet | `xrpl-evm-devnet` |
| XRPL EVM Testnet | Check Axelar docs for current name |
| XRPL EVM Mainnet | Check Axelar docs for current name |

## Flow Summary

```
XRPL Account
    │
    │  Payment tx with Memos (call_contract)
    ▼
XRPL Gateway (multisig)
    │
    │  Axelar Relayer picks up tx
    ▼
Axelar Network (Amplifier)
    │
    │  Routes to destination chain
    ▼
Destination Chain Contract
    │
    │  Executes with payload
    ▼
Done (track on Axelarscan)
```

**Limitation:** Messages can only flow **from XRPL to other chains**, not the reverse. XRPL cannot receive GMP messages because it has no smart contracts to execute them.

## When to Use What

| Need | Solution |
|------|----------|
| Send data from XRPL to EVM chain | Axelar GMP (`call_contract`) |
| Bridge XRP to EVM sidechain | Axelar bridge payment to gateway |
| Execute logic on EVM with XRPL trigger | GMP → destination contract |
| Receive data on XRPL from another chain | **Not possible** — XRPL has no smart contracts |
| EVM smart contracts using XRP | Deploy on XRPL EVM sidechain |

## Security Considerations

- **Verify gateway addresses** — always use the official Axelar gateway addresses listed above
- **Gas estimation** — overestimate gas fees; topping up with `add_gas` is possible but adds latency
- **Track transactions** — use Axelarscan to monitor cross-chain message status
- **Hex encoding** — all memo fields must be hex-encoded; double-check encoding to avoid malformed messages
- **Destination address format** — strip the `0x` prefix from EVM addresses before hex-encoding
- **Test on testnet** — always test GMP flows on testnet before mainnet
