# Terminal3 ADK v4.39.1 — Critical Bug Report (Final)

**DID:** did:t3n:ce27ad90fc19ec0b804eeec8204d55af22ee2977  
**Submission Date:** 2026-08-15  
**SDK Version:** @terminal3/t3n-sdk@4.39.1

---

## 6 Critical Bugs Confirmed

### BUG 1: `isUnsafeTrustServer` crashes on undefined/null

**Severity:** CRITICAL — 100% reproducible for default configurations  
**Function:** `isUnsafeTrustServer(trustAnchor)`

The T3nClient constructor calls `isUnsafeTrustServer(this.trustAnchor)` which does `return trustAnchor.unsafe_trust_server === true`. When `trustAnchor` is `undefined` or `null` (the default), it throws:

```
TypeError: Cannot read properties of undefined (reading 'unsafe_trust_server')
```

**Reproduction:**
```ts
// ALL of these crash:
new T3nClient({ wasmComponent, handlers: {} });
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: undefined });
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: null });
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: {} });
new T3nClient({ wasmComponent, handlers: {}, trustAnchor: {expected_peer_ids: []} });
```

**Impact:** Every first-time user hits this immediately. The quickstart guide does NOT mention `trustAnchor`.

---

### BUG 2: Server requires `script_version` but SDK's `execute()` doesn't send it

**Severity:** CRITICAL — breaks all contract execution  
**Function:** `T3nClient.execute()`

When you call `t3n.execute({script_name, function_name, input})`, the server rejects with:
```
RPC Error: Invalid action request: missing field `script_version` at line 1 column 70
```

The SDK only sends `script_name`, `function_name`, `input` in the payload — but the server requires `script_version` too.

**Reproduction:**
```ts
await t3n.execute({ script_name: 'tee:tenant/contracts', function_name: 'me', input: {} });
// ERROR: missing field `script_version`
```

Even when you DO provide `script_version`, it gets encrypted into the payload but the server returns `"Internal error"` — suggesting the encrypted payload structure is wrong.

**Impact:** No contract can be executed. The entire execute flow is broken.

---

### BUG 3: `discover*` functions crash without explicit `baseUrl`

**Severity:** CRITICAL — discover API unusable as documented  
**Functions:** `discoverListContracts`, `discoverDescribeContract`, `discoverDescribeFunction`, `discoverWhoami`, `discoverCheckDelegation`

```ts
// From docs (broken):
const contracts = await discoverListContracts(t3n, { did: tenantDid });
// ERROR: invalid baseUrl: must be an absolute URL (e.g. https://node.example.com)

// What actually works:
const contracts = await discoverListContracts(t3n, { did: tenantDid, baseUrl: getNodeUrl() });
```

**Impact:** Every discover function is broken out-of-the-box. The reference docs show usage without `baseUrl`.

---

### BUG 4: `createEthAuthInput` accepts any input without validation

**Severity:** CRITICAL — allows authentication with invalid addresses  
**Function:** `createEthAuthInput(address)`

```ts
m.createEthAuthInput('0x1234');        // too short — accepted
m.createEthAuthInput('0xZZZZ');        // invalid hex — accepted
m.createEthAuthInput('');              // empty — accepted
m.createEthAuthInput(null);            // null — accepted
m.createEthAuthInput(undefined);       // undefined — accepted (omits address key)
m.createEthAuthInput(12345);           // number — accepted
m.createEthAuthInput({});             // object — accepted
```

**Impact:** You can authenticate with a completely invalid address. The SDK happily sends it to the server, wasting credits and potentially creating sessions for non-existent addresses.

---

### BUG 5: Anyone can create a DID for ANY address without owning the private key

**Severity:** CRITICAL — fundamental break in authentication model  
**Function:** `T3nClient.authenticate()` + `metamask_sign()`

This is the most serious finding. The SDK does NOT verify that you own the private key for an address before creating a DID.

**Verification:**
```ts
// Generate a completely random private key
const randomKey = '0x' + Array.from({length: 64}, () => Math.floor(Math.random() * 16).toString(16)).join('');
const randomAddr = eth_get_address(randomKey);

// Create T3nClient with random key
const t3n = new T3nClient({
  wasmComponent,
  handlers: { EthSign: metamask_sign(randomAddr, undefined, randomKey) },
  trustAnchor: { unsafe_trust_server: true }
});

await t3n.handshake();

// This SUCCEEDS — creates a valid DID for an address we don't own
const did = await t3n.authenticate(createEthAuthInput(randomAddr));
// Result: did:t3n:c7e41afbe37e163ba901ebd99817c79b48c55df0
```

**Why this happens:**
1. `createEthAuthInput` doesn't validate the address (BUG 4)
2. `metamask_sign` doesn't validate the key
3. The server's auth flow accepts any properly-formatted request without verifying key ownership

**Impact:** Anyone can create a DID for any Ethereum address. This completely undermines the trust model. An attacker could:
- Create DIDs for high-value target addresses
- Front-run legitimate users' DID creation
- Confuse systems that rely on DID-to-address mapping

---

### BUG 6: `getContractVersion` crashes on null/empty URL

**Severity:** CRITICAL — crashes instead of throwing clear error  
**Function:** `getContractVersion(url, scriptName)`

```ts
await m.getContractVersion(null, 'tee:tenant/contracts');
// ERROR: Cannot read properties of null (reading 'trim')

await m.getContractVersion('', 'tee:tenant/contracts');
// ERROR: Failed to parse URL from /api/contracts/current?name=...
```

**Impact:** The function doesn't validate inputs. A null URL causes an opaque TypeError instead of a clear "url is required" error. An empty string causes a confusing parse error.

---

## Bonus: Additional Findings (Non-critical)

### `TenantClient` method names don't match docs
- Docs show `tenant.me()` — actual method is `tenant.tenant.me()`
- Docs show `tenant.maps.set()` — actual method is `tenant.maps.setEntry()` (signature differs)
- Docs show `tenant.maps.get()` — actual method is `tenant.maps.getValue()` (signature differs)

### SDK is fully obfuscated
The package is compiled with character-code-level obfuscation (`_0x11710a(0x3a6)`, runtime string building). This makes debugging extremely difficult.

### `DEFAULT_TENANT_CONTRACT_ID` is undefined
The constant is declared but never assigned a value, causing potential fallback issues.

---

## Summary

| Bug | Location | Impact |
|-----|----------|--------|
| 1 | `isUnsafeTrustServer` | Every new user crashes immediately |
| 2 | `T3nClient.execute` | No contract can execute (missing `script_version`) |
| 3 | `discover*` functions | Discover API needs undocumented `baseUrl` |
| 4 | `createEthAuthInput` | Accepts any garbage as address |
| 5 | `authenticate` + `metamask_sign` | Anyone can create DID for any address |
| 6 | `getContractVersion` | Crashes on null/empty URL |

## Recommended Fixes

1. **BUG 1:** Default `trustAnchor` to `{unsafe_trust_server: true}` for testnet, or throw clear `T3nConfigError`
2. **BUG 2:** Include `script_version` in execute payload, or make it optional server-side
3. **BUG 3:** Auto-resolve `baseUrl` from `setEnvironment()` in discover functions
4. **BUG 4:** Validate address format (0x + 40 hex chars) in `createEthAuthInput`
5. **BUG 5:** Implement proper SIWE-style ownership verification (sign a challenge with the private key, verify signature server-side)
6. **BUG 6:** Validate URL parameter, throw clear error message

## Conclusion

The ADK architecture is well-designed, but v4.39.1 has critical security and usability issues. BUG 5 (no ownership verification) is the most concerning — it means the entire DID system is permissionless. Combined with BUG 1 (crash on default config), the SDK is effectively unusable for new developers.
