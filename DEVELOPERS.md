# BasedPad — developer notes

Everything a bot, aggregator or explorer needs to read BasedPad markets from chain. Contract *interfaces* and public addresses only.

## Chains and factories

| Chain | Chain id | Module | Factory | Pool | Quote assets |
| --- | --- | --- | --- | --- | --- |
| Base | 8453 | Global V9 · RWA/stock pairs | `0x2508192A8C5f473496b07d47232d379916FCa23f` | Uniswap v4, 1% | official Coinbase tokenized stocks (B20) |
| Base | 8453 | Global V9 · crypto pairs | `0x0d45358E1e371b05E6469CcB5a5863720a6120D3` | Uniswap v4, 1% | curated Base crypto quotes (cbBTC, cbHYPE, cbZEC, …) |
| Arc | 5042 | Arc Markets V2 | `0x3bB227BF7af8d91f44E72489C62B322a40a6081b` | Uniswap v3, 1% | USDC + curated Arc memes (COOL, ARCHITECTS, FATCAT, ARCAT, BEANCAT, TOLLY, …) |
| Robinhood Chain | 4663 | Robinhood Markets V1 | `0xA56912625a298fc55D34346eFa017E5f3F793A96` | Uniswap v3, 1% | WETH, USDG + curated memes (PONS, CASHCAT, INDEX, DELTA, AI, WALLET, PIPEDOG, …) |
| Robinhood Chain | 4663 | BON/DEX Lineage V1 | `0x8cDCC99F5DEde621F95866500d50a657FFd4BAD9` · router `0x3d4F8C42Ec66CD33a57ba362eF4a7cdA45909eA1` | Uniswap v3, 1% | $BONS or a previous lineage market |
| HyperEVM | 999 | HyperEVM V1 | `0x3E8e9dbB100f0f8B61904485EE9CBa9A0370D0c4` | HyperSwap v3, 1% | WHYPE |

Live addresses, deployment blocks and the curated quote lists are always available from `GET https://basedpad.fun/api/v1`.

## Launch events

```
// Base V9 (RWA + crypto)
event StockDirectTokenLaunched(uint256 indexed id, address indexed token, address indexed creator, bytes32 poolId,
  address quoteToken, address quoteFeed, string quoteTicker, string name, string symbol, string metadataURI,
  uint160 sqrtPriceX96, uint256 startMcapQuote, uint256 referenceTargetQuote)

// Arc V2 · Robinhood V1 · BON/DEX lineage · HyperEVM (two events per launch on Arc/Robinhood/lineage)
event HyperTokenLaunched(uint256 indexed id, address indexed token, address indexed creator, address pool, address feeRecipient,
  address rewardVault, string name, string symbol, string metadataURI, bool tokenIsToken0, int24 tickLower, int24 tickUpper,
  uint128 liquidity, uint160 sqrtPriceX96, uint256 startMcapQuote, uint256 gradTargetQuote)
event ArcQuoteMarketLaunched(uint256 indexed id, address indexed token, address indexed quoteToken, address pool,
  string quoteTicker, uint256 quoteUsdE18, uint256 startMcapQuote, uint256 gradTargetQuote)
event LineageLinked(address indexed token, address indexed parent, uint256 generation, uint256 swapsFromWeth)   // lineage only
```

`ArcQuoteMarketLaunched` carries the market's real quote; `HyperTokenLaunched` alone means the module's default quote (WHYPE on HyperEVM).

## Reading a market

All factories expose the same shape:

```
launchCount() → uint256                     tokenAt(i) → address
getLaunch(token) / launchInfo(token)        → creator, pool, feeRecipient, tokenIsToken0, ticks, liquidity, createdAt, accrued fees
marketState(token)                          → sqrtPriceX96, tick, spot price in quote, market cap in quote, pool balances, graduation
quoteUsdE18(quoteToken)                     → (twapUsdE18, spotUsdE18)   // Arc / Robinhood / lineage: 30-min TWAP reference
```

USD price of a token = spot price in its quote × the quote's USD reference. On Base the reference comes from the V9 registry (Pyth / Chainlink / Coinbase stock feeds), on Arc from a QUOTE/USDC pool TWAP, on Robinhood from a two-hop TWAP (QUOTE→WETH→USDG), on lineage markets from the product of TWAPs along the path to $BONS.

## Economics (every module)

- Fixed supply 1,000,000,000, seeded single-sided into the pool at **$3,000 FDV**; graduation target **$9,000** in the pool.
- The only trading cost is the pool's **1% LP fee**. No BasedPad swap fee.
- LP fees accrue in both pool tokens and split **90% creator / 10% platform**; `collectFees(token)` is permissionless.
- Launch guard on Arc / Robinhood / lineage: a quote whose spot is more than 10% off its 30-minute TWAP cannot be used to open a market.

## Trading

- Base V9: `buy` / `sell` in the quote asset on the factory; ETH buys are routed by the BasedPad OmniBuyer.
- Arc: `buyWithUsdc` / `sellForUsdc` (routed through the quote's USDC pool) or `buy` / `sell` in the quote.
- Robinhood: `buyWithEth` (payable) / `sellForEth`, or `buy` / `sell` in the quote.
- BON/DEX lineage: `BonDexRouter.buyWithEth(token, minOut, recipient, deadline)` (payable) and `sellForEth` execute the whole path in one transaction. `pathOf(token)` on the lineage factory lists the pools in order; every hop after WETH→PONS is a BasedPad pool.

Pools are standard Uniswap v3 / v4 pools — aggregators can also route them directly.

## BON/DEX lineage

Markets on the lineage factory are quoted in `$BONS` (root) or in another lineage market. A generation-N token is bought through N+2 pools (`WETH → PONS → BONS → G1 → … → GN`); nothing is sold back on the way, so every trade is pressure on every ancestor. Each market can carry an assigned Robinhood tokenized stock with its Chainlink total-return feed: `stockOf(token)` and `stockUsdE18(token)`.

## Metadata

`metadataURI` → `ipfs://` JSON: `{ name, symbol, description, image (ipfs://), website, x, telegram }`. Fetch through `https://basedpad.fun/api/token-metadata?uri=ipfs://…` and artwork through `https://basedpad.fun/api/ipfs-image/{cid}` (both cached, immutable).

## ABIs

`abi/` in this repository: `arc-markets-v2.abi.json`, `robinhood-markets-v1.abi.json`, `bondex-lineage-v1.abi.json`, `bondex-router.abi.json`, and `base-v9.signatures.txt`.
