# Path B external integration runbook

**Goal:** install the buyer-side x402 trust gate, run a no-spend proof, and produce a
transcript that counts as an external adoption signal — not TWZRD dogfood.

**Package:** [`twzrd-x402-gate`](https://www.npmjs.com/package/twzrd-x402-gate) (npm,
MIT, zero deps). Current: `0.8.14`.

This is a technical runbook, not a pitch. Every command and field below is copied from
the package's own README and the monorepo's operator-acceptance doc, not paraphrased.

---

## Outreach sequence (72h SLA)

| Window | Target | Role |
|---|---|---|
| 0–24h | **Vicky** | P0 — live install on her buyer host |
| 24–48h | **Nick** | P1 if Vicky is blocked or delayed |
| 48–72h | **Lucas** | P2, closes the window |

Success = **one named external host + one artifact** (section 5), not a star count or
a download number. Move to the next name in sequence rather than wait indefinitely on
one — the point is to get an artifact inside 72h, not to be polite about ordering.

---

## 0. What this actually does

`twzrd-x402-gate` hooks into an x402 client's `onBeforePaymentCreation` — after the
client has picked which payment requirement to pay, **before** it builds a payment
payload or asks a wallet to sign. It runs a free preflight (`ReadinessCard`) plus a
free merchant wash-check against the seller (`payTo`), and can abort the payment
before any signature is produced. Chain-neutral envelope; Solana-deep reputation
(Base/EVM currently return explicit `unknown`, not a score).

It does **not** sit in the settlement path, does not hold funds, and does not require
an account.

---

## 1. Install (pick your client stack)

```bash
# official @x402/* path
npm install twzrd-x402-gate@0.8.14 @x402/core @x402/fetch @x402/svm

# or PayAI x402-solana (stock client, primary Solana seat via beforePayment)
npm install twzrd-x402-gate@0.8.14 x402-solana@2.1.0
```

> **ESM only.** `require()` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED` — set
> `"type": "module"` in `package.json`, or use `.mjs`, or an ESM-aware bundler.

### Official `@x402/core` client

```typescript
import { x402Client } from "@x402/core/client";
import { installTwzrdAutoGate } from "twzrd-x402-gate";

const client = new x402Client();
installTwzrdAutoGate(client, { refuseWashFlagged: true });
// then register schemes + wrapFetchWithPayment as usual
```

### PayAI `x402-solana`

```typescript
import { createX402Client } from "x402-solana";
import { createTwzrdBeforePaymentHook } from "twzrd-x402-gate";

const client = createX402Client({
  wallet,
  network: "solana",
  beforePayment: createTwzrdBeforePaymentHook({ refuseWashFlagged: true }),
});
```

**Compatibility note:** proxied clients (AgentCash `.fetch`, ClawRouter `:8402`) handle
402 internally and return 200 — the guard must wrap the client's *input*, never its
*output*, or it never sees the 402 to act on. `installTwzrdAutoGate` enforces this
composition order by construction.

---

## 2. Prove it blocks — zero spend, zero wallet

Two proofs, and they are **not the same thing** — mixing them up is the easiest way
to think you've produced an adoption signal when you've only produced a mechanism
demo. Both are useful; only one is designed to correlate.

### 2a. Live block proof (real network call, no attribution)

```bash
npm install twzrd-x402-gate@0.8.14
npx twzrd-gate-eval-refuse
```

This makes a **real** `POST /v1/intel/preflight` call against a known TWZRD-owned
block fixture, checks the response is `decision=block, can_spend=false`, then proves
your client's pre-sign hook aborts before any signer is invoked (`signer_invocation_count
= 0`, `payment_retry_count = 0`). It costs nothing and touches the live server — but
by design it carries no attribution headers and, per its own source comment, does
**not** increment the Live Board's `gate_evals` counter. It's a mechanism proof you
can paste into a DM or issue, not an adoption signal.

### 2b. Attributed proof (what actually correlates)

There is no public zero-install command for this — it requires wiring the gate into
your own client with `attribution` set (section 3 below) and making it run for real,
even once, even against a synthetic/cheap target. That's the step the rest of this
doc is about.

---

## 3. Instrumentation — what makes a run *external*, not dogfood

Add `attribution` so your preflight calls carry correlation headers:

```typescript
installTwzrdAutoGate((guarded) => wrapFetchWithPayment(guarded, wallet), {
  refuseWashFlagged: true,
  attribution: {
    integration: "your-project-name",       // stable, yours
    runId: crypto.randomUUID(),             // fresh per run — echo it back to us
  },
});
```

This stamps **only the TWZRD preflight request** (never the paid `/v1/intel/trust`
call, never your resource fetch) with:

```
X-TWZRD-Integration: <integration>
X-TWZRD-Run-Id:      <runId>
X-TWZRD-Client:      twzrd-x402-gate/<version>
```

No PII, secret, wallet address, or payload is added. Both fields must be set together
or nothing is stamped.

### The bar for "this counts" (all of the following)

1. **You own the environment** — your machine/CI, not ours.
2. Your `integration` id is non-internal, and you post the same `runId` yourself —
   in your own issue, PR, or a message to us. We do not accept a `runId` we generated
   on your behalf as evidence.
3. **Server-side join** (for production adoption, not just the local harness): the
   same `X-TWZRD-Integration` / `X-TWZRD-Run-Id` observed on a real preflight request
   from non-internal lineage.
4. Prefer evidence that you installed via npm and ran the actual hook path, not just
   this repo's own unit tests.

### What does **not** count

- npm download counts
- Free preflight HTTP hits with no attribution and no operator-posted `runId`
- A `runId` we generated, with no transcript you posted yourself
- The local harness alone, run inside our own repo/CI
- A successful payment with no gate decision in the path

---

## 4. Evidence checklist (screen-share ready)

Before calling a run a proof, confirm every line:

```text
[ ] Host is NOT owned/operated by TWZRD
[ ] Operator / org is named
[ ] Gate is wired into the pay path (installTwzrdAutoGate / createTwzrdBeforePaymentHook
    on a real client) — a curl to /v1/intel/preflight alone does not satisfy this
[ ] Gate evaluation timestamp is BEFORE any signTransaction / signer invocation
[ ] On block: signer_invocation_count = 0, payment_retry_count = 0, usdc_spent = 0
[ ] Raw gate/preflight JSON response saved
[ ] Payer pubkey recorded
[ ] Target URL / seller payTo recorded
[ ] attribution.integration + attribution.runId were set (section 3) and the operator
    posts the same runId themselves
```

## 5. Artifact template

Fill this in and post it yourself (issue, PR comment, DM) — we do not accept a
transcript we generated on your behalf as evidence (see section 3, bar #2).

```markdown
### TWZRD Path B external proof

* Operator / org: …
* Host runtime: … (not TWZRD-operated)
* Integration: twzrd-x402-gate@0.8.14 + [x402-solana | @x402/core]
* Target endpoint / payTo: …
* Payer pubkey: …
* Timestamp (UTC): …
* attribution.integration / attribution.runId: …

#### Gate decision (raw response)
\`\`\`json
{ "decision": "block", "trust_score": 30.0, "can_spend": false, "...": "..." }
\`\`\`

#### Outcome
* Signer invoked: NO (block) / YES (allow — include tx signature)
* signer_invocation_count: 0
* Mainnet tx: N/A (block) / <signature> (allow)
```

Real `decision` values from the live API are lowercase: `allow`, `warn`, `block` — not
`ALLOW`/`BLOCK`. Copy the response verbatim rather than retyping it.

---

## 6. Report back

Paste the transcript JSON (or the accurate one-liner below, filled in with your own
`preflight_id`/`runId`) wherever you're talking to us — a GitHub issue, a DM, a PR
comment. That's the whole loop:

> Preflight returned `decision=<X>` on `<seller>` (`preflight_id <N>`, integration
> `<yours>`, run `<uuid>`). Gate `approved=<bool>` reason `<reason>`. Signer
> invocations: 0. No USDC spent unless the gate allowed it.

If you'd rather not install anything yet: `npx twzrd-preflight <wallet-or-url>` is the
free, no-auth CLI check — same trust data, no gate wiring, good for a first look.

---

## After the first real artifact

Public framing waits for a real artifact, not the other way around:

> Recorded an external refuse-before-sign: independent buyer host, TWZRD gate on the
> pay path, block with zero keypair signatures. Internal supply was step one — this is
> step two. `npm i twzrd-x402-gate@0.8.14` — hook before sign, not after.

Don't post this (or anything like it) before section 5's artifact is real and
operator-posted. An ops-funded settle or a self-run local proof is not this claim.

---

## Reference

- Package + full README (config table, seller-side settle guard, PayAI
  agentic-payments variant, proxied-client compatibility detail — npm renders the
  same README shown in the source tarball): https://www.npmjs.com/package/twzrd-x402-gate
- Free preflight API: https://intel.twzrd.xyz (`POST /v1/intel/preflight`, no auth)
- Section 3's "what counts as external" criteria is inlined in full above — it's
  sourced from an internal operator-acceptance doc that isn't independently public;
  nothing in this runbook is missing because of that.
