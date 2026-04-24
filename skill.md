# Skill: JSON-RPC over HTTP with RSA Signature Authentication

## Metadata
- **skill_id**: jsonrpc-rsa-signed-api-integration
- **version**: 2.0.0
- **author**: agent-system
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
