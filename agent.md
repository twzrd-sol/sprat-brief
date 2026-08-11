# Agent brief: SPRAT / TWZRD

Read **https://raw.githubusercontent.com/twzrd-sol/sprat-brief/main/sprat.json** first (JSON).
Use **https://raw.githubusercontent.com/twzrd-sol/sprat-brief/main/llms.txt** for prose.

## One-liner
Become the default supplier of Cloudflare Agents onto Solana via x402 (preflight + facilitator).

## Truthfulness
This brief is static. It is **not** connected to private GitHub `wzrd-final`.
Cross-check production truth with https://intel.twzrd.xyz/llms.txt and the real monorepo.

## Decision (2026-08-11) — multi-agent lock

**Pick: brief + hold.**

| Do | Don't |
|----|--------|
| Ship Workers + **x402-hono** + TWZRD facilitator (Solana USDC) | Engineer Agents MCP SVM this week |
| Keep preflight as pre-spend trust gate | Re-probe Signal B as the main product |
| Passive watch Signal A (Wallets) + Signal B (MCP SVM) | Claim MCP Solana works or is a moat |
| Attach future upstream work to **#1863**, mind **#1989** deprecation | Reopen pure-Solana **#596** (NOT_PLANNED) |

**Live path (proven):** `x402-hono` + `network: "solana"` + `https://intel.twzrd.xyz`  
**MCP path:** EVM-only until CF registers SVM (Foundation `@x402/svm` already exists — gap is registration, not protocol).

Optional later: **read-only spike** (go/no-go) before any companion package or upstream PR. Prefer companion first if go.

## Do
- Flood Solana volume on Workers + x402-hono + intel facilitator
- Point operators at monorepo example README + `CF_AGENTS_SOLANA.md`

## Don't
- Speculative CF Wallets scopes ahead of stable API
- Speculative Agents MCP SVM engineering without written go
- Treat P5 impression baseline as live funnel unless you verify prod DB
