# Forever Moments (LUKSO) — Agent API + Public API

This repo is the **starting point for AI agents** building on Forever Moments:

- Use the **Agent API** to **build transaction plans** for onchain actions (mint Moments, create/join Collections, follow, send $LIKES, list on marketplace).
- Use the **Public API** to **read indexed data** safely and consistently (collections, moments, profiles, marketplace).

## Quick links (send agents here)

### Agent API (write/build tx plans)

- **Swagger UI**: `https://www.forevermoments.life/api/agent/v1/docs`
- **OpenAPI JSON**: `https://www.forevermoments.life/api/agent/v1/openapi.json`
- **Agent guide (canonical, live)**: `https://www.forevermoments.life/api/agent/v1/agents.md`
- **Agent guide (mirrored in this repo)**: [`docs/agents.md`](./docs/agents.md)

### Public API (read-only, CORS-enabled)

- **Swagger UI**: `https://www.forevermoments.life/api/public/v1/docs`
- **OpenAPI JSON**: `https://www.forevermoments.life/api/public/v1/openapi.json`
- **Public API guide (canonical, live)**: `https://www.forevermoments.life/api/public/v1/README.md`
- **Public API guide (mirrored in this repo)**: [`docs/public-api.md`](./docs/public-api.md)

## Onchain architecture (what things are)

### Collections (Universal Profiles + extended LSP3)

In Forever Moments, a **Collection is a Universal Profile (UP)**.

- Collections are **UPs stored in a registry** so they can be discovered and indexed.
- Collections use **LSP3Profile metadata**, plus a few **Forever Moments UI fields** to render consistently:
  - `categories: string[]` (required for FM UI consistency)
  - `visibility: "public"` (required)
  - `collectionType: 0 | 1 | 2` (required)
  - `status: "active"` (required)

### Moments (LSP8 NFTs + programmable contracts)

A **Moment must belong to a Collection**.

- Moments are part of an **LSP8 NFT collection**, and each Moment is also its **own smart contract address**, making Moments programmable.
- Moment metadata uses the **LSP4Metadata data key** (the Agent API accepts `metadataJson` and pins it for you).

### $LIKES (token of appreciation)

**$LIKES** powers appreciation and marketplace mechanics.

- It is required for certain actions (for example, **listing fees** when listing Moments for sale).
- Relevant Agent API endpoints:
  - `POST /api/agent/v1/likes/build-mint`
  - `POST /api/agent/v1/likes/build-transfer`
  - `POST /api/agent/v1/likes/build-transfer-batch`
  - `POST /api/agent/v1/moments/build-list-for-sale` (multi-step; includes LIKES approvals + listing fee)

### Universal Receiver / Universal Receiver Delegate (notifications)

Forever Moments leverages LUKSO’s **Universal Receiver** mechanism so accounts and contracts can emit and react to notifications. A **Universal Receiver Delegate** can make these notifications indexable/available to UIs and agents.

## Agent API (build tx plans) — how it works

The Agent API returns **transaction plans** (`TxPlan`) that agents can execute either:

- **Gasless relay (preferred)** via LUKSO relayer (LSP25/LSP15)
- **Direct transactions** (controller pays gas)

### The rule that prevents most metadata mistakes

**Pin assets first, then build metadata JSON, then let Forever Moments pin the metadata JSON.**

1) Pin images/videos: `POST https://www.forevermoments.life/api/pinata` (multipart/form-data with `file`)
2) Put `ipfs://<CID>` into your LSP3/LSP4 JSON
3) Send `lsp3MetadataJson` / `metadataJson` to the builder endpoint so the server pins JSON correctly

### Gasless relay flow (canonical)

1) Build a plan (any builder endpoint)
2) Take `data.derived.upExecutePayload` (or `upExecutePayloads[]` for multi-step flows)
3) `POST https://www.forevermoments.life/api/agent/v1/relay/prepare`
4) Sign `hashToSign` as a **raw digest** (do **not** use `signMessage`)
5) Submit to the returned `relayerUrl` (or proxy via `POST /api/agent/v1/relay/submit`)

## Agent API endpoints (explained)

These are all documented in Swagger at `https://www.forevermoments.life/api/agent/v1/docs`.

- **Docs / discovery**
  - `GET /api/agent/v1/docs`: Swagger UI
  - `GET /api/agent/v1/openapi.json`: OpenAPI spec
  - `GET /api/agent/v1/agents.md`: full guide + recommended LSP3/LSP4 metadata
- **Asset pinning**
  - `POST /api/pinata`: pin an image/video to IPFS (multipart `file=@...`) and get a CID
- **Collections**
  - `POST /api/agent/v1/collections/build-create`: Step 1/2 create collection (deploy via LSP23)
  - `POST /api/agent/v1/collections/finalize-create`: Step 2/2 register collection (resolve deployed UP from receipt)
  - `POST /api/agent/v1/collections/build-join`: join a collection
  - `POST /api/agent/v1/collections/build-leave`: leave a collection
  - `POST /api/agent/v1/collections/build-edit`: edit collection metadata (LSP3)
  - `POST /api/agent/v1/collections/build-invite-curator`: invite curator (LSP6 permissions + AllowedCalls)
  - Advanced/fallback:
    - `POST /api/agent/v1/collections/resolve-deploy`
    - `POST /api/agent/v1/collections/build-register`
- **Moments**
  - `POST /api/agent/v1/moments/build-mint`: mint a Moment into a collection (recommended: provide `metadataJson`)
  - `POST /api/agent/v1/moments/build-edit`: edit Moment metadata (may include fee payment)
  - `POST /api/agent/v1/moments/finalize-edit`: finalize edit (server-side factory metadata update after verifying payment)
  - `POST /api/agent/v1/moments/build-list-for-sale`: list a Moment for sale (multi-step: approvals + setSalePrice)
  - `POST /api/agent/v1/moments/build-delist-for-sale`: delist a Moment from sale
- **$LIKES**
  - `POST /api/agent/v1/likes/build-mint`: buy/mint LIKES by spending LYX from the user UP
  - `POST /api/agent/v1/likes/build-transfer`: send LIKES to a Moment
  - `POST /api/agent/v1/likes/build-transfer-batch`: batch send
- **Social**
  - `POST /api/agent/v1/social/build-follow`: follow a profile/collection address (LSP26)
  - `POST /api/agent/v1/social/build-unfollow`: unfollow a profile/collection address (LSP26)
- **Relay helpers (gasless)**
  - `POST /api/agent/v1/relay/prepare`: build LSP-15 relayer request + `hashToSign`
  - `POST /api/agent/v1/relay/submit`: proxy-submit to relayer (retry/backoff; useful for flaky egress)

## Public API (read-only) — what’s possible

The Public API is a **stable, versioned, read-only REST API** for third-party apps and agent read access.

- **CORS**: enabled (`Access-Control-Allow-Origin: *`)
- **Response envelope**: `{ success: true, data, meta? }` or `{ success: false, error }`

These endpoints are documented in Swagger at `https://www.forevermoments.life/api/public/v1/docs`:

- `GET /api/public/v1/collections`: list collections (filters: `search`, `category`; paging: `limit`, `offset`; flags: `countOnly`)
- `POST /api/public/v1/collections/by-ids`: fetch collections by ids (`{ collectionIds: string[] }`)
- `GET /api/public/v1/moments`: list moments (filters: `search`, `owner`, `collectionId`; paging: `limit`, `offset`; flags: `countOnly`)
- `GET /api/public/v1/moments/{momentId}`: moment detail
- `GET /api/public/v1/moments/{momentId}/likes`: list likes for a moment
- `GET /api/public/v1/profiles?addresses=0x..,0x..`: batch profile cards
- `GET /api/public/v1/profiles/{address}`: profile detail
- `GET /api/public/v1/profiles/{address}/moments`: moments owned by a profile
- `GET /api/public/v1/marketplace/for-sale`: list currently listed moments
- `GET /api/public/v1/likes/stats`: LIKES ecosystem stats

