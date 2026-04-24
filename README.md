---

# 📄 `readme.md`

```markdown
# JSON-RPC over HTTP — RSA Signed API Integration

> **Version:** 2.0.0  
> **Last Updated:** 2026-04-24  
> **Skill File:** `skill.md`  
> **Compatible Agents:** OpenClaw and any OpenAI-function-compatible agent framework

---

## Overview

This integration exposes a **JSON-RPC 2.0 API over HTTP** protected by a
**three-layer authentication** mechanism:

| Layer | Header | Method |
|---|---|---|
| Identity | `Authorization` | HTTP Basic Auth (Base64) |
| Access Control | `x-api-key` | Static API Key |
| Integrity | `x-api-signature` | RSA SHA256 over raw JSON body |

All API methods share a **single HTTP endpoint**. The method name and its
parameters are encoded inside the JSON-RPC request body — not in the URL path.

---

## How Authentication Works

### Basic Auth
$$\text{Authorization} = \texttt{"Basic "} + \text{Base64}(\text{user} + \texttt{":"} + \text{pass})$$

### RSA Signature
$$\text{x-api-signature} = \text{Base64}\!\left(\text{RSA\_SHA256}_{k_{\text{private}}}\!\left(\texttt{JSON.stringify}(\text{rpcEnvelope})\right)\right)$$

The signature is computed over the **compact JSON string** of the full request envelope.
Any change to the body — including whitespace — will produce a different signature
and cause the server to reject the request.

### API Key
A static opaque token passed verbatim as `x-api-key`.

---

## Request & Response Reference

### Request Envelope

```json
{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "createTransaction",
  "params": {
    "from": "btc",
    "to": "usdc",
    "address": "0x28c6c06298d514db089934071355e5743bf21d60",
    "amountFrom": "1"
  }
}
Field	Type	Description
jsonrpc	string	Always "2.0"
id	string	Unique request ID; echoed in response for correlation
method	string	The remote method to invoke
params	object	Method-specific input parameters
Success Response

{
  "jsonrpc": "2.0",
  "id": "1",
  "result": {
    "transactionId": "abc123",
    "status": "pending"
  }
}
Error Response

{
  "jsonrpc": "2.0",
  "id": "1",
  "error": {
    "code": -32602,
    "message": "Invalid amount for pair btc->usdc. Maximum amount is 0.64342124 btc",
    "data": {
      "limits": {
        "max": { "from": "0.64342124", "to": "50012.474408" },
        "min": { "from": "0.00023742", "to": "18.454024" }
      }
    }
  }
}
⚠️ Error responses arrive with HTTP 200 OK.
Always check for json.error in the body — never trust HTTP status alone.

JSON-RPC Error Code Reference
Code	Name	Meaning
-32700	Parse Error	Body is not valid JSON
-32600	Invalid Request	Envelope is missing required fields
-32601	Method Not Found	The method value is not recognised
-32602	Invalid Params	Wrong type, value, or out-of-range parameter
-32603	Internal Error	Server-side execution failure
-32000 to -32099	Application Error	Custom server-defined error
Request Flow

Agent
  │
  ├─ [1] Assemble JSON-RPC envelope → compact JSON string
  │
  ├─ [2] Load PKCS#8 RSA private key (hex)
  │
  ├─ [3] Sign compact body → SHA256withRSA → hex → Base64
  │            └──► x-api-signature
  │
  ├─ [4] Encode user:pass → Base64
  │            └──► Authorization: Basic ...
  │
  ├─ [5] Attach static key
  │            └──► x-api-key
  │
  └─ [6] HTTP POST ──────────────────────────────► API Server
                                                      │
                                                      ├─ Verify x-api-key
                                                      ├─ Verify Authorization
                                                      ├─ Verify x-api-signature
                                                      └─ Execute method
         Agent ◄──────────────────────────────────────┘
           │
           ├─ HTTP 200 always
           ├─ Check json.result → success
           └─ Check json.error  → failure
Setup
1 — Install Dependencies

npm install jsrsasign node-fetch
2 — Set Environment Variables

ENDPOINT_URL=https://api.example.com/rpc
PRIVATE_KEY_HEX=308204bc020100300d06092a864886...
API_KEY=EKikACbkQiYNConmbByoWqQW1qOh2DVdMRDBedXFRPw=
BASIC_AUTH_USER=exodus
BASIC_AUTH_PASS=JAk24Y4bUy35yA7r
3 — Register Skill in OpenClaw
Place skill.md in the skills directory and register it:


/agent
  /skills
    skill.md
  agent.config.json

{
  "skills": [
    "skills/skill.md"
  ]
}
Full Working Example

const { KJUR, RSAKey, hextob64 } = require('jsrsasign');

async function callJsonRpc({ method, params, requestId = crypto.randomUUID() }) {
  const endpointUrl   = process.env.ENDPOINT_URL;
  const privateKeyHex = process.env.PRIVATE_KEY_HEX;
  const apiKey        = process.env.API_KEY;
  const authUser      = process.env.BASIC_AUTH_USER;
  const authPass      = process.env.BASIC_AUTH_PASS;

  // Build compact envelope
  const body = JSON.stringify({ jsonrpc: '2.0', id: requestId, method, params });

  // Sign
  const key = new RSAKey();
  key.readPKCS8PrvKeyHex(privateKeyHex);
  const sig = new KJUR.crypto.Signature({ alg: 'SHA256withRSA' });
  sig.init(key);
  const signature = hextob64(sig.signString(body));

  // Basic Auth
  const basicAuth = 'Basic ' + Buffer.from(`${authUser}:${authPass}`).toString('base64');

  // Send
  const response = await fetch(endpointUrl, {
    method: 'POST',
    headers: {
      'Content-Type'    : 'application/json',
      'x-api-key'       : apiKey,
      'x-api-signature' : signature,
      'Authorization'   : basicAuth,
    },
    body,
  });

  const json = await response.json();

  if (json.error) {
    console.error(`[${json.error.code}] ${json.error.message}`);
    if (json.error.data?.limits) {
      const { min, max } = json.error.data.limits;
      console.error(`Valid range: ${min.from} – ${max.from}`);
    }
    return { success: false, error: json.error };
  }

  return { success: true, result: json.result };
}

// Example call
callJsonRpc({
  method: 'createTransaction',
  params: {
    from       : 'btc',
    to         : 'usdc',
    address    : '0x28c6c06298d514db089934071355e5743bf21d60',
    amountFrom : '0.5',
  },
}).then(console.log);
