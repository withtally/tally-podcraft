# Tally Podcraft

A Claude Code skill for generating AI podcasts via the [x402 payment protocol](https://www.x402.org/). AI agents pay $5 USDC on Base to generate a podcast episode on any topic.

Built for [Coinbase Agentic Wallets](https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets) — the first wallet infrastructure designed for autonomous AI agents.

## What is this?

This is a **Claude Code skill** that enables AI agents to generate podcasts by paying with USDC on Base mainnet. It demonstrates agent-to-agent paid services using the x402 protocol (Coinbase's HTTP 402 payment standard).

**How it works:**

1. An agent calls `POST /v1/generate` with a topic
2. The server responds with `402 Payment Required` and x402 payment details
3. The agent (using an agentic wallet or x402 client) pays $5 USDC on Base
4. The server verifies payment via the Coinbase facilitator and starts podcast generation
5. The agent polls `GET /v1/generate/:jobId` until the podcast audio is ready

No API keys, no subscriptions, no accounts — just pay and generate.

## Installation

Copy the `SKILL.md` file into your Claude Code skills directory:

```bash
# Clone this repo
git clone https://github.com/withtally/tally-podcraft.git

# Copy to your Claude Code skills
cp -r tally-podcraft ~/.claude/skills/tally-podcraft
```

Or manually create `~/.claude/skills/tally-podcraft/SKILL.md` with the contents of [SKILL.md](./SKILL.md).

## Quick Test

Verify the endpoint is live and returning 402 challenges:

```bash
curl -s -X POST https://api-production-5b87.up.railway.app/v1/generate \
  -H "Content-Type: application/json" \
  -d '{"topic":"Bitcoin"}' \
  -w "\nHTTP Status: %{http_code}\n"
```

Expected: `HTTP Status: 402` with payment requirements in the `payment-required` header.

## Payment Details

| Field | Value |
|-------|-------|
| Price | $5.00 USDC |
| Network | Base mainnet (eip155:8453) |
| Asset | USDC (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`) |
| Protocol | [x402](https://www.x402.org/) |
| Facilitator | Coinbase CDP |

## Agentic Wallets

This service is designed to work with [Coinbase Agentic Wallets](https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets):

- **Autonomous payments**: Agents can pay for podcasts without human intervention
- **Built-in security**: Non-custodial wallets with session spending caps and per-transaction limits
- **x402 native**: The x402 protocol enables seamless machine-to-machine payments
- **Base mainnet**: Fast, cheap transactions on Coinbase's L2

An agent with an agentic wallet handles the entire flow — receiving the 402 challenge, signing the USDC payment, and retrying with payment proof — all autonomously.

## API Reference

See [SKILL.md](./SKILL.md) for the full API reference including:

- Request/response schemas
- x402 client SDK usage (TypeScript)
- Polling patterns for job completion
- Status codes and error handling

## Example: TypeScript with x402 Client

```bash
npm install @x402/core @x402/evm viem
```

```typescript
import { x402Client, x402HTTPClient } from "@x402/core/client";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { toClientEvmSigner } from "@x402/evm";
import { createPublicClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { base } from "viem/chains";

// Any wallet with USDC on Base — no CDP account needed
const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
const publicClient = createPublicClient({ chain: base, transport: http() });
const signer = toClientEvmSigner(account, publicClient);

const coreClient = new x402Client();
registerExactEvmScheme(coreClient, { signer });
const httpClient = new x402HTTPClient(coreClient);

const url = "https://api-production-5b87.up.railway.app/v1/generate";
const body = JSON.stringify({ topic: "The Future of AI Agents" });
const headers = { "Content-Type": "application/json" };

// 1. Get 402 challenge
const res = await fetch(url, { method: "POST", body, headers });
// 2. Sign USDC payment
const paymentRequired = httpClient.getPaymentRequiredResponse((n) => res.headers.get(n));
const payload = await httpClient.createPaymentPayload(paymentRequired);
const paymentHeaders = httpClient.encodePaymentSignatureHeader(payload);
// 3. Retry with payment proof → 202
const paidRes = await fetch(url, { method: "POST", body, headers: { ...headers, ...paymentHeaders } });
const { data } = await paidRes.json();
// 4. Poll: GET /v1/generate/{data.jobId}
```

## How Podcasts Are Generated

The pipeline behind the API:

1. **Research** — Deep web research on the topic using Tavily
2. **Script** — AI-generated podcast script with natural dialogue
3. **Audio** — Text-to-speech synthesis via ElevenLabs
4. **Delivery** — Signed audio URL (expires after 7 days)

Generation takes several minutes depending on episode length and research depth.

## License

MIT
