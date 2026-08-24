# 54ch10 for agents

Pre-interact JSON briefs for **addresses**, **tokens**, and **URLs** (phishing / scam / wallet / URL-reputation heuristics). Brand: **54ch10** (not “SACHIO”). Analytics-only tooling — not financial, legal, or investment advice.

Base URL: `https://54ch10.uk`

## Why call this before a wallet interact

Agents that can sign, approve, bridge, or open a dapp URL should brief the counterparty **before** spending gas or privileges:

1. Resolve the target (`address` / `token` / `url`).
2. `GET /v1/brief` → `score` (0–100, higher = more concerning), `band`, `flags[]`, `summary`, `sources[]`.
3. Gate the next tool call on policy (e.g. refuse `critical`, require human confirm on `elevated`).
4. Log `cached_at` + `sources[].status` so operators can audit stub vs live signals.

Live free signals (no paid keys): OpenPhish community feed for `type=url` (host match, KV-cached); public Ethereum `eth_getCode` for EVM `address` / `token` (contract vs EOA vs EIP-7702). Marked `sources[].status=live` only when the upstream responds.

Do **not** treat a low score as clearance. Scores are imperfect heuristics and can be wrong or incomplete.

## Auth / payment modes

| Mode | How | Limits |
|------|-----|--------|
| **x402 (canonical)** | Unauthenticated `GET /v1/brief` → HTTP **402** + `PAYMENT-REQUIRED`; retry with `PAYMENT-SIGNATURE` (USDC) | $0.01 / brief (preferred: Base USDC) |
| Stripe key | `Authorization: Bearer 54k_…` on `/v1/brief` | Solo 500 / Pro 5k / Team 25k per month |
| Demo free | `X-54ch10-Free: 1` **or** `GET /v1/brief/free` | 20 briefs / UTC day / IP |

Indexers (CDP Bazaar, PayAI discovery, x402scan, agent402) must probe **`GET /v1/brief`** — it always returns **402** without auth/payment. Do **not** use `/v1/brief/free` for discovery.

Keys are minted after Stripe Checkout; claim via landing `?paid=1&session_id=cs_…`. Invalid `54k_` keys return `401`.

### x402 (agents — no Stripe checkout)

Discovery: `GET https://54ch10.uk/.well-known/x402` (also `/.well-known/x402.json`).

Flow:

1. `GET /v1/brief?type=…&q=…` → HTTP **402** with `PAYMENT-REQUIRED` (base64 x402 v2 JSON, compliant `extensions.bazaar`).
2. Client signs USDC payment (exact scheme) and retries with `PAYMENT-SIGNATURE`.
3. Worker verifies+settles via PayAI facilitator (`https://facilitator.payai.network`) and returns the brief + `PAYMENT-RESPONSE`.

Receive wallets (54ch10 brand):

- **EVM / Base USDC:** `0xF9eb0Caa13B78f92E2850bf5961eB9736354aA3d`
- **Solana USDC:** `EcGUGSda9VNzYgUmYA8mTg88MhNYeyA5tgaCgwzxfFNs`

Preferred rail (x402 docs): **USDC on Base** (`eip155:8453`). Facilitator sponsors settlement gas — receive wallet does **not** need pre-funding to accept payments. Optional later: send a little ETH/SOL only if self-settling.

Compatible clients: `@x402/fetch` `wrapFetchWithPayment`, Cloudflare Agents `withX402Client`, any x402 v2 wallet agent.

Smoke (expect 402):

```bash
curl -si 'https://54ch10.uk/v1/brief?type=url&q=https://example.com' | head -40
```

Demo free (expect 200 under quota):

```bash
curl -si 'https://54ch10.uk/v1/brief?type=url&q=https://example.com' \
  -H 'X-54ch10-Free: 1' | head -40
# or: curl -si 'https://54ch10.uk/v1/brief/free?type=url&q=https://example.com'
```

## Endpoint

```
GET /v1/brief?type=address|token|url&q=<query>     # 402 unless Bearer / PAYMENT-SIGNATURE / X-54ch10-Free
GET /v1/brief/free?type=address|token|url&q=<query>  # human/demo free path
```

Response fields (success): `ok`, `type`, `q`, `score`, `band` (`clear` \| `caution` \| `elevated` \| `critical` \| `unknown`), `flags`, `summary`, `sources`, `cached_at`, `cache_hit`, `tier` (`free` \| `pro` \| `x402`), `disclaimer`, `meta`.

Cache TTL ≈ 15 minutes. CORS: `*`.

## Example curl

```bash
# Canonical — expect HTTP 402 + PAYMENT-REQUIRED (indexers)
curl -si 'https://54ch10.uk/v1/brief?type=url&q=https://example.com/connect-wallet' | head -40

# Demo free — URL before connect-wallet click
curl -sS 'https://54ch10.uk/v1/brief/free?type=url&q=https://example.com/connect-wallet'

# Demo free via header
curl -sS 'https://54ch10.uk/v1/brief?type=address&q=0x0000000000000000000000000000000000000000' \
  -H 'X-54ch10-Free: 1'

# Stripe paid key
curl -sS 'https://54ch10.uk/v1/brief?type=token&q=ETH' \
  -H 'Authorization: Bearer 54k_YOUR_KEY'
```

Agent-shaped policy sketch:

```text
brief = GET /v1/brief?type=…&q=…   # pay via x402 client, or use Bearer / free header for demos
if brief.band in {elevated, critical}: halt_or_escalate(brief)
else: proceed_with_interact()
```

## Pricing (Stripe Payment Links — humans)

| Plan | Price | Included |
|------|-------|----------|
| Solo | $29/mo | 500 briefs / mo, API key |
| Pro | $99/mo | 5k briefs + watchlists |
| Team | $299/mo | 25k briefs + seats / keys |

- Solo: https://buy.stripe.com/dRmcN69Gu3yI6Pd6bu28800
- Pro: https://buy.stripe.com/aFa28s8Cq8T2c9x57q28802
- Team: https://buy.stripe.com/3cI6oI9Gu4CMgpNbvO28801

Also: `GET /health`, `GET /LEGAL_DISCLAIMER.txt`, `GET /.well-known/x402`, `GET /openapi.json`.

## Disclaimer

Informational tooling only. Not financial, investment, legal, or tax advice. Not an offer or recommendation to buy, sell, hold, stake, bridge, swap, or interact with any asset, protocol, or URL. No custody, trading, or token promotion. You are solely responsible for decisions. Full text: https://54ch10.uk/LEGAL_DISCLAIMER.txt
