## Forever Moments Public API (v1)

This file is a **mirrored copy** of the canonical Public API guide served by the app at:

- `https://www.forevermoments.life/api/public/v1/README.md`

If this file ever drifts, prefer the live canonical version above.

---

## Forever Moments Public API (v1)

This API is a **stable, public, read-only** surface for building third-party frontends on top of Forever Moments contracts and indexed data.

### Base URLs

- **OpenAPI JSON**: `/api/public/v1/openapi.json`
- **Swagger UI**: `/api/public/v1/docs`
- **This guide**: `/api/public/v1/README.md`

### Response format

- **Success**:
  - `success: true`
  - `data`: payload
  - `meta`: pagination/diagnostics (optional)
- **Error**:
  - `success: false`
  - `error.code`, `error.message`

### CORS

All public endpoints include `Access-Control-Allow-Origin: *`, so browser-based frontends can call them directly.

### Pagination

Most list endpoints use:

- `limit` (max 100)
- `offset` (0-based)

### Collections

List collections:

```bash
curl "https://your-domain.com/api/public/v1/collections?limit=21&offset=0&sortBy=most-recent"
```

Count only:

```bash
curl "https://your-domain.com/api/public/v1/collections?countOnly=true"
```

Fetch by ids:

```bash
curl -X POST "https://your-domain.com/api/public/v1/collections/by-ids" \
  -H "Content-Type: application/json" \
  -d '{"collectionIds":["0xabc...","0xdef..."]}'
```

### Moments

List moments:

```bash
curl "https://your-domain.com/api/public/v1/moments?limit=21&offset=0&sortBy=most-recent"
```

Filter by owner:

```bash
curl "https://your-domain.com/api/public/v1/moments?owner=0x1234...&limit=21&offset=0"
```

Moment detail:

```bash
curl "https://your-domain.com/api/public/v1/moments/0xMomentAddress..."
```

Moment likes:

```bash
curl "https://your-domain.com/api/public/v1/moments/0xMomentAddress.../likes?limit=20&offset=0"
```

### Profiles

Batch fetch profile cards:

```bash
curl "https://your-domain.com/api/public/v1/profiles?addresses=0xabc...,0xdef..."
```

Profile detail:

```bash
curl "https://your-domain.com/api/public/v1/profiles/0xabc..."
```

Profile moments:

```bash
curl "https://your-domain.com/api/public/v1/profiles/0xabc.../moments?limit=20&offset=0"
```

### Marketplace

Currently listed moments:

```bash
curl "https://your-domain.com/api/public/v1/marketplace/for-sale?limit=50&sortBy=listed-recently"
```

### Likes stats

```bash
curl "https://your-domain.com/api/public/v1/likes/stats"
```

