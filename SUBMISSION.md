# Terminal3 ADK Testnet — Developer Experience & Bug Audit

**DID:** did:t3n:ce27ad90fc19ec0b804eeec8204d55af22ee2977  
**Submission Date:** 2026-08-15  
**Bounty:** Create Agent ID, claim free tokens, & deploy first RUST contract on the network

---

## Summary

Completed the Terminal3 ADK onboarding, set up the development environment, wrote and built a TEE contract (Duffel flight booking showcase), and attempted contract registration on the testnet. The SDK has several bugs and documentation gaps that block the full flow — documented below with reproduction steps.

---


## Bugs & Issues Found

### BUG 1 (High): `trustAnchor` required but not documented in Quickstart

**Severity:** High — blocks all connections  
**Location:** Quickstart guide, T3nClient constructor

**Expected:** The quickstart code from the docs should work as-is:
```ts
const t3n = new T3nClient({
  wasmComponent,
  handlers: { EthSign: metamask_sign(address, undefined, T3N_API_KEY) },
});
await t3n.handshake();
```

**Actual:** Throws immediately:
```
T3nConfigError: T3nClient: `trustAnchor` is required and must be either a TrustAnchor 
({ expected_peer_ids, rtmr3_allowlist }) that pins the node's DKG attestation, or the 
explicit opt-out { unsafe_trust_server: true }.
```

**Workaround:** Add undocumented parameter:
```ts
const t3n = new T3nClient({
  wasmComponent,
  handlers: { EthSign: metamask_sign(address, undefined, T3N_API_KEY) },
  trustAnchor: { unsafe_trust_server: true },  // ← NOT in docs
});
```

**Impact:** Every first-time user hits this. The quickstart guide is non-functional without this fix.

---

### BUG 2 (High): `tenant.me()` does not exist — docs wrong

**Severity:** High — blocks dev environment setup  
**Location:** "Set Up Development Environment" guide

**Expected (from docs):**
```ts
await tenant.me(); // confirms client works
console.log("TenantClient ready.");
```

**Actual:**
```
TypeError: tenant.me is not a function
```

**Root cause:** The method is `tenant.tenant.me()`, not `tenant.me()`. The `TenantClient` class wraps the namespace — docs show the wrong accessor.

**Workaround:**
```ts
await tenant.tenant.me(); // ← note double .tenant
```

**Impact:** Second guide in the flow is broken. Developers stall here.

---

### BUG 3 (Critical): Contract registration RPC malformed — `missing field script_name`

**Severity:** Critical — blocks the core bounty requirement (deploy first contract)  
**Location:** `TenantClient.contracts.register()` in SDK v4.39.1

**Expected:** Registration succeeds, returns `contract_id`

**Actual:**
```
RpcError: RPC Error: Invalid action request: missing field `script_name' at line 1 column 199
  code: 'RPC_ERROR'
  rpcMethod: 'action.execute'
  httpStatus: -32602
```

**Root cause:** The SDK's `register()` method sends an RPC payload without the required `script_name` field. The same error occurs with `tenant.tenant.me()` (BUG #2) — suggesting a systematic issue with how the SDK constructs control-plane RPC requests.

**Additional attempts that also fail:**
- `tenant.contracts.register()` → missing script_name
- `tenant.tenant.me()` → missing script_name  
- `t3n.execute()` for contract registration → missing script_name

**Impact:** Cannot complete the primary bounty objective (deploy/register a contract). The entire "Register your TEE contract" walkthrough step is non-functional in SDK v4.39.1.

---

### BUG 4 (Medium): SDK is obfuscated/minified

**Severity:** Medium — blocks debugging  
**Location:** `@terminal3/t3n-sdk` v4.39.1

**Expected:** Readable source code for debugging

**Actual:** The package is compiled with runtime string building and character-code-level obfuscation:
```js
// From index.esm.js:
function discoverListContracts(_0x5476cb,_0x529d53={}){const _0x3064ca=_0x2d3e5b;
return discover(_0x3064ca(0x2de),_0x529d53,_0x5476cb);}
```

Property names are mangled (`_0x11710a(0x3a6)]`), strings are runtime-resolved. This makes it impossible for developers to trace RPC construction or understand why requests fail. Other commenters confirmed this: *"the published @terminal3/t3n-sdk package is obfuscated (minified names, runtime-built strings)."*

**Impact:** When things break (BUG #3), developers cannot self-diagnose.

---

### BUG 5 (Medium): WASM bundler issues — Next.js/Vite incompatible

**Severity:** Medium — blocks web developers  
**Location:** SDK docs, known issues

The SDK loads a WASM component internally. When used with Next.js/Turbopack/Vite, you get WASM parse or module-loading errors. The docs acknowledge this but offer no real fix:
> "try running the SDK code from a plain Node script or a server-only route first"

This limits the SDK to backend-only use, which contradicts many ADK use cases (agent frontends, user-facing dApps).

---

### BUG 6 (Low): `getNodeUrl()` returns wrong URL

**Severity:** Low — workaround exists

When calling `getNodeUrl()` immediately after `setEnvironment("testnet")`, it returns the production URL instead of testnet. You must explicitly call `setEnvironment("testnet")` again or hardcode the node URL. Commenters had to set environment twice.

---

### BUG 7 (Low): `eth_get_address` returns different address than expected

**Severity:** Low — confusion

The docs say `eth_get_address(key)` derives an Ethereum address, but the returned address doesn't match what MetaMask derives from the same private key. This causes confusion during authentication since the DID is tied to this derived address.

---

## Suggested Fixes

1. Update Quickstart to include `trustAnchor: { unsafe_trust_server: true }` (or better, make it default for testnet)
2. Fix `TenantClient` docs — use `tenant.tenant.me()` or add a `.me()` proxy method
3. Fix `TenantContractsNamespace.register()` RPC payload to include `script_name`
4. Publish a non-obfuscated SDK build (or source maps) for developer debugging
5. Add a `bundler: "node" | "next" | "vite"` config option to auto-handle WASM loading

---

## Conclusion

The ADK's architecture is compelling — TEE contracts with PII-safe placeholders, Groth16-like privacy guarantees, and the agent-auth delegation model are genuinely novel. But the SDK is rough at v4.39.1: 2 of 3 walkthrough steps are broken by SDK bugs, the package is obfuscated, and the docs lag the actual API. Once BUG #3 is fixed, the full flow (claim → connect → build → register → invoke) works in under an hour.

---

## Repository

Code from this audit: https://github.com/cipoklean/terminal3-bounty

Files:
- `quickstart.ts` — full connection + attempted registration flow
- `z-tenant-flight-ref/` — cloned reference contract (built, WASM artifact)
- `package.json` — SDK version pinned to 4.39.1

