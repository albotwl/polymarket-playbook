Migration to V2

---

# 📘 Polymarket CLOB V2 — Production Documentation (Post-Cutover)

## Overview

Polymarket завершил полный переход на **CLOB V2** 28 апреля 2026 года.

* ✅ V2 — единственная активная версия
* ❌ V1 полностью отключён (нет backward compatibility)
* 🌐 Production endpoint:

  ```
  https://clob.polymarket.com
  ```
* ⚠️ Все интеграции обязаны использовать V2 SDK и новый формат ордеров

---

## Core Changes Summary

| Компонент  | V1               | V2                   |
| ---------- | ---------------- | -------------------- |
| SDK        | `py-clob-client` | `py-clob-client-v2`  |
| Collateral | USDC.e           | **pUSD**             |
| Nonce      | required         | ❌ removed            |
| Fees       | в ордере         | dynamic (match-time) |
| Builder    | HMAC headers     | `builderCode`        |
| Signing    | EIP-712 v1       | **EIP-712 v2**       |

---

## SDK Migration

> ⚠️ Проверено по PyPI и официальной документации **2026-08-06**. Ниже была
> ошибка: форма `ClobClient({...})` с ключом `chain` — это **TypeScript**-клиент.
> Python-клиент V2 сохранил именованные аргументы и `chain_id`.

### Какой пакет ставить

| Пакет | Что это | Версия | Python |
|-------|---------|--------|--------|
| `polymarket-client` | **официальный unified SDK** ([Polymarket/py-sdk](https://github.com/Polymarket/py-sdk)) — данные, авторизация, торговля, builder, gasless-кошелёк. Именно его использует quickstart в документации | 0.3.0 (2026-08-04) | ≥3.11 |
| `py-clob-client-v2` | клиент **только CLOB V2** | 1.1.0 (2026-07-17) | ≥3.9.10 |
| ~~`py-clob-client`~~ | **архивирован, нерабочий** — «should not be used for new or existing integrations» | — | — |

`polymarket-client` ещё до 1.0: его README предупреждает, что минорные релизы
на ветке 0.x могут ломать совместимость. Фиксируйте версию.

### Installation

```bash
pip install polymarket-client        # unified SDK (по умолчанию)
pip install py-clob-client-v2        # если нужен только CLOB
```

---

### Client Initialization

#### V1 — нерабочий

```python
from py_clob_client.client import ClobClient          # архивирован
ClobClient(host, chain_id=137, key=PK, signature_type=2, funder=PROXY)
```

#### V2 — unified SDK

```python
import os
from polymarket import PublicClient, SecureClient

with PublicClient() as client:                        # без ключей, только чтение
    ...

with SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],
) as client:                                          # деривация L2-кредов внутри
    ...
```

#### V2 — CLOB-only

```python
import os
from py_clob_client_v2 import ClobClient

client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"])
creds = client.create_or_derive_api_key()             # именно api_key, не api_creds
client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"], creds=creds)
```

### Changes

* `py_clob_client` → `polymarket` или `py_clob_client_v2`
* `create_or_derive_api_creds()` → `create_or_derive_api_key()` (CLOB-клиент)
* клиенты unified SDK — контекст-менеджеры (`with` / `async with`)
* `chainId` → `chain` и объектный конфиг — **только TypeScript**; в Python
  остались именованные аргументы и `chain_id`

---

## Order Model (Critical Changes)

### Removed Fields

* `nonce`
* `feeRateBps`
* `taker`

### Added Fields

* `timestamp` (SDK-managed)
* `metadata`
* `builder`

---

### Minimal Order Example

```python
order = {
    "tokenID": "...",
    "price": 0.55,
    "size": 100,
    "side": Side.BUY,
    "expiration": 1714000000
}
```

---

## Fee Model

### Old

* Fee задавался вручную (`feeRateBps`)

### New

* Fee рассчитывается протоколом при матчингe:

```
fee = C × feeRate × p × (1 - p)
```

### Key Rules

* Makers: **0 fee**
* Takers: платят комиссию
* ❌ Manual fee calculation → удалить

---

## Collateral: pUSD

### Migration

* USDC.e заменён на **pUSD**
* pUSD = ERC-20 backed by USDC

### API Users

Необходимо:

* учитывать баланс в pUSD
* использовать wrap:

```solidity
CollateralOnramp.wrap()
```

---

## Builder Program

### Removed

* `POLY_BUILDER_*` headers
* builder-signing-sdk

### New

```python
builderCode = "0x..."
```

Можно:

* передавать в каждом ордере
* задать в client config

---

## EIP-712 Signing

### Changes

```diff
version: "1" → "2"
verifyingContract → updated
```

### Important

* применяется только к Exchange domain
* API auth остаётся прежним

---

## Nonce System Removal

* Nonce больше не используется
* Уникальность:

```
timestamp (milliseconds)
```

---

## API Changes

### Order Payload

```diff
- nonce
- feeRateBps
- taker
+ timestamp
+ metadata
+ builder
```

---

### Headers

```diff
- POLY_BUILDER_*
+ builder в payload
```

---

### Auth

Без изменений:

* API key
* signature
* passphrase

---

## Contracts

* Chain: Polygon (137)
* Использовать только V2 addresses

👉 Источник:
[https://docs.polymarket.com/resources/contracts](https://docs.polymarket.com/resources/contracts)

---

## Bot Migration Impact

### Remove

* feeRateBps logic
* nonce tracking
* manual fee calculations

---

### Add

* pUSD balance tracking
* `userUSDCBalance` (для market orders)

---

### Update

* order schema
* signing domain
* contract addresses

---

### Verify

* execution logic
* risk management (fees теперь динамические)

---

## Known Failure Cases

### ❌ Invalid fee rate

```json
"invalid fee rate (0)"
```

Причина:

* использование V1 поля `feeRateBps`

---

### ❌ Broken Orders

Причины:

* используется nonce
* старый SDK
* старые contract addresses

---

## Minimal Working Example

Unified SDK — `price` и `size` **строки**:

```python
import os
from polymarket import SecureClient

with SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],
) as client:
    response = client.place_limit_order(
        token_id=token_id,
        side="BUY",
        price="0.52",
        size="100",
    )
    if not response.ok:
        raise RuntimeError(f"{response.code}: {response.message}")
```

CLOB-only — числа и явный `order_type`:

```python
from py_clob_client_v2 import OrderArgs, OrderType, PartialCreateOrderOptions, Side

resp = client.create_and_post_order(
    order_args=OrderArgs(token_id=token_id, price=0.52, side=Side.BUY, size=100),
    options=PartialCreateOrderOptions(tick_size="0.01"),
    order_type=OrderType.GTC,
)
```

---

## Migration Checklist

### SDK

* [ ] установить `polymarket-client` (или `py-clob-client-v2`, если нужен только CLOB)
* [ ] удалить `py-clob-client` — он архивирован и нерабочий
* [ ] заменить `create_or_derive_api_creds()` → `create_or_derive_api_key()`

### Orders

* [ ] удалить `nonce`
* [ ] удалить `feeRateBps`
* [ ] удалить `taker`

### Signing

* [ ] обновить EIP-712 (version = 2)
* [ ] обновить verifyingContract

### Collateral

* [ ] перейти на pUSD
* [ ] добавить wrap flow (если API-only)

### Fees

* [ ] удалить manual fee logic

### Builder

* [ ] заменить на `builderCode`

### Contracts

* [ ] обновить все адреса

---

## Final Notes

* V2 — это **полный rewrite**, не просто апдейт
* Любая логика V1 ломает ордера
* Основные изменения:

  * fee → dynamic
  * nonce → removed
  * collateral → pUSD

---

## Status

✅ Ready for production
❌ V1 compatibility отсутствует

-