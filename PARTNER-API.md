# BasedPad public API — for DEX aggregators, explorers and bots

No authentication, CORS open, JSON only, **multi-chain**. Base URL: `https://basedpad.fun`

| Endpoint | What it returns |
| --- | --- |
| `GET /api/v1` | Launchpad identity: name, logo URLs, supported chains, factory addresses per chain, launch event signatures, economics, endpoint list. |
| `GET /api/v1/tokens` | Every token launched through BasedPad, newest first. Query params: `chain=all\|base\|arc\|robinhood\|hyperevm` (default `all`), `page` (default 1), `limit` (default 100, max 200), `status=active\|graduated`, `module=<module id>`, `since=<ISO-8601>` for incremental sync. |
| `GET /api/v1/tokens/{address}` | One token by contract address (if the same address exists on several chains, `{ address, chains: [...] }`). |
| `GET /api/ipfs-image/{cid}` | Cached, immutable token artwork (long-lived `Cache-Control`). |

Responses carry `Access-Control-Allow-Origin: *`; `OPTIONS` is answered for preflight. List responses are cached ~10 s.

## Token object

```json
{
  "address": "0xa87f…f36A",
  "chainId": 4663,
  "chain": "robinhood",
  "module": "robinhood-v1",
  "moduleLabel": "Robinhood · ETH / USDG / meme pair",
  "name": "BONS",
  "symbol": "BONS",
  "decimals": 18,
  "totalSupply": "1000000000",
  "tokenStandard": "ERC20",
  "icon": "https://basedpad.fun/api/ipfs-image/bafy…",
  "iconIpfs": "ipfs://bafy…",
  "metadataUri": "ipfs://bafy…",
  "description": "…",
  "website": "https://…",
  "x": "https://x.com/…",
  "telegram": "https://t.me/…",
  "discord": null,
  "creator": { "wallet": "0xFF88…9c07" },
  "quote": { "symbol": "PONS", "address": "0x39dB…4571", "decimals": 18, "usdPeg": null },
  "pool": { "id": null, "address": "0xd6B3…3D7C", "manager": null, "fee": 10000, "dex": "uniswap-v3" },
  "launch": { "block": "55010100", "time": "2026-09-05T10:26:59.000Z", "factory": "0xA569…3A96" },
  "status": "ACTIVE",
  "market": {
    "priceUsd": 0.0000236,
    "marketCapUsd": 23619.92,
    "progress": 0.586,
    "volume24hUsd": 127891.49,
    "totalVolumeUsd": 412330.10,
    "swaps": 1023,
    "holders": 179,
    "traders": 463,
    "updatedAt": "2026-09-06T07:40:12.000Z"
  },
  "lineage": { "generation": 0, "swapsFromWeth": 2, "parent": "0x39dB…4571", "parentSymbol": "PONS", "root": "0xa87f…f36A" },
  "urls": { "market": "https://basedpad.fun/robinhood/token/0xa87f…f36A", "explorer": "https://robinhoodchain.blockscout.com/token/0xa87f…f36A" }
}
```

Notes

- `x`, `telegram`, `discord`, `website`, `description` are `null` when the creator did not provide them; `icon` is `null` until the metadata has been resolved.
- `progress` is the graduation progress (market cap ÷ the module's graduation target, capped at 1). Every module opens at **$3,000 FDV** on a fixed 1B supply and graduates at **$9,000**.
- `market.priceUsd` / `marketCapUsd` are `null` for a few legacy Base markets (pre-V6) that have no live on-chain valuation; use `marketState()` on the factory instead.
- Base markets use Uniswap **v4**: `pool.id` is the V4 pool id and `pool.manager` the PoolManager; Arc / Robinhood use Uniswap **v3** and HyperEVM HyperSwap v3, where `pool.address` is the pool contract.
- `lineage` is present only for BON/DEX lineage markets on Robinhood Chain (see DEVELOPERS.md).

## Incremental sync

```
GET /api/v1/tokens?chain=all&since=2026-09-06T00:00:00Z&limit=200
```

`since` filters on `launch.time`. Poll every minute; new launches appear within ~10 s of the on-chain event.

## Onchain alternative

Every module emits a launch event you can index directly — addresses, signatures and pool types per chain are in `GET /api/v1` (`factories`, `events`) and in DEVELOPERS.md. `metadataURI` is an `ipfs://` JSON document with `name`, `symbol`, `description`, `image` (ipfs://), `website`, `x`, `telegram`.

## Launchpad logo

- Square: `https://basedpad.fun/brand/launchpad-512.png` (also `launchpad-256.png`)
- Rounded badge: `https://basedpad.fun/brand/launchpad-badge-256.png` (also `launchpad-badge-64.png`)
- Banner: `https://basedpad.fun/brand/banner-1600x533.jpg`
- Site: https://basedpad.fun · X: https://x.com/basedpadfun
