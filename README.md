# Paid x402 APIs on Base — $0.005–$0.01 per call, USDC, no key, no signup

Three production HTTP 402 endpoints you can call directly from an agent. No API key, no
account, no email. Pay per request in USDC on Base mainnet and get JSON back.

| Endpoint | What it answers | Price |
|---|---|---|
| [`tokensafe.x402api.workers.dev`](https://tokensafe.x402api.workers.dev) | Is this Base ERC-20 token safe to interact with? | $0.005 |
| [`railaudit.x402api.workers.dev`](https://railaudit.x402api.workers.dev) | Can an x402 payment actually settle right now? | $0.010 |
| [`slippage.x402api.workers.dev`](https://slippage.x402api.workers.dev) | What will this swap really cost me in this pool? | $0.005 |

Network: `eip155:8453` (Base). Asset: USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`.
Payout address: `0xc9d0ab76b5e9cbe3da320347ccdaa111e515e6d0`.

## Why these are cheap and honest

- Every data point is read **live from chain** (ERC-20 balances, `getReserves()`, token
  decimals, facilitator liveness). Nothing is cached or guessed.
- When a source does not answer, the field is **`null`** and the verdict says so. There is
  no path that turns "unknown" into "safe".
- One HTTP 402 challenge per call, `exact` scheme, 60s timeout.

## Machine-readable manifests (free, unauthenticated, 200)

Crawlers and agents should not have to guess how to pay us. All of these are served
**free, HTTP 200, no payment**, while only the actual data routes are paywalled:

| Path | Purpose |
|---|---|
| `/.well-known/x402` | x402 discovery manifest (canonical) |
| `/agents.json` | Agent card |
| `/openapi.json` | OpenAPI 3.1 spec with inline `x402` price + scheme |
| `/llms.txt` | Plain-text summary for LLM crawlers |
| `/health` | Liveness, payTo, price, network |

Verified live 2026-10-06: `agent402.tools` reports all three origins as
`routable: true`, `health: 1`, `toolCount: 3`, `discoveryPath: /.well-known/x402`.

## Call one in 20 seconds

Any x402 client works. With `curl`:

```bash
# 1. Ask what it costs (no payment, no key)
curl -s https://tokensafe.x402api.workers.dev/health
# {"ok":true,"catalogued":true,"payTo":"0xc9d0...","price_usd":0.005,"network":"eip155:8453"}

# 2. Call it — the 402 body carries the exact payment requirements
curl -s 'https://tokensafe.x402api.workers.dev/api/probe?address=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'
```

You get back a standard x402 v2 challenge:

```json
{"x402Version":2,"error":"Payment required",
 "resource":{"url":"https://railaudit.x402api.workers.dev/audit", ...},
 "accepts":[{"scheme":"exact","network":"eip155:8453","amount":"10000",
             "asset":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
             "payTo":"0xc9d0ab76b5e9cbe3da320347ccdaa111e515e6d0",
             "maxTimeoutSeconds":60}]}
```

Sign with your facilitator and resend the same request with an `X-PAYMENT` header. The
challenge is also base64 in the `payment-required` response header.

## The three endpoints

### 1. Token safety verdict — $0.005

`GET /api/probe?address=0x…` (40 hex chars)

Live USDC and ETH balances for the address, token metadata (name, symbol, price, source),
an explicit risk list, and a verdict of `no_flags` or `review_recommended`.

### 2. x402 rail audit — $0.010

`GET /audit`

Probes the facilitators that can settle x402 payments right now — liveness, latency,
advertised kinds — plus Base RPC reachability and current block. Verdict: `usable`,
`no_facilitator_available`, or `chain_unreachable`. Useful before you pay anyone.

### 3. Swap slippage simulation — $0.005

`GET /slip?pool=0x…&amount=<integer token0 base units>`

Reads `getReserves()` and token decimals on chain, then returns amount out, execution
ratio, and slippage percent measured against the pool's own spot rate, plus reserves and
decimals. Verdict: `acceptable` or `expensive_route`.

## Listings

- agent402.tools index — all three listed, `routable: true`
  (`POST https://agent402.tools/api/index/register`, free, no account)
