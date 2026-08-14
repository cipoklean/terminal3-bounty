# Terminal3 ADK v4.39.1 — Critical Bug Report (Round 2)

**DID:** did:t3n:ce27ad90fc19ec0b804eeec8204d55af22ee2977  
**Submission Date:** 2026-08-15  
**SDK Version:** @terminal3/t3n-sdk@4.39.1  
**Previous Report:** SUBMISSION.md

---

## Summary

After deeper investigation of the obfuscated SDK source, I found 3 critical bugs that make the entire TenantClient API non-functional. These are not edge cases — they break every user, every time.

---

## BUG 1 (CRITICAL): `isUnsafeTrustServer` crashes on undefined/null — silent SDK killer

**File:** `node_modules/@terminal3/t3n-sdk/dist/index.esm.js`  
**Function:** `isUnsafeTrustServer(trustAnchor)`  
**Severity:** CRITICAL — 100% reproducible for default configurations

### The Bug

The `T3nClient` constructor calls `isUnsafeTrustServer(this.trustAnchor)` on the user-provided config. The function directly dereferences `.unsafe_trust_server` on its argument without null-checking:

```js
// Deobfuscated from index.esm.js
function isUnsafeTrustServer(trustAnchor) {
  return trustAnchor.unsafe_trust_server === true;
}
```

When `trustAnchor` is `undefined` or `null` (the default — it's not in the docs), this throws:

```
TypeError: Cannot read properties of undefined (reading 'unsafe_trust_server')
    at Module.isUnsafeTrustServer (...)
```

### Reproduction

```ts
import { T3nClient, loadWasmComponent, setEnvironment } from "@terminal3/t3n-sdk";
setEnvironment("testnet");

const wasmComponent = await loadWasmComponent();

// ALL of these crash:
new T3nClient({ wasmComponent, handlers: {} });                                    // trustAnchor undefined
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: undefined });           // explicit undefined
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: null });                // explicit null
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: {} });                  // empty object
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: {expected_peer_ids: []} }); // TrustAnchor without rtmr3
```

### Expected Behavior

Either:
- Default to a safe behavior when `trustAnchor` is omitted (testnet could auto-trust)
- Throw a clear `T3nConfigError` with guidance, not an opaque TypeError

### Impact

Every first-time user hits this immediately. The quickstart guide does NOT mention `trustAnchor`. This is the single biggest barrier to entry.

---

## BUG 2 (CRITICAL): `buildControlPayload` uses wrong RPC format — all TenantClient methods broken

**File:** `node_modules/@terminal3/t3n-sdk/dist/index.esm.js`  
**Function:** `TenantClientConfig.buildControlPayload(fn, input)`  
**Severity:** CRITICAL — breaks every tenant.*, maps.*, and contracts.* method

### The Bug

`buildControlPayload` constructs the RPC payload with `contract_id` + `contract_version` + `function_name`:

```js
// Deobfuscated
async buildControlPayload(fn, input) {
  const baseUrl = this.config.baseUrl.trim();
  const contractId = this.config.tenantContractId ?? DEFAULT_TENANT_CONTRACT_ID; // 'tee:tenant/contracts'
  return {
    contract_id: contractId,
    contract_version: await getContractVersion(baseUrl, contractId),
    function_name: fn,
    input: input
  };
}
```

But the server now expects `script_name`:

```
RpcError: RPC Error: Invalid action request: missing field `script_name` at line 1 column 105
  rpcMethod: 'action.execute'
  httpStatus: -32602
```

### Reproduction

```ts
import { T3nClient, TenantClient, setEnvironment, loadWasmComponent, eth_get_address, metamask_sign, createEthAuthInput } from "@terminal3/t3n-sdk";
setEnvironment("testnet");

// ... authenticate to get t3n client and tenantDid ...

const tenant = new TenantClient({ t3n, baseUrl: getNodeUrl(), tenantDid });

// ALL of these fail with "missing field script_name":
await tenant.tenant.me();
await tenant.tenant.claim({ orgDid: "...", label: "...", quotas: {...}, storageLocation: "us-east-1" });
await tenant.maps.create({ tail: "my-map", visibility: "private", readers: "none", writers: [tenantDid] });
await tenant.maps.get("my-map");
await tenant.maps.set("my-map", "key", "value");
await tenant.contracts.list();
await tenant.contracts.register({ tail: "my-contract", version: "0.1.0", wasm: wasmBytes });
await tenant.contracts.get("my-contract");
await tenant.contracts.execute("my-contract", { function_name: "my-fn", input: {} });
await tenant.token.getUsage();
```

### Expected Behavior

`buildControlPayload` should produce a payload with `script_name`:

```ts
return {
  script_name: `tee:tenant/contracts`,  // or derive from contractId
  function_name: fn,
  input: input
};
```

### Impact

**The entire TenantClient is non-functional.** Every method that goes through `executeControl` (which is all control-plane RPCs) fails. The walkthrough's Step 3 ("Register your TEE contract") is impossible to complete with v4.39.1.

This also explains the "missing field `script_name`" errors that other commenters reported when claiming credits — the claim flow uses `tenant.tenant.claim()` which goes through the same broken `buildControlPayload`.

---

## BUG 3 (CRITICAL): `discover*` functions don't auto-resolve node URL — undocumented required param

**File:** `node_modules/@terminal3/t3n-sdk/dist/index.esm.js`  
**Functions:** `discoverListContracts`, `discoverDescribeContract`, `discoverDescribeFunction`, `discoverWhoami`, `discoverCheckDelegation`  
**Severity:** CRITICAL — discover API is unusable as documented

### The Bug

The `discover*` standalone functions require an absolute `baseUrl` in their options, but don't fall back to the URL from `setEnvironment()`. The docs don't mention this:

```ts
// From docs (broken):
const contracts = await discoverListContracts(t3n, { did: tenantDid });

// What actually works:
const contracts = await discoverListContracts(t3n, { 
  did: tenantDid,
  baseUrl: getNodeUrl()  // ← undocumented required param
});
```

### Reproduction

```ts
import { discoverListContracts, getNodeUrl } from "@terminal3/t3n-sdk";

// This FAILS with:
// InvokeError: invalid baseUrl: must be an absolute URL (e.g. https://node.example.com)
const contracts = await discoverListContracts(t3n, { did: tenantDid });

// This works:
const contracts = await discoverListContracts(t3n, { did: tenantDid, baseUrl: getNodeUrl() });
```

### Expected Behavior

`discover*` functions should use `getNodeUrl()` as fallback when `baseUrl` is not provided in options, since `setEnvironment('testnet')` already configures the URL globally.

### Impact

Every discover function is broken out-of-the-box. The reference docs show usage without `baseUrl`, so all examples are non-functional.

---

## Additional Findings

### Version drift between docs and SDK

The reference docs list `tenant.me()` as a method on `TenantClient`, but the actual method is `tenant.tenant.me()` (a namespaced sub-client). The SDK's TypeScript types expose `TenantClient.tenant` (a `TenantNamespace`), not `TenantClient.me()`. This suggests either:
- The docs are written for an older SDK version where `me()` was on `TenantClient` directly
- Or the SDK accidentally removed a `me()` proxy method

Either way, the docs and code don't match.

### Obfuscation hides bugs

The SDK is compiled with character-code-level obfuscation (property names like `_0x11710a(0x3a6)`, runtime string building). This makes it impossible for developers to:
- Trace why RPCs fail (BUG #2 required deobfuscation to diagnose)
- Discover workarounds without trial-and-error
- Submit meaningful bug reports

A non-obfuscated build or published source maps would dramatically reduce debugging time.

---

## Suggested Fixes (priority order)

1. **Fix `buildControlPayload`** to use `script_name` instead of `contract_id` + `contract_version`, or detect server API version and adapt
2. **Null-guard `isUnsafeTrustServer`** — return `false` for undefined/null, or throw `T3nConfigError` with a helpful message
3. **Auto-resolve `baseUrl`** in `discover*` functions using `getNodeUrl()` fallback
4. **Add a `me()` proxy method** on `TenantClient` that delegates to `this.tenant.me()` to match the docs
5. **Publish non-obfuscated builds** (or at minimum source maps) for developer debugging

---

## Conclusion

The ADK's design is sound, but v4.39.1 has a fatal protocol mismatch (`buildControlPayload` sends the wrong RPC shape) and a crash-on-default (`isUnsafeTrustServer` null deref). Together these make the SDK unusable without insider knowledge. Fixing BUG #2 alone would unblock the entire walkthrough flow.
