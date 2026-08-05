# Polymarket On-Chain Operations

> CTF merge, redeem, allowances, and USDC balance on Polygon.

## Table of Contents

- [Overview](#overview)
- [CTF Merge](#ctf-merge)
- [Redeem](#redeem)
- [Allowances](#allowances)
- [USDC Balance](#usdc-balance)
- [Proxy Wallets vs EOA](#proxy-wallets-vs-eoa)

---

## Overview

Polymarket runs on **Polygon PoS**. Outcome tokens follow the **Conditional Token Framework (CTF)** standard. All on-chain operations interact with CTF contracts on Polygon.

> ⚠️ **CLOB V2 cutover: 2026-04-28 11:00 UTC.** The exchange contracts and the
> collateral token both changed. There is no V1 compatibility. Addresses below
> re-verified against <https://docs.polymarket.com/resources/contracts> on
> **2026-08-06** — this page previously published the retired V1 exchanges as
> current, and code that subscribed to them collected nothing while reporting
> no error.

### Core Contract Addresses (Polygon Mainnet, Chain ID 137)

| Contract | Address |
|----------|---------|
| **CTF Exchange (V2)** | `0xE111180000d2663C0091e4f400237545B87B996B` |
| **Neg Risk CTF Exchange (V2)** | `0xe2222d279d744050d28e00520010520000310F59` |
| **Conditional Tokens (CTF)** | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |
| **Gnosis Safe Factory** | `0xaacfeea03eb1561c4e67d661e40682bd20e3541b` |
| **Polymarket Proxy Factory** | `0xaB45c5A4B0c941a2F231C04C3f49182e1A254052` |
| **Deposit Wallet Factory** | `0x00000000000Fb5C9ADea0298D729A0CB3823Cc07` |
| **UMA Adapter** | `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74` |
| **UMA Optimistic Oracle** | `0xCB1822859cEF82Cd2Eb4E6276C7916e692995130` |

### Collateral Contracts

Collateral is **pUSD**, an ERC-20 on Polygon backed by USDC. It replaced USDC.e
at the V2 cutover.

| Contract | Address |
|----------|---------|
| **pUSD — CollateralToken (proxy)** | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` |
| pUSD — CollateralToken (impl) | `0x6bBCef9f7ef3B6C592c99e0f206a0DE94Ad0925f` |
| **CollateralOnramp** (`wrap()`) | `0x93070a847efEf7F70739046A929D47a521F5B8ee` |
| CollateralOfframp | `0x2957922Eb93258b93368531d39fAcCA3B4dC5854` |
| CtfCollateralAdapter | `0xAdA100Db00Ca00073811820692005400218FcE1f` |
| NegRiskCtfCollateralAdapter | `0xadA2005600Dec949baf300f4C6120000bDB6eAab` |

The Polymarket UI wraps USDC → pUSD automatically. **API-only traders must call
`wrap()` on the CollateralOnramp themselves.**

### Retired — do NOT use

| Contract | Address | Status |
|----------|---------|--------|
| CTF Exchange (V1) | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` | dead since 2026-04-28 |
| Neg Risk CTF Exchange (V1) | `0xC5d563A36AE78145C45a50134d48A1215220f80a` | dead since 2026-04-28 |
| Neg Risk Adapter (CLOB v1) | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | deprecated 2026-07-14; redeems closed 2026-07-17 |
| USDC.e (Bridged USDC) | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | still a real token, no longer the collateral |

They are listed only so that a reader who finds them in old code recognises
them. Subscribing to a retired exchange yields **zero events and no error**,
which is indistinguishable from a quiet market — if you are watching logs, watch
the V2 addresses and alarm on prolonged silence.

---

## CTF Merge

> **The core of market-making profit.**

Merging combines equal quantities of ALL outcome tokens back into the collateral (USDC).

For a binary market:
```
1 UP token + 1 DN token → $1.00 USDC
```

### Why Merge?

If you bought:
- 100 UP tokens at $0.47 each = $47.00
- 100 DN tokens at $0.50 each = $50.00
- Total cost: $97.00

Merging 100 pairs:
- 100 × $1.00 = $100.00
- **Profit: $3.00** (regardless of which outcome wins)

### How to Merge

Verified against the official docs 2026-08-06. These are **gasless** wallet
transactions executed through the relayer, and they return a handle you wait on
rather than a receipt:

```python
import os
from polymarket import SecureClient

with SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],
) as client:
    # amount: base-unit int, or "max" to merge the smaller of the YES/NO balances
    transaction = client.merge_positions(
        condition_id=market.condition_id,
        amount="max",
    )
    outcome = transaction.wait()
```

`AsyncSecureClient` is identical with `async with` / `await`.

### Merge Economics

| Scenario | UP Cost | DN Cost | Total | Merge Return | Profit |
|----------|---------|---------|-------|-------------|--------|
| Tight spread | $0.49 | $0.50 | $0.99 | $1.00 | $0.01/pair |
| Normal spread | $0.47 | $0.50 | $0.97 | $1.00 | $0.03/pair |
| Wide spread | $0.45 | $0.50 | $0.95 | $1.00 | $0.05/pair |

The merge edge is **sigma minus 1.0** when buying at the ask: `merge_edge = 1.0 - (up_ask + dn_ask)` … wait, that's negative. Actually the merge edge comes from buying at bid-side or getting filled on resting orders below the ask.

**Real merge edge = 1.0 - (cost_per_up + cost_per_dn)**

---

## Redeem

After a market resolves, winning tokens can be redeemed for $1.00 each.

```python
# Redeems the wallet's balances for BOTH outcomes — no amount parameter
transaction = client.redeem_positions(
    condition_id=market.condition_id,
)
outcome = transaction.wait()
```

The inverse of merge is `split_position(condition_id=…, amount=1_000_000)`
(base units), which turns collateral into a full set of outcome tokens.

### Important Notes

- Only the winning token pays out; losing tokens are worth $0.00
- Executed **gasless** through the relayer, so no POL is required
- You must wait until `umaResolutionStatus == "resolved"` before redeeming

---

## Allowances

Before trading, you must approve the **V2** CTF Exchange
(`0xE1111800…B996B`, or the Neg Risk one for multi-outcome markets) to spend
your **pUSD**. An allowance granted to a V1 exchange address does nothing.

The unified SDK's wallet workflows handle approvals as part of its gasless
transaction flow; there is no verified standalone `set_allowance()` call in the
current SDK, so approve directly against the token contract if you are managing
allowances yourself.

### Check Existing Allowance

If your orders are being rejected with allowance errors, verify:
1. The approval is set for the **V2** exchange contract, on **pUSD**
2. The approval amount is sufficient for your order size
3. You're using the correct wallet (EOA vs proxy)
4. You actually hold pUSD, not raw USDC — see the balance section below

---

## Collateral Balance (pUSD)

Since the V2 cutover the tradable balance is **pUSD**, not USDC.e. Holding USDC
is not the same as being funded: it has to be wrapped first.

```solidity
// API-only traders wrap USDC -> pUSD themselves.
// CollateralOnramp: 0x93070a847efEf7F70739046A929D47a521F5B8ee
CollateralOnramp.wrap()
```

(A balance getter exists on the SDK clients but its exact name was not verified
here — read it from the client rather than copying a name from this page.)

### Balance Considerations

- Both USDC and pUSD use **6 decimals** (1 = 1,000,000 units)
- Keep some POL for gas (minimal, ~0.01 per transaction)
- Open orders **lock** collateral — available balance = total - locked
- The UI wraps for you; an API-only bot that never calls `wrap()` will look
  funded on-chain and still fail every order

---

## Proxy Wallets vs EOA

### EOA (Externally Owned Account)

- Your direct Ethereum wallet (MetaMask, etc.)
- Use `signature_type=0` in orders
- You control the private key directly
- Simpler setup

### Proxy Wallet (Gnosis Safe)

- Created by Polymarket when you deposit via their UI
- Use `signature_type=2` in orders
- The proxy wallet holds your funds
- Your EOA is the owner/signer

### How to Know Which You're Using

The unified SDK takes the wallet address directly and works out the signature
type for you — the address on polymarket.com is the proxy:

```python
import os
from polymarket import SecureClient

client = SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],   # proxy wallet address
)
```

The CLOB-only client still exposes `signature_type` explicitly, as V1 did:

```python
from py_clob_client_v2 import ClobClient

# deposited via the Polymarket UI -> proxy wallet
client = ClobClient(host=host, chain_id=137, key=PK, signature_type=2, funder=PROXY)

# sent collateral straight to your own EOA -> EOA
client = ClobClient(host=host, chain_id=137, key=PK, signature_type=0)
```

### Common Issue

Using the wrong `signature_type` causes orders to be rejected silently or with cryptic errors. If your orders aren't going through, try switching between 0 and 2.

---

## Negative Risk Markets

Multi-outcome events trade against the **Neg Risk CTF Exchange (V2)**
(`0xe2222d…310F59`) for capital-efficient trading:

- A **No share** in any market can be converted into **1 Yes share in every other market** in the event
- Orders on neg risk markets must specify `negRisk: true` / `neg_risk: True`
- Watching on-chain fills for these markets means subscribing to **both**
  exchanges — the neg-risk one settles a whole class of markets, and omitting it
  looks like a delivery gap rather than a missing subscription
- The **CLOB v1 Neg Risk Adapter** (`0xd91E80…35296`) was deprecated 2026-07-14
  and its redeem window closed 2026-07-17. Collateral adapters now live at
  `CtfCollateralAdapter` / `NegRiskCtfCollateralAdapter` (see the table above)

### Identifying Neg Risk

The Gamma API includes `negRisk` boolean on events/markets. For **augmented neg risk** (new outcomes can be added):
```json
{"enableNegRisk": true, "negRiskAugmented": true}
```

**Rule**: Only trade named outcomes in augmented neg risk — ignore placeholders until clarified.

---

## Gasless Transactions (Builder Program)

Through the **Relayer Client**, Builder Program members can sponsor gas for users:

- Wallet deployment, token approvals, CTF operations (split/merge/redeem), transfers
- Users only need collateral (pUSD) — no POL required
- Relayer endpoint: `https://relayer-v2.polymarket.com/`
- Requires Builder API credentials for authentication

---

## Subgraph (On-Chain GraphQL)

Polymarket subgraphs (hosted by Goldsky) provide indexed on-chain data:

| Subgraph | Description |
|----------|-------------|
| Positions | User token balances |
| Orders | Order book and trade events |
| Activity | Splits, merges, redemptions, neg risk conversions |
| Open Interest | Per-market and global OI |
| PNL | User position P&L |

Base endpoint: `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/`

Source: [github.com/Polymarket/polymarket-subgraph](https://github.com/Polymarket/polymarket-subgraph)

---

*See also: [CLOB API](polymarket-api.md) · [Market Structure](polymarket-markets.md) · [Metrics](metrics-calculations.md)*
