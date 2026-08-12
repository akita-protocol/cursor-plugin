---
name: akita-connect
description: Use the Akita Connect protocol from `akita-sdk/connect` — a `liquid://` URI + request/response handshake for installing agents, plugins, and allowances on an ARC-58 wallet across devices (typically desktop→mobile via QR). Use when building a connect flow, generating/scanning a connect URI, constructing an `AgentInstallRequest`, handling the wallet-side response, or implementing the origin server that serves the request body. Strong triggers include "connect uri", "liquid://", "encodeConnectUri", "decodeConnectUri", "AgentInstallRequest", "ConnectRequest", "ConnectResponse", "AgentInstallResponse", "newAgentAccount", "cross-device agent install".
---

# Akita Connect

Akita Connect is a cross-device handoff protocol. A desktop app (or server) publishes a signed "install these plugins and allowances" request at a URL, encodes a small pointer into a `liquid://` URI, and renders it as a QR code. A mobile wallet scans the QR, fetches the request body from the publisher's origin, processes it locally, and posts back a response.

The SDK ships the URI codec and the TypeScript types. It does **not** implement the HTTP transport — that's the publisher's responsibility.

## Imports

```typescript
import {
  encodeConnectUri,
  decodeConnectUri,
} from 'akita-sdk/connect'

import type {
  AkitaConnectUri,
  AgentInstallRequest,
  AgentInstallPlugin,
  SerializedAllowance,
  SerializedMethodDefinition,
  ConnectRequest,
  ConnectResponse,
  AgentInstallResponse,
  ConnectErrorResponse,
} from 'akita-sdk/connect'
```

## URI format

```
liquid://<origin>?requestId=<uuid>
```

- Scheme: `liquid://`
- Host: the publisher's origin (e.g. `signal.akita.community`)
- Query: `requestId` (also accepted as `request_id` when decoding)

`AkitaConnectUri` shape:

```typescript
interface AkitaConnectUri {
  origin: string      // e.g. "signal.akita.community"
  requestId: string   // UUID
}
```

## Encode / decode

```typescript
const uri = encodeConnectUri({
  origin: 'signal.akita.community',
  requestId: 'abc-123',
})
// → "liquid://signal.akita.community?requestId=abc-123"

const parts = decodeConnectUri('liquid://signal.akita.community?requestId=abc-123')
// → { origin: "signal.akita.community", requestId: "abc-123" }
```

Notes:

- `decodeConnectUri` internally swaps `liquid://` for `https://` so it can reuse the `URL` constructor. It accepts `requestId` or `request_id`.
- `decodeConnectUri` throws on a malformed URI. Wrap it when handling untrusted input (QR scans, deep links).

## `AgentInstallRequest` — the payload

This is the body the mobile wallet fetches from `origin` after decoding the URI. It describes the agent to install and the plugin / allowance grants on the wallet.

```typescript
interface AgentInstallRequest {
  type: 'agent-install'
  v: 1 | 2
  agent: {
    name: string
    address: string                // Algorand account for the agent
  }
  network: 'localnet' | 'testnet' | 'mainnet'
  escrowName: string
  newAgentAccount: boolean         // true → wallet funds 4.352 ALGO to agent for MBR
  plugins: AgentInstallPlugin[]
  allowances?: SerializedAllowance[]  // v2: escrow-level (optional)
}
```

Field semantics:

| Field | Meaning |
|---|---|
| `type` | Discriminator — always `'agent-install'` |
| `v` | Protocol version (see below) |
| `agent.address` | The account identity the wallet will authorize |
| `newAgentAccount` | If true, the wallet sends 4.352 ALGO to `agent.address` to cover MBR |
| `escrowName` | Escrow the plugins + allowances bind to |
| `plugins` | Per-plugin install descriptors |
| `allowances` | v2 escrow-level allowances (preferred) |

## Versioning

`v: 1 | 2`.

- **v1**: allowances live inside each `AgentInstallPlugin.allowances[]`. Deprecated but still parsed.
- **v2**: allowances live at the request level (`AgentInstallRequest.allowances`), keyed by `{escrow, asset}`. Preferred.

No explicit negotiation — the publisher declares the version in the payload and the wallet must support at least up to that version.

## `AgentInstallPlugin`

```typescript
interface AgentInstallPlugin {
  id: string                                  // e.g. "payPlugin"
  coverFees?: boolean
  cooldown?: string                           // seconds; serialized bigint
  methods?: SerializedMethodDefinition[]
  allowances?: SerializedAllowance[]          // v1 only (deprecated)
}

interface SerializedMethodDefinition {
  name: string       // 4-byte ABI selector as hex, e.g. "a1b2c3d4"
  cooldown: string   // serialized bigint
}
```

## `SerializedAllowance`

A discriminated union of the three wallet allowance shapes, with every bigint serialized as a string for JSON transport.

```typescript
type SerializedAllowance =
  | { asset: string; type: 'flat';   amount: string; useRounds?: boolean }
  | { asset: string; type: 'window'; amount: string; interval: string; useRounds?: boolean }
  | { asset: string; type: 'drip';   rate: string; max: string; interval: string; useRounds?: boolean }
```

Why strings? JavaScript numbers can't round-trip bigints above 2^53 through JSON, and every Algorand amount and asset ID is a `uint64`. The wallet parses these back to bigints when installing.

## `ConnectRequest` and `ConnectResponse`

```typescript
type ConnectRequest = AgentInstallRequest   // alias; future request types may be added

type ConnectResponse = AgentInstallResponse | ConnectErrorResponse

interface AgentInstallResponse {
  type: 'agent-install'
  success: true
  walletAppId: string       // the wallet that installed
  escrowAddress: string     // created or confirmed escrow
  network: string
}

interface ConnectErrorResponse {
  type: 'error'
  success: false
  message: string
}
```

Discriminate by `type` or `success`.

## End-to-end flow

```
┌────────────┐       ┌────────────┐       ┌─────────────┐
│  Desktop   │       │  Origin    │       │   Mobile    │
│  / Server  │       │  Server    │       │   Wallet    │
└─────┬──────┘       └─────┬──────┘       └──────┬──────┘
      │                    │                     │
      │ 1. build request   │                     │
      │  AgentInstallReq   │                     │
      │                    │                     │
      │ 2. PUT /request/{uuid}                   │
      ├───────────────────►│                     │
      │                    │                     │
      │ 3. encodeConnectUri                      │
      │  + render QR                             │
      │                                          │
      │            4. scan QR, decodeConnectUri  │
      │            � │
      │                                          │
      │            4. scan QR, decodeConnectUri  │
      │            ◄─────────────────────────────┤
      │                                          │
      │            5. GET /request/{uuid}        │
      │                    ◄─────────────────────┤
      │                    │                     │
      │            6. process: install agent,    │
      │               plugins, allowances        │
      │                                          │
      │            7. POST /response/{uuid}      │
      │                  {AgentInstallResponse}  │
      │                    ◄─────────────────────┤
      │ 8. poll / push     │                     │
      │◄───────────────────┤                     │
```

Steps 2, 5, 7 are the publisher's responsibility. The SDK provides only the URI codec + types. No origin validation is built in — the wallet must enforce its own trusted-origin list when fetching the request body.

## Publisher-side example

```typescript
import { randomUUID } from 'crypto'
import { encodeConnectUri, type AgentInstallRequest } from 'akita-sdk/connect'

// 1. Build the request
const request: AgentInstallRequest = {
  type: 'agent-install',
  v: 2,
  agent: {
    name: 'Trading bot',
    address: agentAlgorandAddress,
  },
  network: 'mainnet',
  escrowName: 'trading-bot',
  newAgentAccount: true,
  plugins: [
    {
      id: 'payPlugin',
      coverFees: true,
      methods: [{ name: '12345678', cooldown: '0' }],
    },
  ],
  allowances: [
    {
      asset: '0',
      type: 'drip',
      rate: '1000000',       // 1 ALGO per interval
      max: '100000000',      // cap at 100 ALGO
      interval: '100',
      useRounds: true,
    },
  ],
}

// 2. Store at a URL the mobile wallet can fetch
const requestId = randomUUID()
await putRequestToOrigin(requestId, request)

// 3. Encode URI + render QR
const uri = encodeConnectUri({
  origin: 'signal.akita.community',
  requestId,
})
// → render `uri` as a QR code
```

## Wallet-side example

```typescript
import { decodeConnectUri, type AgentInstallRequest, type ConnectResponse } from 'akita-sdk/connect'

// 1. Scan + decode the QR payload
const { origin, requestId } = decodeConnectUri(scannedUri)

// 2. Enforce trusted origin (SDK does not do this)
if (!trustedOrigins.has(origin)) throw new Error(`Untrusted origin: ${origin}`)

// 3. Fetch the request body
const resp = await fetch(`https://${origin}/request/${requestId}`)
const request: AgentInstallRequest = await resp.json()

// 4. Parse serialized bigints
const allowances = (request.allowances ?? []).map(a => ({
  ...a,
  amount:   'amount'   in a ? BigInt(a.amount)   : undefined,
  interval: 'interval' in a ? BigInt(a.interval) : undefined,
  rate:     'rate'     in a ? BigInt(a.rate)     : undefined,
  max:      'max'      in a ? BigInt(a.max)      : undefined,
}))

// 5. Apply: create escrow, addPlugin for each, addAllowances, fund agent if newAgentAccount
//    (use the WalletSDK — see akita-wallet-sdk skill)

// 6. Send the response
const response: ConnectResponse = {
  type: 'agent-install',
  success: true,
  walletAppId: wallet.appId.toString(),
  escrowAddress: escrowAddr,
  network: request.network,
}
await fetch(`https://${origin}/response/${requestId}`, {
  method: 'POST',
  body: JSON.stringify(response),
})
```

## Common mistakes to avoid

- **Not validating `origin` before fetching** — `decodeConnectUri` parses untrusted input. Maintain an explicit allowlist on the wallet side.
- **Passing raw bigints in the request** — they won't survive JSON serialization. Convert to strings before `JSON.stringify` and convert back on the wallet side.
- **Using v1 allowance placement with `v: 2`** — put allowances on the request, not inside each plugin, or set `v: 1` if you really need per-plugin allowances.
- **Forgetting `newAgentAccount` funding math** — when true, the wallet sends 4.352 ALGO (MBR cushion) to the agent. Ensure the wallet has the funds available in the target escrow.
- **Reusing a `requestId`** — the origin must treat `requestId` as single-use; replayed IDs let an attacker re-trigger installs.
- **Trusting the `network` field without checking the wallet's current network** — warn or reject if the request's network doesn't match the wallet's.

## Reference

- URI codec: `akita-sdk/connect/uri.ts` (`encodeConnectUri`, `decodeConnectUri`)
- Types: `akita-sdk/connect/types.ts`
- Related wallet APIs: `akita-sdk/wallet` → `addPlugin`, `addAllowances`, `newEscrow` (see the `akita-wallet-sdk` skill)
