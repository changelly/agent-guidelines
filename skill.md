# Skill: JSON-RPC over HTTP with RSA Signature Authentication

## Metadata
- **skill_id**: jsonrpc-rsa-signed-api-integration
- **version**: 2.0.0
- **tags**: [jsonrpc, api, rsa, signature, authentication, http, security]

---

## Description
This skill enables agents to communicate with a **JSON-RPC 2.0 over HTTP** API that requires:
1. **RSA SHA256 digital signature** over the raw JSON-RPC request body
2. **Dynamic signature** header (`x-api-signature`)

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
Parse the PKCS#8 hex private key and PKCS#1 public key using crypto libraries
```
crypto.createPrivateKey({
  key: Buffer.from(privateKeyString, 'hex'),
  format: 'der',
  type: 'pkcs8',
});

crypto.createPublicKey(privateKey).export({
  type: 'pkcs1',
  format: 'der',
});
```

### Step 3 — Sign the Raw JSON Body
```
const message = {
  jsonrpc: '2.0',
  id: 'test',
  method: 'getCurrencies',
};

const body = JSON.stringify(message);

const signature = crypto.sign('sha256', Buffer.from(body), privateKey).toString('base64');
```
This becomes x-api-signature

### Step 4 — Assemble Headers
```
{
    'Content-Type': 'application/json',
    'X-Api-Key': crypto.createHash('sha256').update(publicKey).digest('base64'),
    'X-Api-Signature': signature,
}
```
### Step 5 — Send HTTP POST Request
Method: POST (always, per JSON-RPC over HTTP spec)
URL: endpoint_url
Headers: assembled above
Body: the raw JSON string from Step 1

### Step 6 — Parse and Validate JSON-RPC Response
Parse response body as JSON
Check for presence of error field:
If error → extract code, message, data and return as structured error
If result → return as success payload

### Step 7 — Return Structured Output

JavaScript Implementation Reference
```
const crypto = require('crypto');

const privateKeyString = 'YOUR_PRIVATE_KEY';

const privateKey = crypto.createPrivateKey({
  key: Buffer.from(privateKeyString, 'hex'),
  format: 'der',
  type: 'pkcs8',
});

const publicKey = crypto.createPublicKey(privateKey).export({
  type: 'pkcs1',
  format: 'der',
});

const message = {
  jsonrpc: '2.0',
  id: 'test',
  method: 'getCurrencies',
};

const body = JSON.stringify(message);

const signature = crypto.sign('sha256', Buffer.from(body), privateKey);

(async () => {
  const res = await fetch('https://api.changelly.com/v2', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Api-Key': crypto.createHash('sha256').update(publicKey).digest('base64'),
      'X-Api-Signature': signature.toString('base64'),
    },
    body,
  });

  console.log(await res.text());
})();

```
⚠️ Critical security guidelines for agents handling this skill:

Never log privateKeyHex in plain text
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
    "private_key_hex": "308204bc020100300d06092a86..."
  }
}
```
