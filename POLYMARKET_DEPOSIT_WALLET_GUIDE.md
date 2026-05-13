# Polymarket Deposit Wallets — Implementation & Trading Guide

> Project-agnostic reference for implementing the Polymarket **deposit-wallet** model end-to-end: deterministic address derivation, relayer-sponsored deployment, approvals, funding, ERC-7739-wrapped POLY_1271 order signing, and the V2 CLOB transaction set (buy/sell/split/merge/redeem).
>
> **Network:** Polygon Mainnet (chain ID 137)
> **Collateral:** pUSD
> **Wallet contract:** ERC-1967 proxy
> **Signature type:** `POLY_1271` (3) with ERC-7739 `TypedDataSign` wrap
> **On-chain ops:** submitted via Polymarket relayer (`executeDepositWalletBatch`) — gas sponsored
>
> Companion: [CLOB-V2-TRANSACTIONS.md](./CLOB-V2-TRANSACTIONS.md) covers V2 order semantics for **any** wallet type. This guide layers the deposit-wallet specifics on top.

---

## 1. What a deposit wallet is

A **deposit wallet** is the Polymarket "managed smart wallet" model for new API users. Mechanically:

| Property | Value |
|---|---|
| Contract type | ERC-1967 proxy |
| Factory (Polygon 137) | `0x00000000000Fb5C9ADea0298D729A0CB3823Cc07` |
| Implementation (Polygon 137) | `0x58CA52ebe0DadfdF531Cde7062e76746de4Db1eB` |
| Addressing | Deterministic CREATE2 from owner EOA |
| Order validation | ERC-1271 via the wallet's `isValidSignature` |
| On-chain actions | Submitted by Polymarket relayer; wallet executes its own batch |
| Gas | **Relayer-sponsored.** Owner EOA pays nothing for wallet ops. |
| Collateral | pUSD held **at the wallet** (not at the EOA) |

Three signing patterns coexist on Polymarket — choose deliberately:

| Wallet model | `signatureType` enum | Signer | Funder | Gas |
|---|---|---|---|---|
| EOA | `EOA = 0` | EOA itself | same address | EOA pays for everything |
| Gnosis Safe (`POLY_PROXY` legacy) | `POLY_GNOSIS_SAFE = 2` | EOA owner | Safe contract | EOA pays POL for split/merge/redeem; CLOB orders gas-free |
| **Deposit wallet** | `POLY_1271 = 3` | EOA owner | Deposit-wallet contract | **Relayer sponsors everything**; EOA only signs |

Deposit wallets are the right choice when:
- You're a new API integrator (Polymarket's "primary pathway" per their docs).
- You want zero POL gas on the owner EOA.
- You want one wallet that holds collateral, holds positions, and signs orders — no separate Safe to maintain.

Deposit wallets are **not** the right choice when you need:
- Direct EOA control over funds (no smart-contract custody).
- Browser-wallet UX with Metamask (those workflows still use the legacy Safe-`POLY_PROXY` path).

---

## 2. Required infrastructure

### 2.1 SDKs

```
npm install \
  @polymarket/builder-relayer-client@^0.0.9 \
  @polymarket/builder-signing-sdk \
  @polymarket/clob-client-v2@^1.0.6 \
  ethers@^5 viem axios
```

- `@polymarket/builder-relayer-client@0.0.9` adds `RelayClient.deployDepositWallet()`, `executeDepositWalletBatch()`, and `deriveDepositWalletAddress()`.
- `@polymarket/clob-client-v2@1.0.6` is the first version whose `ExchangeOrderBuilderV2` produces the **ERC-7739 wrapped** POLY_1271 signature deposit wallets require. Earlier versions (e.g. `1.0.0`) accept `signatureType=3` in the enum but emit a plain EIP-712 signature that the deposit wallet's `isValidSignature` rejects as INVALID. If you can't bump globally, vendor the wrap (see §6.4).
- `@polymarket/builder-signing-sdk` carries `BuilderConfig` for relayer auth.

### 2.2 Credentials

| Variable | What it is | Source |
|---|---|---|
| `DEPOSIT_OWNER_PRIVATE_KEY` | EOA private key — signs orders and `WALLET` batches | Generate or reuse a key dedicated to this purpose |
| `BUILDER_API_KEY`, `BUILDER_SECRET`, `BUILDER_PASS_PHRASE` | Relayer HMAC creds | Provisioned by Polymarket. Required for every relayer call. |
| `RELAYER_URL` | Relayer endpoint | `https://relayer-v2.polymarket.com` (mainnet) |
| `CLOB_HOST` | CLOB endpoint | `https://clob.polymarket.com` |
| `RPC_URL` | Polygon RPC | Any provider — Infura / Alchemy / your own node. The owner Wallet must have a Provider attached or `EthersSigner` rejects with `signer is missing provider`. |
| `DEPOSIT_WALLET` | Predicted/deployed wallet address | Derived from owner EOA (see §3); persist after first run |

CLOB L1/L2 auth and relayer auth are **separate systems** — builder creds go in headers for relayer requests; CLOB derives its own API key from the EOA signer (§7).

---

## 3. Deterministic address derivation

The deposit wallet address is fully determined by the owner EOA + chain. You can compute it **before** deployment:

```
walletId        = bytes32(owner)                       // 20-byte EOA, left-padded
encArgs         = abi.encode(factory, walletId)
salt            = keccak256(encArgs)
bytecodeHash    = SoladyLibClone.initCodeHashERC1967(implementation, encArgs)
depositWallet   = CREATE2(deployer = factory, salt, bytecodeHash)
```

In practice, use the SDK helper:

```typescript
import { RelayClient } from '@polymarket/builder-relayer-client';
import { BuilderConfig } from '@polymarket/builder-signing-sdk';
import { Wallet } from '@ethersproject/wallet';
import { ethers } from 'ethers';

const provider = new ethers.providers.JsonRpcProvider(process.env.RPC_URL);
// Wallet MUST have a provider — builder-abstract-signer throws otherwise.
const owner = new Wallet(process.env.DEPOSIT_OWNER_PRIVATE_KEY!, provider);

const builderConfig = new BuilderConfig({
  localBuilderCreds: {
    key: process.env.BUILDER_API_KEY!,
    secret: process.env.BUILDER_SECRET!,
    passphrase: process.env.BUILDER_PASS_PHRASE!,
  },
});

const relayer = new RelayClient(
  process.env.RELAYER_URL!,
  137,                  // Polygon mainnet
  owner as any,
  builderConfig,
);

const predicted = await relayer.deriveDepositWalletAddress();
// predicted is the CREATE2 address; equals the eventual deployed address.
```

Two practical consequences:
1. You can fund / pre-approve at the predicted address before deployment lands. (Funds sent before deploy are visible after deploy — the address is the same.)
2. Owner-key compromise = wallet compromise. Rotate carefully; the address is bound to one key forever.

---

## 4. Deploying the wallet

### 4.1 Relayer request (`WALLET-CREATE`)

```typescript
const isDeployed = await relayer.getDeployed(predicted, 'WALLET');
if (!isDeployed) {
  const resp = await relayer.deployDepositWallet();
  const confirmed = await resp.wait();
  if (
    !confirmed ||
    (confirmed.state !== 'STATE_MINED' && confirmed.state !== 'STATE_CONFIRMED')
  ) {
    throw new Error(`WALLET-CREATE did not confirm (state=${confirmed?.state})`);
  }
  // confirmed.transactionHash is the on-chain deploy tx
}
```

Wire-level payload sent by the SDK:

```json
{
  "type": "WALLET-CREATE",
  "from": "0xOwnerEOA",
  "to":   "0x00000000000Fb5C9ADea0298D729A0CB3823Cc07"
}
```

- **No user signature required** for `WALLET-CREATE` — the relayer authorizes by the deterministic-address mapping.
- The relayer's `pollUntilState` reaches `STATE_MINED` once the deploy tx is included on Polygon (~5–15 s typical).
- After `STATE_MINED`, `getDeployed(addr, 'WALLET')` returns `true`. Re-running `deployDepositWallet` would be rejected by the factory.

### 4.2 Relayer transaction states

| State | Meaning |
|---|---|
| `STATE_NEW` | Submitted to relayer; not yet broadcast |
| `STATE_EXECUTED` | Relayer broadcasted to mempool |
| `STATE_MINED` | Included in a block |
| `STATE_CONFIRMED` | Included + sufficient confirmations |
| `STATE_FAILED` | Tx reverted or rejected before broadcast |
| `STATE_INVALID` | Relayer refused (bad signature, exhausted nonce, malformed) |

Always treat `STATE_MINED` and `STATE_CONFIRMED` as success; everything else is a failure that needs investigation. The SDK's `.wait()` polls until one of those states or a timeout.

---

## 5. Approvals (must run **from** the deposit wallet)

A freshly deployed deposit wallet has no on-chain approvals. The CLOB and the redemption adapters can't pull pUSD or move conditional tokens until you authorize them — and the authorization **must originate from the deposit wallet**, not from the owner EOA. The relayer batches these calls via `executeDepositWalletBatch`.

### 5.1 The six required approvals

| Call | Target contract | Spender / Operator | Why |
|---|---|---|---|
| `pUSD.approve(MAX)` | pUSD (`0xC011…2DFB`) | CTF Exchange V2 (`0xE111…996B`) | CLOB binary-market BUYs pull pUSD |
| `pUSD.approve(MAX)` | pUSD | NegRisk Exchange V2 (`0xe222…0F59`) | CLOB 3+-outcome BUYs pull pUSD |
| `pUSD.approve(MAX)` | pUSD | NegRiskAdapter (`0xd91E…5296`) | NegRisk split/merge/redeem pulls pUSD |
| `CTF.setApprovalForAll(true)` | CTF (`0x4D97…6045`) | CTF Exchange V2 | CLOB binary-market SELLs move outcome tokens |
| `CTF.setApprovalForAll(true)` | CTF | NegRisk Exchange V2 | CLOB 3+-outcome SELLs move outcome tokens |
| `CTF.setApprovalForAll(true)` | CTF | NegRiskAdapter | NegRisk split/merge/redeem moves outcome tokens |

If your bot will use the `CtfCollateralAdapter` for **binary** split/merge/redeem (recommended — see §9), add:

| Call | Target | Spender | Why |
|---|---|---|---|
| `pUSD.approve(MAX)` | pUSD | CtfCollateralAdapter (`0xAdA1…cE1f`) | Binary split pulls pUSD |
| `CTF.setApprovalForAll(true)` | CTF | CtfCollateralAdapter | Binary redeem moves CTF positions |

### 5.2 Submitting the approval batch

```typescript
import {
  DepositWalletCall,
  RelayerTransactionState,
} from '@polymarket/builder-relayer-client';

const PUSD = '0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB';
const CTF  = '0x4D97DCd97eC945f40cF65F87097ACe5EA0476045';
const MAX  = ethers.constants.MaxUint256.toString();

const SPENDERS = [
  { name: 'CTF Exchange V2',     addr: '0xE111180000d2663C0091e4f400237545B87B996B' },
  { name: 'NegRisk Exchange V2', addr: '0xe2222d279d744050d28e00520010520000310F59' },
  { name: 'NegRiskAdapter',      addr: '0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296' },
  // { name: 'CtfCollateralAdapter', addr: '0xAdA100Db00Ca00073811820692005400218FcE1f' },
];

const erc20   = new ethers.utils.Interface(['function approve(address,uint256) returns (bool)']);
const erc1155 = new ethers.utils.Interface(['function setApprovalForAll(address,bool)']);

const calls: DepositWalletCall[] = SPENDERS.flatMap((s) => [
  { target: PUSD, value: '0', data: erc20.encodeFunctionData('approve', [s.addr, MAX]) },
  { target: CTF,  value: '0', data: erc1155.encodeFunctionData('setApprovalForAll', [s.addr, true]) },
]);

const deadline = Math.floor(Date.now() / 1000 + 3600).toString();
const resp = await relayer.executeDepositWalletBatch(calls, depositWallet, deadline);
const confirmed = await resp.wait();
if (
  !confirmed ||
  (confirmed.state !== RelayerTransactionState.STATE_MINED &&
   confirmed.state !== RelayerTransactionState.STATE_CONFIRMED)
) {
  throw new Error(`approvals did not confirm (state=${confirmed?.state})`);
}
```

### 5.3 What's in the `WALLET` batch under the hood

The SDK signs and POSTs:

```json
{
  "type": "WALLET",
  "from": "0xOwnerEOA",
  "to":   "0x00000000000Fb5C9ADea0298D729A0CB3823Cc07",
  "nonce": "<from /nonce?address=0xOwnerEOA&type=WALLET>",
  "signature": "0x<65-byte EIP-712 ECDSA>",
  "depositWalletParams": {
    "depositWallet": "0xWalletAddr",
    "deadline":      "<unix-sec>",
    "calls": [
      { "target": "0xTokenOrContract", "value": "0", "data": "0xCalldata" },
      …
    ]
  }
}
```

The signature is a **normal 65-byte EIP-712** signature over the `DepositWallet.Batch` typed-data:

```
EIP-712 domain:
  name:              "DepositWallet"
  version:           "1"
  chainId:           137
  verifyingContract: <depositWallet>
```

This is **different** from the CLOB order signature (§6) — same wallet, two distinct domains. Don't try to share signatures between batch and order paths.

### 5.4 Idempotency

The approval batch is safe to re-run. ERC-20 `approve(MAX, MAX)` is a no-op when the allowance is already MAX; `setApprovalForAll(true)` is a no-op when already true. Re-running `WALLET-CREATE` is also safe — `getDeployed` returns `true`, you simply skip.

---

## 6. Order signing — POLY_1271 + ERC-7739

Two signatures matter for deposit wallets and they are **not** interchangeable:

| Use | Domain | Signature shape |
|---|---|---|
| Wallet batches (`WALLET`, see §5) | `DepositWallet` v1 @ wallet address | Plain 65-byte EIP-712 |
| **CLOB orders** | `Polymarket CTF Exchange` v2 @ exchange | **ERC-7739 `TypedDataSign` wrapped** |

For CLOB orders, the deposit wallet's on-chain `isValidSignature(hash, signature)` decodes an ERC-7739 envelope and rejects any signature that isn't shaped exactly as the spec requires.

### 6.1 The signed order struct (CTF Exchange V2)

```solidity
struct Order {
    uint256 salt;
    address maker;          // deposit wallet
    address signer;         // deposit wallet (NOT the EOA — both fields equal the wallet)
    uint256 tokenId;
    uint256 makerAmount;
    uint256 takerAmount;
    uint8   side;           // 0 = BUY, 1 = SELL
    uint8   signatureType;  // 3 = POLY_1271
    uint256 timestamp;
    bytes32 metadata;
    bytes32 builder;
}
```

`ORDER_TYPE_STRING`:
```
Order(uint256 salt,address maker,address signer,uint256 tokenId,uint256 makerAmount,uint256 takerAmount,uint8 side,uint8 signatureType,uint256 timestamp,bytes32 metadata,bytes32 builder)
```

EIP-712 domain:
```
name:              "Polymarket CTF Exchange"
version:           "2"
chainId:           137
verifyingContract: 0xE111180000d2663C0091e4f400237545B87B996B   (negRisk uses 0xe222…0F59)
```

### 6.2 The ERC-7739 `TypedDataSign` wrap

Instead of signing the raw `Order` struct directly, you sign a **wrapper** struct whose `contents` field is the order and whose remaining fields name the inner ERC-1271 verifier domain (the deposit wallet's `DepositWallet` v1).

`TypedDataSign` struct:
```
TypedDataSign(Order contents,string name,string version,uint256 chainId,address verifyingContract,bytes32 salt)
Order(...above...)
```

Signed value:
```typescript
{
  contents: {
    salt:          order.salt,
    maker:         order.maker,
    signer:        order.signer,
    tokenId:       order.tokenId,
    makerAmount:   order.makerAmount,
    takerAmount:   order.takerAmount,
    side:          order.side,
    signatureType: 3,
    timestamp:     order.timestamp,
    metadata:      order.metadata,
    builder:       order.builder,
  },
  name:              'DepositWallet',
  version:           '1',
  chainId:           137,
  verifyingContract: <depositWalletAddress>,
  salt:              '0x' + '0'.repeat(64),
}
```

Outer EIP-712 domain stays on **CTF Exchange V2** (`Polymarket CTF Exchange` v2 @ exchange address). The inner DepositWallet domain rides as plain wrapper fields.

### 6.3 The final signature payload

The signature submitted to the CLOB is **not** just the inner ECDSA result. The ERC-7739 envelope appends the inner domain separator + contents hash + the typed-data type string + its length, so the wallet's `isValidSignature` can re-derive everything:

```
signature = innerSig (65 bytes)
         || appDomainSeparator (32 bytes)   // keccak256(EIP712Domain ‖ "Polymarket CTF Exchange" ‖ "2" ‖ chainId ‖ exchangeV2)
         || contentsHash (32 bytes)         // keccak256(ORDER_TYPE_HASH ‖ all 11 order fields)
         || contentsType (raw bytes)        // utf-8 bytes of ORDER_TYPE_STRING above
         || uint16_BE(contentsType.length)
```

Concretely:

```typescript
import { ethers } from 'ethers';

const ORDER_TYPE_STRING =
  'Order(uint256 salt,address maker,address signer,uint256 tokenId,' +
  'uint256 makerAmount,uint256 takerAmount,uint8 side,uint8 signatureType,' +
  'uint256 timestamp,bytes32 metadata,bytes32 builder)';
const ORDER_TYPE_HASH    = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(ORDER_TYPE_STRING));
const DOMAIN_TYPE_HASH   = ethers.utils.keccak256(
  ethers.utils.toUtf8Bytes('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
);
const NAME_HASH    = ethers.utils.keccak256(ethers.utils.toUtf8Bytes('Polymarket CTF Exchange'));
const VERSION_HASH = ethers.utils.keccak256(ethers.utils.toUtf8Bytes('2'));

const appDomainSeparator = ethers.utils.keccak256(
  ethers.utils.defaultAbiCoder.encode(
    ['bytes32', 'bytes32', 'bytes32', 'uint256', 'address'],
    [DOMAIN_TYPE_HASH, NAME_HASH, VERSION_HASH, 137, EXCHANGE_V2_ADDRESS],
  ),
);

const contentsHash = ethers.utils.keccak256(
  ethers.utils.defaultAbiCoder.encode(
    ['bytes32','uint256','address','address','uint256','uint256','uint256','uint8','uint8','uint256','bytes32','bytes32'],
    [
      ORDER_TYPE_HASH, order.salt, order.maker, order.signer,
      order.tokenId, order.makerAmount, order.takerAmount,
      order.side, order.signatureType, order.timestamp,
      order.metadata, order.builder,
    ],
  ),
);

// innerSig = ownerEoa._signTypedData(ctfDomain, { TypedDataSign, Order }, value)  (§6.2)
const contentsTypeHex = ethers.utils.hexlify(ethers.utils.toUtf8Bytes(ORDER_TYPE_STRING));
const lenHex          = ORDER_TYPE_STRING.length.toString(16).padStart(4, '0');

const signature =
  '0x' + innerSig.slice(2) + appDomainSeparator.slice(2)
       + contentsHash.slice(2) + contentsTypeHex.slice(2) + lenHex;
```

### 6.4 Recommended: use `clob-client-v2 ≥ 1.0.6`

`@polymarket/clob-client-v2@1.0.6` produces this exact signature when you pass `signatureType: 3`. Use the high-level API and skip the manual encoding:

```typescript
import { ClobClient, SignatureTypeV2, Side, OrderType } from '@polymarket/clob-client-v2';
import { Wallet } from '@ethersproject/wallet';

const ownerEoa = new Wallet(process.env.DEPOSIT_OWNER_PRIVATE_KEY!);

// 1. Derive API key with a default (no sigType) client — the deposit wallet's
//    isValidSignature is unused here; the EOA owns the key.
const tempClient = new ClobClient({
  host: process.env.CLOB_HOST!,
  chain: 137,
  signer: ownerEoa,
});
const creds = await tempClient.createOrDeriveApiKey();

// 2. Typed client — orders go out with sigType=3, ERC-7739 wrap.
const client = new ClobClient({
  host: process.env.CLOB_HOST!,
  chain: 137,
  signer: ownerEoa,                                 // EOA signs
  creds,
  signatureType: SignatureTypeV2.POLY_1271,         // 3
  funderAddress: process.env.DEPOSIT_WALLET!,       // maker = signer = deposit wallet
});
```

Important: **`maker` and `signer` on the order must both equal the deposit wallet address**, never the EOA. The SDK does this automatically when `funderAddress` is set and `signatureType=3`.

If you're stuck on an earlier `clob-client-v2` (e.g. `1.0.0` for compatibility reasons), vendor `ExchangeOrderBuilderV2` from `1.0.6/src/order-utils/exchangeOrderBuilderV2.ts` and use the SDK only for HTTP/HMAC. Submit your locally-signed order through `client.postOrder(signedOrder, OrderType.FAK)` — the SDK's `isV2Order(order)` detects V2 by struct shape and routes the payload through L2 auth correctly.

### 6.5 CLOB API key derivation

The `createOrDeriveApiKey` call uses the **EOA** signer with no `signatureType`. The returned `{ key, secret, passphrase }` is the L2 HMAC credential — cache it on disk to skip the ~250 ms per-process re-derivation.

After deployment + funding, optionally bump the CLOB cache (signature_type=3 path is L2-authed, so the SDK does this automatically on first authenticated call):

```typescript
await client.updateBalanceAllowance({ asset_type: 'COLLATERAL' });
```

The unauthenticated public `GET /balance-allowance/update?asset_type=COLLATERAL&signature_type=3` is rejected with `401`; ignore it.

---

## 7. Funding

A deposit wallet trades from collateral held **at the wallet address**, not at the owner EOA. pUSD at the EOA is invisible to the CLOB and will not count as buying power.

### 7.1 If you receive pUSD directly

Send pUSD to `DEPOSIT_WALLET`. That's it.

### 7.2 If you receive native USDC (most CEX withdrawals)

Polygon-native USDC (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`) is **not** Polymarket's collateral. Bridge it via `https://bridge.polymarket.com`, which performs `USDC → USDC.e → pUSD` in one atomic backend operation (~30 s typical):

1. `POST bridge.polymarket.com/deposit { address: DEPOSIT_WALLET }` → returns an ephemeral "intent" EVM address.
2. Transfer the native USDC from the deposit wallet to that intent address. Because the deposit wallet is a smart contract, you do this via `executeDepositWalletBatch` (one ERC-20 `transfer` call). Relayer pays gas; the wallet pays USDC.
3. Poll `GET bridge.polymarket.com/status/<intent>` until both transactions' `status === 'COMPLETED'`. The bridge backend credits pUSD to the deposit wallet.
4. Verify with `pUSD.balanceOf(DEPOSIT_WALLET)`.

```typescript
const erc20 = new ethers.utils.Interface(['function transfer(address,uint256) returns (bool)']);
const transferData = erc20.encodeFunctionData('transfer', [intent, parseUnits(amount.toString(), 6)]);

const resp = await relayer.executeDepositWalletBatch(
  [{ target: USDC_NATIVE, value: '0', data: transferData }],
  depositWallet,
  Math.floor(Date.now()/1000 + 3600).toString(),
);
await resp.wait();
// Then poll bridge /status/{intent} until both legs COMPLETED.
```

Bridge constraints: minimum $2; intent addresses are single-use; the bridge handles all collateral wrapping internally.

### 7.3 If you receive USDC.e

Wrap directly via `CollateralOnramp` (`0x93070a847efEf7F70739046A929D47a521F5B8ee`):

```typescript
// As one WALLET batch:
//   1. USDC.e.approve(onramp, amount)
//   2. onramp.convert(USDC.e, amount)
```

After conversion, pUSD lands at the deposit wallet.

---

## 8. Trading on V2 CLOB

Once the wallet is deployed, approved, and funded, you trade exactly like any V2 wallet — see [CLOB-V2-TRANSACTIONS.md](./CLOB-V2-TRANSACTIONS.md) for the protocol-level reference. The deposit-wallet-specific differences:

| Concern | Deposit-wallet behaviour |
|---|---|
| `signer` field on order | Deposit wallet address |
| `maker` field on order | Deposit wallet address |
| `signatureType` | `3` (POLY_1271) |
| Signature shape | ERC-7739 `TypedDataSign` wrap (§6) |
| Order placement | `client.postOrder(signedOrder, orderType)` — same endpoint, same HMAC L2 auth |
| Sponsoring gas | Not your concern — Polymarket matches on-chain at no cost to the maker |

### 8.1 Market BUY (FOK / FAK)

```typescript
const response = await client.createAndPostMarketOrder(
  {
    tokenID: '7546…',
    side: Side.BUY,
    amount: 5.00,        // pUSD to spend
    price: 0.99,         // max-price ceiling (slippage guard)
  },
  { tickSize: '0.01', negRisk: false },
  OrderType.FOK,         // or OrderType.FAK
);

// Response:
//   sharesReceived = parseFloat(response.takingAmount)
//   pUsdSpent      = parseFloat(response.makingAmount)
//   avgPrice       = pUsdSpent / sharesReceived
```

### 8.2 Market SELL (FOK / FAK)

```typescript
const response = await client.createAndPostMarketOrder(
  {
    tokenID: '7546…',
    side: Side.SELL,
    amount: 10,          // SHARES to sell (not pUSD)
    price: 0.01,         // min-price floor
  },
  { tickSize: '0.01', negRisk: false },
  OrderType.FAK,
);

// Response (fields are flipped vs BUY):
//   pUsdReceived = parseFloat(response.takingAmount)
//   sharesSold   = parseFloat(response.makingAmount)
//   avgPrice     = pUsdReceived / sharesSold
```

### 8.3 GTC / GTD limit orders

```typescript
await client.createAndPostOrder(
  { tokenID, side: Side.BUY, price: 0.43, size: 10 },
  { tickSize: '0.01', negRisk: false },
  OrderType.GTC,
);
```

`size` is **shares** for both sides. GTC/GTD orders are cancelled by the CLOB if no `postHeartbeat` arrives for 10+ seconds — send one every ~5 s while you have resting orders.

### 8.4 Order-status semantics (V2, all wallet types)

| `status` | Emitted by | Meaning | Action |
|---|---|---|---|
| `matched` | FOK, FAK, GTC, GTD | Filled now; `takingAmount` / `makingAmount` populated | Persist fill |
| `live` | GTC, GTD | Resting on book | Watch WSS for fills |
| `delayed` | **FAK only** (and sports-market FOK empirically) | On Polymarket's books, matching delay in flight | Write `pending_orders` row keyed by `orderID`; **never retry** |
| `unmatched` | FAK only | Window closed, no fill | Genuine liquidity failure — safe to retry |
| hard reject | any | `errorMsg` populated, no usable `orderID` | Fix root cause (allowance / balance / signature) |

`status='delayed'` is the single biggest correctness trap: it returns `success: true` and a valid `orderID` but empty fill amounts. Code that decides retry-vs-persist based on amount presence will misclassify it as failure and double-submit. **Always read `status` first.**

### 8.5 Production market-BUY retry pattern (FOK×N → FAK×1)

From the companion CLOB-V2-TRANSACTIONS.md (§"Hybrid Market Order Retry Pattern"):

```typescript
const MAX_FOK = 4, MAX_FAK = 1, RETRY_MS = 3000;
const TOTAL   = MAX_FOK + MAX_FAK;

for (let attempt = 1; attempt <= TOTAL; attempt++) {
  const orderType = attempt <= MAX_FOK ? OrderType.FOK : OrderType.FAK;
  const r = await client.createAndPostMarketOrder(/* … */, /* … */, orderType);

  if (r.status === 'matched' && r.takingAmount) break;                  // success
  if (r.status === 'delayed' && r.orderID)      { writePending(r); break; } // in flight
  const isLiquidityMiss = r.orderID && r.status !== 'matched' && r.status !== 'delayed';
  if (!isLiquidityMiss) break;                                          // hard reject
  if (attempt < TOTAL) await new Promise(r => setTimeout(r, RETRY_MS));
}
```

Invariant: **`orderId` present + `status` ∉ {`matched`, `delayed`} == liquidity miss.** Covers FAK's spec-correct `'unmatched'`, FOK's in-practice empty status, and degenerate `'matched'` returns with zero fill amounts. Do not raise `MAX_FAK` above 1 — a delayed-FAK on attempt 2 duplicates an in-flight order from attempt 1.

### 8.6 Order constraints (recap)

| Constraint | Value |
|---|---|
| Min shares per order | 5 |
| Min order cost | $1.00 pUSD |
| Price range | $0.01 — $0.99 |
| Position-detection lag | 3–10 s after fill before `data-api.polymarket.com/positions` reflects it |

---

## 9. On-chain split / merge / redeem via relayer

Split, merge, and redeem are on-chain operations that move pUSD ↔ CTF tokens. With a deposit wallet, you **don't** send these as Safe.execTransaction or direct EOA tx — you submit them as `DepositWalletCall` items through the relayer.

### 9.1 Why always go through the adapter, never raw CTF

Direct calls on `CTF` (`0x4D97…6045`) work on-chain, but the CLOB balance cache won't see the resulting tokens (`getBalanceAllowance` returns `0`), blocking every subsequent sell. **Always** route through:

- `CtfCollateralAdapter` (`0xAdA100Db00Ca00073811820692005400218FcE1f`) for binary markets.
- `NegRiskAdapter` (`0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296`) for 3+-outcome markets.

The adapters internally handle pUSD ↔ USDC.e wrapping and emit the events the CLOB indexer watches.

### 9.2 Split (pUSD → outcome tokens)

```typescript
const adapterIface = new ethers.utils.Interface([
  'function splitPosition(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint256[] partition, uint256 amount)',
]);

const data = adapterIface.encodeFunctionData('splitPosition', [
  PUSD,
  '0x' + '0'.repeat(64),  // parent collection = root
  conditionId,
  [1, 2],                 // binary partition
  ethers.utils.parseUnits(amount.toString(), 6),  // raw 6-decimal pUSD
]);

await relayer.executeDepositWalletBatch(
  [{ target: CTF_ADAPTER, value: '0', data }],
  depositWallet,
  Math.floor(Date.now()/1000 + 3600).toString(),
).then(r => r.wait());
```

NegRisk variant (single condition, no parent / partition args):

```typescript
const negRiskIface = new ethers.utils.Interface([
  'function splitPosition(bytes32 conditionId, uint256 amount)',
]);
const data = negRiskIface.encodeFunctionData('splitPosition', [conditionId, amountRaw]);
// target: NEGRISK_ADAPTER
```

### 9.3 Merge (outcome tokens → pUSD)

Same call shape, different function selector:

```typescript
const adapterIface = new ethers.utils.Interface([
  'function mergePositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint256[] partition, uint256 amount)',
]);
const data = adapterIface.encodeFunctionData('mergePositions', [
  PUSD, '0x'+'0'.repeat(64), conditionId, [1, 2], amountRaw,
]);
// target: CTF_ADAPTER
```

Merge requires holding ≥ `amount` of **each** outcome token. The market must not be settled — use `redeem` after resolution.

### 9.4 Redeem (winning tokens → pUSD after settlement)

```typescript
const adapterIface = new ethers.utils.Interface([
  'function redeemPositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint256[] indexSets)',
]);
const data = adapterIface.encodeFunctionData('redeemPositions', [
  PUSD, '0x'+'0'.repeat(64), conditionId, [1, 2],   // both outcomes; winning side pays out, losing burns for $0
]);
// target: CTF_ADAPTER
```

NegRisk variant — amounts array, no parent / collateral args:

```typescript
const negRiskIface = new ethers.utils.Interface([
  'function redeemPositions(bytes32 conditionId, uint256[] amounts)',
]);
// amounts[outcomeIndex - 1] = shares*1e6, rest = 0
const data = negRiskIface.encodeFunctionData('redeemPositions', [conditionId, amounts]);
```

### 9.5 Pre-flight (replaces `callStatic` simulation)

With Safe-based wallets, `safe.callStatic.execTransaction(…)` simulates the call locally before paying gas and surfaces revert reasons like missing approvals or already-redeemed positions. The relayer **does not support local simulation** of `executeDepositWalletBatch` — the only way to find out the call would fail is to submit it and read the failure state.

Replace `callStatic` with cheap RPC reads before submission:

| Op | Pre-check (read calls, free) |
|---|---|
| `split(amount)` | `pUSD.balanceOf(wallet) >= amount` **and** `pUSD.allowance(wallet, adapter) >= amount` |
| `merge(shares)` | `CTF.balanceOf(wallet, yesPositionId) >= shares` **and** `CTF.balanceOf(wallet, noPositionId) >= shares` |
| `redeem(conditionId)` | Gamma `market.closed === true`, and **at least one** `CTF.balanceOf(wallet, positionId) > 0` |

If any pre-check fails, abort without submitting the batch. You'll waste a relayer round-trip on legitimate races (e.g. a position got redeemed in another process between the check and the submit) — accept that as the cost of not paying gas to learn the revert.

### 9.6 Already-redeemed detection

The adapter reverts with `subtraction overflow` when you try to redeem a position that's already been redeemed (CTF burns the tokens; the next call sees zero balance and underflows). Treat any relayer batch failure whose error message contains `subtraction overflow` as **`ALREADY_REDEEMED`** — non-retryable, but not a "real" error either. Use a per-process in-memory cache (`redeemedThisSession: Set<conditionId>`) to absorb the lag between on-chain redemption and Polymarket's positions API reflecting the change (can be minutes).

---

## 10. Operational concerns

### 10.1 Cache CLOB API credentials

`createOrDeriveApiKey` makes 1–3 HTTP requests and EIP-712 signatures. Cache the result on disk and reuse for the process lifetime; only re-derive on credential rotation:

```typescript
// On first run:
const creds = await tempClient.createOrDeriveApiKey();
fs.writeFileSync('.clob-creds.json', JSON.stringify(creds));

// On subsequent runs:
const creds = JSON.parse(fs.readFileSync('.clob-creds.json', 'utf8'));
```

### 10.2 Provider attachment

`@polymarket/builder-abstract-signer` wraps your ethers Wallet, and its `EthersSigner` constructor explicitly throws `signer is missing provider` if the Wallet has no Provider attached. Always construct as:

```typescript
const owner = new Wallet(privateKey, provider);   // NOT just `new Wallet(privateKey)`
```

This bites once during the first `deriveDepositWalletAddress()` call and is otherwise silent.

### 10.3 Idempotency / re-runnable init script

A correct deposit-wallet bootstrap script can re-run any number of times without side effects:

1. Compute predicted address.
2. `getDeployed(addr, 'WALLET')` — if true, skip deployment.
3. Always submit the approvals batch (no-op for already-MAX allowances and already-true setApprovalForAll flags; the relayer will still mine a tx — track the cost or guard with a `pUSD.allowance` read if it matters).
4. Always print the address so operators can copy it to their `.env`.

### 10.4 Two distinct keys

It is operationally cleaner to use a dedicated `DEPOSIT_OWNER_PRIVATE_KEY` rather than reuse a Safe owner key or a generic trading key — the deposit wallet binds permanently to the EOA, so loss/compromise of the key is loss/compromise of the wallet. Separate keys per environment (mainnet/test) prevent cross-contamination.

### 10.5 Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `signer is missing provider` | Wallet constructed without provider | `new Wallet(pk, provider)` (§10.2) |
| `WALLET-CREATE STATE_INVALID` | Builder creds wrong or expired | Verify `BUILDER_*` env from Polymarket dashboard |
| Orders rejected as INVALID_SIGNATURE | Vendored signer mismatch or wrong domain | Confirm clob-client-v2 ≥ 1.0.6 OR re-verify wrap byte-exact (§6.3); confirm `maker == signer == DEPOSIT_WALLET` |
| `balance: 0` even after funding | pUSD sent to the **owner EOA** instead of the deposit wallet | Send to `DEPOSIT_WALLET` |
| `allowance: 0` after fresh deploy | Approvals not run | Run §5.2 once |
| `subtraction overflow` on redeem | Already redeemed | Treat as `ALREADY_REDEEMED`, skip silently |
| Bridge wrap doesn't credit pUSD | USDC sent from owner EOA instead of via deposit-wallet batch | Send from deposit wallet (§7.2) |
| `CLOB Client] request error` | Spurious internal log from clob-client-v2 — safe to suppress | `console.error` wrapper that filters this line |
| status=`delayed` retried & duplicated | Code gated on amount presence, not `status` | Read `status` first (§8.4) |

### 10.6 Web "Enable Trading" button conflict

Polymarket's web app's "Enable Trading" button runs the same `WALLET-CREATE` + approvals flow your script does. If your script deployed the wallet first, the web app sees an already-provisioned wallet that **it didn't provision**, and surfaces `An unknown error occurred`. This is cosmetic — the wallet is already trading-ready. Operators shouldn't click "Enable Trading" on API-provisioned wallets.

---

## 11. Minimal end-to-end skeleton

A 100-line bootstrap that takes you from "I have an owner key" to "I'm ready to trade":

```typescript
import 'dotenv/config';
import { ethers } from 'ethers';
import { Wallet } from '@ethersproject/wallet';
import {
  RelayClient,
  DepositWalletCall,
  RelayerTransactionState,
} from '@polymarket/builder-relayer-client';
import { BuilderConfig } from '@polymarket/builder-signing-sdk';
import { ClobClient, SignatureTypeV2 } from '@polymarket/clob-client-v2';

const PUSD = '0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB';
const CTF  = '0x4D97DCd97eC945f40cF65F87097ACe5EA0476045';
const SPENDERS = [
  ['CTF Exchange V2',     '0xE111180000d2663C0091e4f400237545B87B996B'],
  ['NegRisk Exchange V2', '0xe2222d279d744050d28e00520010520000310F59'],
  ['NegRiskAdapter',      '0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296'],
] as const;

async function main() {
  const provider = new ethers.providers.JsonRpcProvider(process.env.RPC_URL);
  const owner    = new Wallet(process.env.DEPOSIT_OWNER_PRIVATE_KEY!, provider);

  const builderConfig = new BuilderConfig({
    localBuilderCreds: {
      key: process.env.BUILDER_API_KEY!,
      secret: process.env.BUILDER_SECRET!,
      passphrase: process.env.BUILDER_PASS_PHRASE!,
    },
  });
  const relayer = new RelayClient(process.env.RELAYER_URL!, 137, owner as any, builderConfig);

  // 1. Derive + deploy
  const addr = await relayer.deriveDepositWalletAddress();
  console.log('Deposit wallet:', addr);
  if (!(await relayer.getDeployed(addr, 'WALLET'))) {
    const r = await relayer.deployDepositWallet();
    const c = await r.wait();
    if (!c || (c.state !== RelayerTransactionState.STATE_MINED && c.state !== RelayerTransactionState.STATE_CONFIRMED)) {
      throw new Error(`deploy failed: ${c?.state}`);
    }
    console.log('Deployed:', c.transactionHash);
  }

  // 2. Approve
  const MAX     = ethers.constants.MaxUint256.toString();
  const erc20   = new ethers.utils.Interface(['function approve(address,uint256) returns (bool)']);
  const erc1155 = new ethers.utils.Interface(['function setApprovalForAll(address,bool)']);
  const calls: DepositWalletCall[] = SPENDERS.flatMap(([_, sp]) => [
    { target: PUSD, value: '0', data: erc20.encodeFunctionData('approve', [sp, MAX]) },
    { target: CTF,  value: '0', data: erc1155.encodeFunctionData('setApprovalForAll', [sp, true]) },
  ]);
  const deadline = Math.floor(Date.now()/1000 + 3600).toString();
  const ar = await relayer.executeDepositWalletBatch(calls, addr, deadline);
  await ar.wait();
  console.log('Approvals applied.');

  // 3. CLOB client (sigType=3)
  const temp  = new ClobClient({ host: process.env.CLOB_HOST!, chain: 137, signer: owner });
  const creds = await temp.createOrDeriveApiKey();
  const clob  = new ClobClient({
    host: process.env.CLOB_HOST!,
    chain: 137,
    signer: owner,
    creds,
    signatureType: SignatureTypeV2.POLY_1271,
    funderAddress: addr,
  });

  // 4. Fund externally with pUSD (or bridge USDC, see §7), then trade:
  //
  //   await clob.createAndPostMarketOrder(
  //     { tokenID, side: Side.BUY, amount: 5, price: 0.99 },
  //     { tickSize: '0.01', negRisk: false },
  //     OrderType.FAK,
  //   );
  //
  //   await relayer.executeDepositWalletBatch(
  //     [{ target: '0xAdA1…cE1f', value: '0', data: redeemCalldata }],
  //     addr,
  //     deadline,
  //   ).then(r => r.wait());
}

main().catch(e => { console.error(e); process.exit(1); });
```

---

## 12. Reference

### 12.1 Polygon mainnet (chain 137) addresses

| Contract | Address | Source |
|---|---|---|
| Deposit Wallet Factory | `0x00000000000Fb5C9ADea0298D729A0CB3823Cc07` | builder-relayer-client config |
| Deposit Wallet Implementation | `0x58CA52ebe0DadfdF531Cde7062e76746de4Db1eB` | builder-relayer-client config |
| pUSD | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` | clob-client-v2 config |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | clob-client-v2 config |
| CTF Exchange V2 | `0xE111180000d2663C0091e4f400237545B87B996B` | clob-client-v2 config |
| NegRisk Exchange V2 | `0xe2222d279d744050d28e00520010520000310F59` | clob-client-v2 config |
| NegRiskAdapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | clob-client-v2 config |
| CtfCollateralAdapter | `0xAdA100Db00Ca00073811820692005400218FcE1f` | Polymarket docs |
| NegRiskCtfCollateralAdapter | `0xadA2005600Dec949baf300f4C6120000bDB6eAab` | Polymarket docs |
| CollateralOnramp (USDC.e → pUSD) | `0x93070a847efEf7F70739046A929D47a521F5B8ee` | Polymarket docs |
| CollateralOfframp (pUSD → USDC.e) | `0x2957922Eb93258b93368531d39fAcCA3B4dC5854` | Polymarket docs |
| USDC.e (bridged) | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | — |
| USDC native (Circle) | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` | — |

### 12.2 Endpoints

| Use | URL |
|---|---|
| Relayer | `https://relayer-v2.polymarket.com` |
| CLOB | `https://clob.polymarket.com` |
| User WSS (own-fill reconciliation) | `wss://ws-subscriptions-clob.polymarket.com/ws/user` |
| Market WSS (orderbook) | `wss://ws-subscriptions-clob.polymarket.com/ws/market` |
| Gamma API (market metadata) | `https://gamma-api.polymarket.com` |
| Data API (positions, activity) | `https://data-api.polymarket.com` |
| Bridge | `https://bridge.polymarket.com` |

### 12.3 EIP-712 domains

```
DepositWallet (used for WALLET batch signatures)
  name              "DepositWallet"
  version           "1"
  chainId           137
  verifyingContract <depositWalletAddress>

Polymarket CTF Exchange V2 (used for CLOB orders)
  name              "Polymarket CTF Exchange"
  version           "2"
  chainId           137
  verifyingContract 0xE111180000d2663C0091e4f400237545B87B996B   // or NegRisk V2 = 0xe222…0F59
```

### 12.4 Companion documents

- [CLOB-V2-TRANSACTIONS.md](./CLOB-V2-TRANSACTIONS.md) — protocol-level V2 reference (order types, status semantics, retry pattern, response shapes, index-set math, fee model). Wallet-model-agnostic.
- Polymarket official: `https://docs.polymarket.com/trading/deposit-wallets`

### 12.5 Glossary

| Term | Definition |
|---|---|
| Deposit wallet | Per-user ERC-1967 proxy deployed by Polymarket's factory; the canonical "managed smart wallet" for new API users |
| Owner EOA | The externally-owned account whose signature authorizes the deposit wallet's actions via ERC-1271 |
| Relayer | Polymarket's server-side service that submits on-chain txs on the deposit wallet's behalf, paying gas |
| Builder creds | HMAC API key / secret / passphrase that authenticates relayer requests |
| WALLET batch | A bundle of contract calls executed by the deposit wallet in a single relayer tx |
| POLY_1271 | `signatureType=3`: the deposit wallet validates orders via on-chain ERC-1271 |
| ERC-7739 `TypedDataSign` | The wrapper format the deposit wallet's `isValidSignature` expects — inner EIP-712 sig + app-domain separator + contents hash + type string + length |
| Intent address | Single-use bridge-deposit address returned by `bridge.polymarket.com/deposit` |
| pUSD | Polymarket's collateral token; required for all V2 trading |

---

*Last updated: 2026-05-13.*
