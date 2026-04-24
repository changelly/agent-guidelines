# Skill: JSON-RPC over HTTP with RSA Signature Authentication

## Metadata
- **skill_id**: jsonrpc-rsa-signed-api-integration
- **version**: 2.0.0
- **tags**: [jsonrpc, api, rsa, signature, authentication, http, security]

---

## Description
This skill enables agents to communicate with a **JSON-RPC 2.0 over HTTP** API that requires:
1. **RSA SHA256 digital signature** over the raw JSON-RPC request body
2. **API Key** header (`x-api-key`)
3. **Basic Auth** header (`Authorization`)
4. **Dynamic signature** header (`x-api-signature`)

All requests follow the **JSON-RPC 2.0 specification** — a single HTTP endpoint receives
all method calls, differentiated by the `method` field in the body.

---

## JSON-RPC 2.0 Protocol Rules

| Rule | Detail |
|---|---|
| Transport | HTTP POST only |
| Endpoint | Single URL for all methods |
| Content-Type | `application/json` |
| Request field `jsonrpc` | Always `"2.0"` |
| Request field `id` | Unique string or integer per request |
| Request field `method` | The API method to call (e.g. `"createTransaction"`) |
| Request field `params` | Object containing method-specific parameters |
| Success response | Contains `result` field |
| Error response | Contains `error` field with `code`, `message`, and optional `data` |

> ⚠️ HTTP status will be **200 OK** even for JSON-RPC errors.
> Always inspect the response body for an `error` field — never rely on HTTP status alone.

---

## Prerequisites / Dependencies

| Dependency | Purpose |
|---|---|
| `jsrsasign` | RSA key parsing and SHA256withRSA signing |
| `node-fetch` or `axios` | HTTP request execution |
| `Buffer` / `btoa` | Base64 encoding for Basic Auth |
| PKCS#8 RSA Private Key (hex) | Signing identity |

---

## Inputs

| Parameter | Type | Required | Description |
|---|---|---|---|
| `endpoint_url` | `string` | ✅ | Single base URL for all JSON-RPC calls |
| `method` | `string` | ✅ | JSON-RPC method name (e.g. `"createTransaction"`) |
| `params` | `object` | ✅ | Method parameters as a key-value object |
| `request_id` | `string` | ✅ | Unique request identifier (e.g. `"1"`, UUID) |
| `private_key_hex` | `string` | ✅ | PKCS#8 RSA private key in HEX format |
| `api_key` | `string` | ✅ | Static API key value |
| `basic_auth_user` | `string` | ✅ | Username for Basic Auth |
| `basic_auth_pass` | `string` | ✅ | Password for Basic Auth |

---

## Outputs

| Output | Type | Description |
|---|---|---|
| `success` | `boolean` | `true` if response contains `result`, `false` if `error` |
| `result` | `object/null` | JSON-RPC result payload (if successful) |
| `error` | `object/null` | JSON-RPC error object: `{ code, message, data }` |
| `response_id` | `string` | The `id` echoed back from the server |
| `headers_sent` | `object` | The computed headers that were sent |

---

## Step-by-Step Execution Plan

### Step 1 — Build the JSON-RPC Request Body
- Construct the JSON-RPC 2.0 envelope:
```json
{
  "jsonrpc": "2.0",
  "id": "<request_id>",
  "method": "<method>",
  "params": { ...params }
}
```
## Serialize to a raw JSON string using JSON.stringify()
⚠️ Do not pretty-print — use compact serialization to ensure signature consistency

### Step 2 — Load RSA Private Key
Parse the PKCS#8 hex private key using RSAKey from jsrsasign
Call key.readPKCS8PrvKeyHex(privateKeyHex)
Step 3 — Sign the Raw JSON Body
Create a KJUR.crypto.Signature instance with algorithm "SHA256withRSA"
Initialize with the loaded RSA key
Call sig.signString(rawBody) on the compact JSON string
Convert hex result to Base64 via hextob64()
This becomes x-api-signature


### Step 4 — Build Basic Auth Header
Concatenate "username:password"
Encode as Base64 using Buffer.from(...).toString('base64')
Prepend "Basic " → Authorization header

### Step 5 — Assemble Headers
```
{
  "Content-Type": "application/json",
  "x-api-key": "<api_key>",
  "x-api-signature": "<base64-rsa-signature>",
  "Authorization": "Basic <base64-credentials>"
}
```
### Step 6 — Send HTTP POST Request
Method: POST (always, per JSON-RPC over HTTP spec)
URL: endpoint_url
Headers: assembled above
Body: the raw JSON string from Step 1
Step 7 — Parse and Validate JSON-RPC Response
Parse response body as JSON
Check for presence of error field:
If error → extract code, message, data and return as structured error
If result → return as success payload

### Step 8 — Return Structured Output

Pseudocode
```bash
FUNCTION call_jsonrpc(endpoint, method, params, requestId, privateKeyHex, apiKey, authUser, authPass):

  // Step 1: Build JSON-RPC body
  body = JSON.stringify({
    jsonrpc : "2.0",
    id      : requestId,
    method  : method,
    params  : params
  })

  // Step 2: Load RSA Key
  key = new RSAKey()
  key.readPKCS8PrvKeyHex(privateKeyHex)

  // Step 3: Sign body
  sig = new KJUR.crypto.Signature({ alg: "SHA256withRSA" })
  sig.init(key)
  signature = hextob64(sig.signString(body))

  // Step 4: Basic Auth
  basicAuth = "Basic " + base64(authUser + ":" + authPass)

  // Step 5: Headers
  headers = {
    "Content-Type"    : "application/json",
    "x-api-key"       : apiKey,
    "x-api-signature" : signature,
    "Authorization"   : basicAuth
  }

  // Step 6: HTTP POST
  response = HTTP_POST(endpoint, headers, body)

  // Step 7: Parse response
  json = JSON.parse(response.body)

  return parsed response
```
JavaScript Implementation Reference
```
const { KJUR, RSAKey, hextob64 } = require('jsrsasign');

/**
 * Call a JSON-RPC 2.0 method over HTTP with RSA signature auth.
 *
 * @param {object} options
 * @param {string} options.endpointUrl      - API base URL
 * @param {string} options.method           - JSON-RPC method name
 * @param {object} options.params           - Method parameters
 * @param {string} options.requestId        - Unique request ID
 * @param {string} options.privateKeyHex    - PKCS#8 RSA private key hex
 * @param {string} options.apiKey           - Static API key
 * @param {string} options.basicAuthUser    - Basic auth username
 * @param {string} options.basicAuthPass    - Basic auth password
 * @returns {Promise<object>}               - Structured response
 */
async function callJsonRpc({
  endpointUrl,
  method,
  params,
  requestId,
  privateKeyHex,
  apiKey,
  basicAuthUser,
  basicAuthPass,
}) {
  // Step 1: Build JSON-RPC envelope (compact, no pretty-print)
  const body = JSON.stringify({
    jsonrpc: '2.0',
    id: requestId,
    method,
    params,
  });

  // Step 2: Load RSA Private Key
  const key = new RSAKey();
  key.readPKCS8PrvKeyHex(privateKeyHex);

  // Step 3: Sign the raw body string
  const sig = new KJUR.crypto.Signature({ alg: 'SHA256withRSA' });
  sig.init(key);
  const signature = hextob64(sig.signString(body));

  // Step 4: Build Basic Auth header
  const basicAuth = 'Basic ' + Buffer.from(`${basicAuthUser}:${basicAuthPass}`).toString('base64');

  // Step 5: Assemble headers
  const headers = {
    'Content-Type'    : 'application/json',
    'x-api-key'       : apiKey,
    'x-api-signature' : signature,
    'Authorization'   : basicAuth,
  };

  // Step 6: Send HTTP POST
  const response = await fetch(endpointUrl, {
    method: 'POST',
    headers,
    body,
  });

  // Step 7: Parse JSON-RPC response
  const json = await response.json();

  if (String(json.id) !== String(requestId)) {
    throw new Error(`Response ID mismatch: expected ${requestId}, got ${json.id}`);
  }

  return json;
}
```
⚠️ Critical security guidelines for agents handling this skill:

Never log privateKeyHex or basicAuthPass in plain text
Never hardcode secrets — read from environment variables
The signature is bound to the exact raw body string — never reuse across requests


Example Agent Invocation
```
{
  "skill_id": "jsonrpc-rsa-signed-api-integration",
  "inputs": {
    "endpoint_url": "https://api.example.com/rpc",
    "method": "createTransaction",
    "params": {
      "from": "btc",
      "to": "usdc",
      "address": "0x28c6c06298d514db089934743bf21d60",
      "amountFrom": "0.5"
    },
    "request_id": "1",
    "private_key_hex": "308204bc020100300d06092a86...",
    "api_key": "EKikAC...",
    "basic_auth_user": "user",
    "basic_auth_pass": "auth_pass"
  }
}
```
