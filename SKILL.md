---
name: tally-podcraft
description: Generate AI podcasts via the Podcraft API using x402 payment protocol. Use when asked to "generate a podcast", "create a podcast", "make a podcast about", "podcraft", or when the user wants to create an AI-generated podcast episode on any topic. Handles payment via x402 (USDC on Base) and polls for completion. Compatible with Coinbase Agentic Wallets.
allowed-tools: Bash(curl:*), Bash(node:*), Read, Write, WebFetch
---

# Tally Podcraft: AI Podcast Generation via x402

Generate AI podcast episodes on any topic by paying with USDC on Base mainnet via the [x402 protocol](https://www.x402.org/). Compatible with [Coinbase Agentic Wallets](https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets) for autonomous agent-to-agent commerce.

## API Endpoint

**Production**: `https://api-production-5b87.up.railway.app`

### POST /v1/generate (payment-gated)

Requires x402 payment header. Returns a job ID to poll.

**Request body:**
```json
{
  "topic": "Bitcoin",
  "durationMinutes": 5,
  "hostName": "Alex",
  "researchDepth": 3
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| topic | string | Yes | — | Podcast topic (1-1000 chars) |
| durationMinutes | int | No | 5 | Episode length (1-30 min) |
| hostName | string | No | — | Host persona name |
| researchDepth | int | No | 3 | Research depth (1-5) |
| researchModel | string | No | — | LLM for research |
| scriptModel | string | No | — | LLM for script |
| voiceId | string | No | — | ElevenLabs voice ID |

**Success response (202):**
```json
{
  "data": {
    "jobId": "cmm2ho89000098oqc...",
    "seriesId": "cmm2ho8a000108oqc...",
    "episodeId": "cmm2ho8b000118oqc..."
  }
}
```

**Without payment (402):**
Returns `402 Payment Required` with `payment-required` header containing x402 payment requirements (Base64-encoded JSON).

### GET /v1/generate/:jobId (free)

Poll for job status. No payment or auth needed.

**Response:**
```json
{
  "data": {
    "jobId": "cmm2ho89...",
    "status": "RUNNING",
    "progress": 45,
    "progressNote": "Generating audio...",
    "error": null,
    "audioUrl": null,
    "createdAt": "2026-02-25T...",
    "completedAt": null
  }
}
```

| Status | Meaning |
|--------|---------|
| PENDING | Queued, not started |
| RUNNING | In progress (check progress %) |
| COMPLETED | Done — audioUrl will be set |
| FAILED | Error — check error field |

The `Retry-After: 5` header is set for in-progress jobs.

## Payment Details

- **Price**: $5.00 USDC per podcast
- **Network**: Base mainnet (eip155:8453)
- **Asset**: USDC (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913)
- **Timeout**: 300 seconds

## Using x402 Client (v2)

The server uses x402 **v2** protocol. Use `@x402/core` + `@x402/evm` (not the legacy `x402-axios`).

**Install:**
```bash
npm install @x402/core @x402/evm viem
```

**Pay and generate:**
```typescript
import { x402Client, x402HTTPClient } from "@x402/core/client";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { toClientEvmSigner } from "@x402/evm";
import { createPublicClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { base } from "viem/chains";

// Any private key with USDC on Base — no CDP account needed
const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
const publicClient = createPublicClient({ chain: base, transport: http() });
const signer = toClientEvmSigner(account, publicClient);

const coreClient = new x402Client();
registerExactEvmScheme(coreClient, { signer });
const httpClient = new x402HTTPClient(coreClient);

const url = "https://api-production-5b87.up.railway.app/v1/generate";
const body = JSON.stringify({ topic: "The History of Bitcoin" });
const headers = { "Content-Type": "application/json" };

// 1. Hit the endpoint — get 402
const res = await fetch(url, { method: "POST", body, headers });

// 2. Parse payment requirements from header
const paymentRequired = httpClient.getPaymentRequiredResponse(
  (name) => res.headers.get(name),
);

// 3. Sign payment (EIP-3009 USDC transfer authorization)
const payload = await httpClient.createPaymentPayload(paymentRequired);
const paymentHeaders = httpClient.encodePaymentSignatureHeader(payload);

// 4. Retry with payment proof — get 202
const paidRes = await fetch(url, {
  method: "POST",
  body,
  headers: { ...headers, ...paymentHeaders },
});

const { data } = await paidRes.json();
console.log(`Job ID: ${data.jobId}`);
```

## Using with Coinbase Agentic Wallets

This service is designed to work with [Coinbase Agentic Wallets](https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets) — wallet infrastructure built for AI agents that enables autonomous spending via x402.

An agent with an agentic wallet can:
1. Hit `POST /v1/generate` and receive a 402 challenge
2. Autonomously sign and submit USDC payment on Base
3. Retry with the payment proof to receive a job ID
4. Poll `GET /v1/generate/:jobId` until the podcast is ready

No human intervention required. The x402 protocol handles payment verification and settlement automatically.

## Polling for Completion

After getting a job ID, poll until completion:

```bash
# Poll every 5 seconds
while true; do
  RESULT=$(curl -s https://api-production-5b87.up.railway.app/v1/generate/$JOB_ID)
  STATUS=$(echo $RESULT | jq -r '.data.status')
  echo "Status: $STATUS ($(echo $RESULT | jq -r '.data.progressNote // "waiting"'))"

  if [ "$STATUS" = "COMPLETED" ]; then
    echo "Audio URL: $(echo $RESULT | jq -r '.data.audioUrl')"
    break
  elif [ "$STATUS" = "FAILED" ]; then
    echo "Error: $(echo $RESULT | jq -r '.data.error')"
    break
  fi

  sleep 5
done
```

## Testing Without Payment

To test the 402 flow without paying:

```bash
# Should return 402 with payment requirements
curl -v -X POST https://api-production-5b87.up.railway.app/v1/generate \
  -H "Content-Type: application/json" \
  -d '{"topic":"Test"}'

# Decode the payment-required header
curl -s -X POST https://api-production-5b87.up.railway.app/v1/generate \
  -H "Content-Type: application/json" \
  -d '{"topic":"Test"}' \
  -D - -o /dev/null 2>/dev/null | grep payment-required | cut -d' ' -f2 | base64 -d | jq .
```

## Notes

- Job IDs are CUIDs (~120 bits of entropy) — not guessable
- Audio URLs are signed and expire after 7 days
- Podcast generation takes several minutes (research, script writing, audio synthesis)
- The pipeline: topic research → script generation → TTS audio → signed URL
