# @accensa/sdk

Middleware hook and receipt verification for [Accensa](https://accensa.github.io/accensa-app/), the merchant back-office for x402 sellers on Stellar.

## Installation

```bash
npm install @accensa/sdk
# or
pnpm add @accensa/sdk
# or
yarn add @accensa/sdk
```

**Runtime:** Node.js 18+. The SDK uses `node:crypto` for Ed25519 signing and SHA-256 hashing. Browser environments are not supported for settlement reporting (signing requires Node.js crypto), but the merkle verification module can run in any JavaScript environment that provides `crypto.createHash`.

**Peer dependency:** `express` (optional) — only needed if you use `attachAccensaHook`.

## Quick Start

### Using `createSettleHook` (recommended)

If your server uses `@x402/core`'s resource server directly, this is the preferred integration. The settle result arrives from the facilitator as ground truth:

```ts
import { createSettleHook } from '@accensa/sdk';

resourceServer.onAfterSettle(
  createSettleHook({
    indexerUrl: 'https://your-accensa-dashboard.vercel.app',
    privateKeyHex: process.env.ACCENSA_PRIVATE_KEY_HEX,
  }),
);
```

### Using `attachAccensaHook` (Express middleware)

For Express-based x402 sellers. Reads the settlement from the `X-PAYMENT-RESPONSE` header after the response finishes:

```ts
import express from 'express';
import { attachAccensaHook } from '@accensa/sdk';

const app = express();

// Mount AFTER your x402 payment middleware
app.use(
  attachAccensaHook({
    indexerUrl: 'https://your-accensa-dashboard.vercel.app',
    privateKeyHex: process.env.ACCENSA_PRIVATE_KEY_HEX,
  }),
);
```

### Verifying a Merkle Receipt

```ts
import { verifyReceipt } from '@accensa/sdk/merkle';

const isValid = verifyReceipt(leaf, proof, root);
```

## API Reference

### `createSettleHook(options)`

Builds an `onAfterSettle` handler for an x402 resource server.

**Options (`SettleHookOptions`):**

| Property        | Type                        | Required | Description                             |
| --------------- | --------------------------- | -------- | --------------------------------------- |
| `indexerUrl`    | `string`                    | Yes      | Base URL of your Accensa deployment     |
| `privateKeyHex` | `string`                    | Yes      | Ed25519 private key in hex format       |
| `timeoutMs`     | `number`                    | No       | Report timeout in ms (default: 5000)    |
| `method`        | `string`                    | No       | HTTP method to attribute (default: GET) |
| `onError`       | `(error, payload?) => void` | No       | Error handler (default: console.error)  |
| `fetchImpl`     | `typeof fetch`              | No       | Custom fetch implementation             |

### `attachAccensaHook(options)`

Returns Express middleware that reports route attribution for x402-paid requests.

Accepts all `SettleHookOptions` plus:

| Property    | Type                    | Required | Description                    |
| ----------- | ----------------------- | -------- | ------------------------------ |
| `attribute` | `(req) => RequestFacts` | No       | Custom route attribution logic |

### `verifyReceipt(leaf, proof, root)`

Verifies a payment receipt against an anchored Merkle batch root using sorted-pair SHA-256 hashing.

- `leaf` — hex-encoded 32-byte hash of the receipt
- `proof` — hex-encoded 32-byte sibling hashes, leaf-to-root order
- `root` — hex-encoded 32-byte Merkle root anchored on-chain

Returns `true` if the proof is valid, `false` otherwise. Throws if any input is not a valid hex-encoded 32-byte value.

### `parseSettlementHeader(header)`

Decodes the base64-encoded `X-PAYMENT-RESPONSE` header into an `X402SettleResult` object. Returns `null` for absent, malformed, or unparseable headers.

### `settlementFromResult(result, facts)`

Combines an x402 settle result with request facts into a `Settlement` object. Returns `null` unless the settlement succeeded and carries a transaction hash.

### `routeFromResourceUrl(url)`

Extracts the pathname from an x402 resource URL for route attribution.

## Signing Contract

The settlement report is authenticated with Ed25519 signatures. The SDK handles this automatically, but non-JS implementers must:

1. **Construct the JSON payload** — a JSON object with `tx_hash`, `route`, `method`, and optional fields (`request_id`, `payer`, `amount`, `network`, `reported_at`).
2. **Sign the raw request body** — the Ed25519 signature is over the exact UTF-8 bytes of the JSON string. The bytes signed must match the body sent in the HTTP request exactly.
3. **Set the header** — pass the hex-encoded signature in the `X-Signature` HTTP header.

The backend verifies this signature before parsing the JSON.

## Wire Contract

The report payload (`SettleHookPayload`) and Ed25519 signature scheme are a wire contract between the SDK and the Accensa indexer. Changes to the payload fields or signing mechanism are **breaking changes** and require a major version bump. See [CONTRIBUTING.md](../../CONTRIBUTING.md) for the semver policy.

## License

MIT
