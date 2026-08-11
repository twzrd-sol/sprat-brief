# Path B external integration runbook

**Goal:** install the buyer-side x402 trust gate and produce a **matched BLOCK+ALLOW
pair** — an external adoption signal, not TWZRD dogfood and not one arm alone.

**Package:** [`twzrd-x402-gate`](https://www.npmjs.com/package/twzrd-x402-gate) (npm,
MIT, zero deps). Current: `0.8.14`.

This is a technical runbook, not a pitch. Every command and field below is copied from
the package's own README and the monorepo's operator-acceptance doc, not paraphrased.

---

## Outreach sequence (72h)

| Priority | Target |
|---|---|
| 1 | **Vicky** |
| 2 | **Nick** |
| 3 | **Lucas** |

No artificial 24h stagger between names — reach all three, the slate is evaluated at
the 72h mark as a whole. Success = **one named external host, one matched pair**
(section 5) — a `BLOCK` run and an `ALLOW` run from the *same* environment and stable
`integration` label, not a star count, a download number, or a single arm alone.

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

### PayAI `x402-solana` — instrumented (this is what produces a real artifact)

Minimal wiring proves the mechanism; this version also proves it — by instrumenting
your own signing boundary so `signerInvocations` is measured, not assumed, and by
capturing everything section 5's artifact template needs in one run.

```typescript
import { createX402Client } from "x402-solana";
import { createTwzrdBeforePaymentHook } from "twzrd-x402-gate";

const integration = "your-project-name";  // stable label, yours
const runId = crypto.randomUUID();        // fresh per arm — run this twice, once
                                           // against a seller you expect to block,
                                           // once against one you expect to allow
const decisions: unknown[] = [];
let signerInvocations = 0;

// Wrap your OWN wallet's signer so a block is proven, not assumed.
const instrumentedWallet = new Proxy(wallet, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (prop === "signTransaction" && typeof value === "function") {
      return async (...args: unknown[]) => {
        signerInvocations += 1;
        return value.apply(target, args);
      };
    }
    return typeof value === "function" ? value.bind(target) : value;
  },
});

const client = createX402Client({
  wallet: instrumentedWallet,
  network: "solana",
  beforePayment: createTwzrdBeforePaymentHook({
    refuseWashFlagged: true,
    gateOnCanSpend: false,   // opt-in only — see package config table
    failOpen: false,
    attribution: { integration, runId },
    onDecision: (detail) => decisions.push({ ...detail, evaluatedAt: new Date().toISOString() }),
  }),
});

try {
  const response = await client.fetch(targetUrl);
  console.log({ outcome: "completed", httpStatus: response.status, integration, runId, signerInvocations, decisions });
} catch (error) {
  console.log({ outcome: "refused", error: String(error), integration, runId, signerInvocations, decisions });
}
```

A block surfaces as a **thrown error** from `client.fetch` (confirmed against the
package's own `twzrd-gate-eval-refuse` bin output: `"observed_error": "Failed to
create payment payload: Payment creation aborted: ..."`), not a resolved response you
have to inspect — the `try`/`catch` split above is the real control flow, not a
simplification.

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
5. **A matched pair, not one arm.** Closing external Path B for the sprint means a
   `BLOCK` run (`signerInvocations: 0`) *and* an `ALLOW` run (`signerInvocations >= 1`,
   real mainnet settlement tx) from the same operator environment and stable
   `integration`, with a fresh `runId` per arm. One arm alone is partial evidence —
   real, worth logging, but not a closed milestone.

### What does **not** count

- npm download counts
- Free preflight HTTP hits with no attribution and no operator-posted `runId`
- A `runId` we generated, with no transcript you posted yourself
- The local harness alone, run inside our own repo/CI
- A successful payment with no gate decision in the path

---

## 4. Evidence checklist (screen-share ready)

Before calling a pair complete, confirm every line for **both** arms:

```text
[ ] Host is NOT owned/operated by TWZRD
[ ] Operator / org is named
[ ] Gate is wired into the pay path (createTwzrdBeforePaymentHook /
    installTwzrdAutoGate on a real client) — a curl to /v1/intel/preflight alone
    does not satisfy this
[ ] Gate evaluation timestamp is BEFORE any signTransaction / signer invocation
[ ] BLOCK arm: signerInvocations = 0, zero gas, zero tx, refusal reason captured
[ ] ALLOW arm: signerInvocations >= 1, HTTP 200, real mainnet settlement signature
[ ] Same operator environment + same integration label across both arms
[ ] Fresh, distinct runId per arm
[ ] Raw gate decision (onDecision detail or preflight JSON) saved for both arms
[ ] Payer pubkey + target URL / seller payTo recorded for both arms
[ ] Both runIds posted by the operator themselves, not generated on their behalf
```

## 5. Artifact template (matched pair)

Fill this in and post it yourself (issue, PR comment, DM) — we do not accept a
transcript we generated on your behalf as evidence (see section 3, bar #2). One arm
alone is worth logging as partial progress but does not close the milestone.

```markdown
### TWZRD Path B external proof — matched pair

* Operator / org: …
* Host runtime: … (not TWZRD-operated)
* Integration: twzrd-x402-gate@0.8.14 + [x402-solana | @x402/core]
* Date: …

#### Arm 1 — BLOCK
* runId: …
* outcome: "refused" (client.fetch threw)
* decision: "block", approved: false
* signerInvocations: 0
* Target endpoint / payTo / payer pubkey: …
* Raw decision payload: \`{ "decision": "block", "trust_score": …, "can_spend": false }\`

#### Arm 2 — ALLOW
* runId: …
* outcome: "completed", httpStatus: 200
* decision: "allow", approved: true
* signerInvocations: 1 (or more)
* Mainnet settlement tx signature: …
* Target endpoint / payTo / payer pubkey: …
```

Real `decision` values from the live API are lowercase: `allow`, `warn`, `block` — not
`ALLOW`/`BLOCK`. Copy responses verbatim rather than retyping them.

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

## Founder messaging

**Until the matched pair is captured, say "enrolling," not "proven":**

> Internal proof is locked across our dual Cloudflare Workers starters on Solana
> mainnet. Onboarding is underway with our first external operators to run pre-sign
> evaluation (`refuse-before-sign`) through `twzrd-x402-gate`.

**Only after a real, operator-posted matched pair (section 5) exists:**

> Recorded an external refuse-before-sign *and* an external allow-and-settle:
> independent buyer host, TWZRD gate on the pay path, matched pair with zero keypair
> signatures on block and a verified mainnet tx on allow. Internal supply was step
> one — this is step two.

A single arm, an ops-funded settle, or a self-run local proof does not unlock the
second message.

---

## Reference

- Package + full README (config table, seller-side settle guard, PayAI
  agentic-payments variant, proxied-client compatibility detail — npm renders the
  same README shown in the source tarball): https://www.npmjs.com/package/twzrd-x402-gate
- Free preflight API: https://intel.twzrd.xyz (`POST /v1/intel/preflight`, no auth)
- Section 3's "what counts as external" criteria is inlined in full above — it's
  sourced from an internal operator-acceptance doc that isn't independently public;
  nothing in this runbook is missing because of that.
