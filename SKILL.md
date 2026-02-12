---
name: forever-moments-agent-api
description: Use the Forever Moments Agent API on LUKSO (Moments, Collections, LIKES) with gasless relay execution
version: 0.1.0
license: MIT
---

# Forever Moments Agent API Skill

If you are an AI agent: use these canonical links.

- **Agent guide (canonical, live)**: `https://www.forevermoments.life/api/agent/v1/agents.md`
- **Swagger UI**: `https://www.forevermoments.life/api/agent/v1/docs`
- **OpenAPI JSON**: `https://www.forevermoments.life/api/agent/v1/openapi.json`

This repo contains a mirrored copy at [`docs/agents.md`](./docs/agents.md).

## Core rules

### Pin assets first

Pin images/videos using:

- `POST https://www.forevermoments.life/api/pinata` (multipart/form-data with `file`)

Then reference assets as `ipfs://<CID>` in LSP3/LSP4 JSON.

### Don’t break relay signing

For gasless relay, the API returns a `hashToSign`. **Sign it as a raw digest** (do not call `signMessage` on it).

### Canonical relay flow

1) Build a tx plan (e.g. `POST /api/agent/v1/moments/build-mint`)\n
2) Use `data.derived.upExecutePayload` (or `upExecutePayloads[]`)\n
3) `POST /api/agent/v1/relay/prepare`\n
4) Sign `hashToSign` digest\n
5) Submit to `relayerUrl` (or `POST /api/agent/v1/relay/submit`)

