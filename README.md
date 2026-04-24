# Product API — AI Agent Integration Guide

> **Skill File:** `skill.md`

---

## What is This?

This guide explains how to integrate the changelly api into an AI agent framework
such as [OpenClaw](https://github.com/openclaw) so that the agent can autonomously
execute crypto trading operations on behalf of users.

The Product API uses **JSON-RPC 2.0 over HTTP** and requires every request to be
cryptographically signed. The provided `skill.md` file encodes all the knowledge
the agent needs to authenticate, call, and handle responses from the API correctly.

---

## How It Works

The agent receives a natural language instruction from a user
(e.g. *"swap 0.5 BTC to USDC"*), translates it into a signed JSON-RPC call,
sends it to the Product API, and returns the result back to the user in plain language.
User
│
│ "swap 0.5 BTC to USDC to address 0x123..."
▼
AI Agent (e.g. OpenClaw)
│
├─ Loads skill.md
├─ Builds JSON-RPC envelope
├─ Signs request body with RSA private key
├─ Attaches x-api-key + Authorization headers
│
▼
Product API (JSON-RPC 2.0 over HTTP)
│
├─ Verifies all 3 auth layers
├─ Executes the requested method
│
▼
AI Agent
│
├─ Parses json.result or json.error
├─ Handles limit errors, retries, corrections
│
▼
User
│
"Transaction created. ID: 1xknl..., status: pending"



---

## Prerequisites

Before running an agent against the Product API you will need:

| Requirement | Description |
|---|---|
| **API Key** | Static key issued by Product, sent as `x-api-key` |
| **RSA Private Key** | PKCS#8 hex-encoded key for request signing |
| **Basic Auth Credentials** | Username and password issued by Product |
| **Agent Framework** | OpenClaw or any OpenAI-function-compatible framework |

---

## Quick Start

### 1 — Setup

1 — Configure Credentials
Store all secrets as environment variables. Never hardcode them.

```bash
PRODUCT_ENDPOINT_URL=https://api.changelly.com/v2/
PRODUCT_PRIVATE_KEY_HEX=308204bc020100300d06092a864886...
PRODUCT_API_KEY=EKikACbkQiYNConmbByoWqQW1=
PRODUCT_BASIC_AUTH_USER=ba_username
PRODUCT_BASIC_AUTH_PASS=ba_password
```

2 — Add the Skill to Your Agent
Copy skill.md into your agent's skills directory and register it.

OpenClaw example:

```bash
/my-agent
  /skills
    skill.md        ← drop it here
  agent.config.json

{
  "skills": [
    "skills/skill.md"
  ]
}
```

Your agent now knows how to authenticate, sign, call, and handle responses
from the Product API without any additional code.

3 — Run Your Agent

openclaw run --config agent.config.json
The agent is now ready to accept user instructions and execute crypto trading
operations through the Product API.

What the Agent Can Do
Once the skill is loaded, the agent is capable of handling the following
Example of request:

Crypto Trading

User Instruction                              Agent Action
"Swap 0.5 BTC to USDC"	                      Calls createTransaction with the given pair and amount
"Exchange ETH to BTC, send to address 0x123"	Builds and signs a createTransaction call

User:   Swap 1 BTC to USDC, send to 0x28c6c06298d514db08993407bf21d60

Agent:  I tried to create the transaction but the amount of 1 BTC exceeds
        the current maximum of 0.64342124 BTC for this pair.
        The valid range is 0.00023742 BTC – 0.64342124 BTC.
        Would you like me to proceed with 0.64 BTC instead?

User:   Yes, go ahead.

Agent:  Done. Transaction created successfully.
        ID: 3c6a5467784e4088ac41abe1c68abc29b9ae695b074a734f13aa2d5016bb26d8 | Status: pending

The Skill File
skill.md is the single file that gives the agent everything it needs to work
with the Product API. It is written in the OpenClaw skill format and contains:

Section	What It Provides
Inputs / Outputs	Typed parameter definitions the agent maps user intent onto
Steps	Ordered, unambiguous instructions for building and sending each request
Implementation	A ready-to-run JavaScript reference the agent can execute
Example Invocation	A concrete JSON example the agent can pattern-match against
You do not need to write any custom tool code. Loading skill.md is sufficient
for a compatible agent to start using the Product API.
