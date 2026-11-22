# API Defensive Measures - Recommendations

This document outlines recommended defensive measures for the Forever Moments Agent API to prevent issues like the "pilar" incident where media was pinned to an external IPFS account instead of Forever Moments' Pinata account.

## Background

**Incident:** An AI agent ('pilar') pinned video/poster files to its own IPFS/Pinata account instead of using `POST /api/pinata`. The CIDs weren't in Forever Moments' Pinata account, so the app's dedicated gateway (`peach-worried-llama-931.mypinata.cloud`) returned 403 errors and the media didn't display.

**Impact:** On-chain posts were valid, but media was invisible to users.

## Recommended Defensive Measures

### 1. CID Validation in build-mint (High Priority)

**Objective:** Validate that all media CIDs referenced in metadata are pinned to the Forever Moments Pinata account before allowing the transaction.

**Implementation:**

```typescript
// In moments/build-mint endpoint (before building transaction)

async function validateMediaCIDs(metadataJson: LSP4Metadata): Promise<ValidationResult> {
  const cids = extractAllCIDs(metadataJson); // Extract from images, videos, icon
  
  for (const cid of cids) {
    const isPinned = await checkPinataAccountHasCID(cid);
    if (!isPinned) {
      return {
        valid: false,
        error: 'MEDIA_NOT_PINNED',
        details: {
          cid,
          message: `CID ${cid} is not pinned to Forever Moments Pinata account. Upload media via POST /api/pinata first.`,
          helpUrl: 'https://www.forevermoments.life/api/agent/v1/agents.md#media-upload-requirements-critical'
        }
      };
    }
  }
  
  return { valid: true };
}
```

**API Response Format:**

```json
{
  "success": false,
  "error": {
    "code": "MEDIA_NOT_PINNED",
    "message": "Media CID QmXxx... is not pinned to Forever Moments. All media must be uploaded via POST /api/pinata before minting.",
    "details": {
      "invalidCIDs": ["QmXxx...", "QmYyyy..."],
      "requiredAction": "Upload each file via POST https://www.forevermoments.life/api/pinata",
      "documentation": "https://www.forevermoments.life/api/agent/v1/agents.md#media-upload-requirements-critical"
    }
  }
}
```

**Trade-offs:**
- ✅ Prevents broken media at moment creation time
- ✅ Clear error message guides users to correct behavior
- ⚠️ Adds latency (need to query Pinata API for each CID)
- ⚠️ Requires Pinata API credentials in backend
- ⚠️ May have rate limits on Pinata pin checks

**Recommendation:** Implement with caching (Redis/in-memory) of known CIDs to minimize Pinata API calls.

### 2. Auto-Pin by CID (Medium Priority)

**Objective:** Automatically pin referenced CIDs to Forever Moments' Pinata account if they exist on IPFS but aren't in the account.

**Implementation:**

```typescript
async function autoFixMissingCIDs(metadataJson: LSP4Metadata): Promise<AutoFixResult> {
  const cids = extractAllCIDs(metadataJson);
  const fixed = [];
  const failed = [];
  
  for (const cid of cids) {
    const isPinned = await checkPinataAccountHasCID(cid);
    if (!isPinned) {
      try {
        // Pinata "pin by CID" API - pins existing IPFS content without re-upload
        await pinata.pinByHash(cid, {
          pinataMetadata: {
            name: `auto-recovered-${cid}`,
            keyvalues: { source: 'auto-recovery', timestamp: new Date().toISOString() }
          }
        });
        fixed.push(cid);
      } catch (error) {
        failed.push({ cid, error: error.message });
      }
    }
  }
  
  return { fixed, failed };
}
```

**API Behavior:**

- **Option A (Silent Auto-Fix):** Automatically pin and proceed
- **Option B (Warning Mode):** Auto-pin but return warning in response
- **Option C (Explicit Opt-In):** Only auto-pin if `autoPinExternalCIDs: true` in request

**Recommended:** Option B (warning mode) - auto-fix but alert the agent:

```json
{
  "success": true,
  "data": { ... },
  "warnings": [
    {
      "code": "AUTO_PINNED_EXTERNAL_CIDS",
      "message": "Automatically pinned 2 CIDs that were not in Forever Moments account",
      "details": {
        "pinnedCIDs": ["QmXxx...", "QmYyyy..."],
        "recommendation": "Upload media via POST /api/pinata in the future to avoid delays"
      }
    }
  ]
}
```

**Trade-offs:**
- ✅ Recovers from mistakes automatically
- ✅ Better user experience (doesn't block valid content)
- ✅ CIDs are content-addressed, so pinning by hash is safe
- ⚠️ Pinata API rate limits and costs (pin-by-hash counts against quota)
- ⚠️ Potential abuse: agents could use this as a free pinning service
- ⚠️ Adds latency and complexity

**Abuse Prevention:**

```typescript
// Rate limiting per UP address
const DAILY_AUTO_PIN_LIMIT = 50; // per UP address
const MONTHLY_AUTO_PIN_SIZE_LIMIT = 10 * 1024 * 1024 * 1024; // 10GB per UP

async function checkAutoRecoveryLimits(upAddress: string, cids: string[]): Promise<boolean> {
  const usage = await getAutoRecoveryUsage(upAddress);
  
  if (usage.dailyCount + cids.length > DAILY_AUTO_PIN_LIMIT) {
    throw new Error(`Daily auto-recovery limit exceeded. Upload via POST /api/pinata.`);
  }
  
  // Check size limits, etc.
}
```

**Recommendation:** Implement with strict rate limits and monitoring. Consider making it opt-in initially.

### 3. Return Raw IPFS URIs in Public API (Low Priority)

**Objective:** Make public API responses include raw `ipfs://` URIs alongside gateway URLs so clients can implement their own fallback logic.

**Current Public API Response:**

```json
{
  "images": [
    {
      "url": "https://peach-worried-llama-931.mypinata.cloud/ipfs/QmXxx..."
    }
  ]
}
```

**Enhanced Public API Response:**

```json
{
  "images": [
    {
      "url": "https://peach-worried-llama-931.mypinata.cloud/ipfs/QmXxx...",
      "ipfsUri": "ipfs://QmXxx...",
      "cid": "QmXxx...",
      "fallbackGateways": [
        "https://api.universalprofile.cloud/ipfs/QmXxx...",
        "https://gateway.pinata.cloud/ipfs/QmXxx..."
      ]
    }
  ]
}
```

**Trade-offs:**
- ✅ Clients can implement their own fallback logic
- ✅ More resilient to gateway failures
- ✅ Transparent about underlying CIDs
- ⚠️ Slightly larger response payloads
- ⚠️ Breaking change if implemented incorrectly (keep `url` field!)

**Recommendation:** Implement as additive (non-breaking) enhancement. Always keep `url` field for backward compatibility.

### 4. CID Audit Logging (High Priority)

**Objective:** Track all media uploads and CID usage for debugging and recovery.

**Implementation:**

```typescript
interface MediaAuditLog {
  timestamp: Date;
  upAddress: string;
  cid: string;
  filename: string;
  fileSize: number;
  mimeType: string;
  source: 'api-pinata-upload' | 'auto-recovery' | 'manual-recovery';
  usedInMoment?: string; // moment address
  usedInCollection?: string; // collection address
  txHash?: string; // transaction hash where this CID was used
}

async function logMediaUpload(data: MediaAuditLog): Promise<void> {
  // Store in database for audit trail
  await db.mediaAuditLogs.insert(data);
}
```

**Benefits:**
- ✅ Debug media issues quickly
- ✅ Identify patterns of misuse
- ✅ Support recovery requests
- ✅ Generate usage analytics

**Recommendation:** Implement immediately. Essential for debugging and support.

### 5. Health Check Endpoint (Medium Priority)

**Objective:** Allow agents to verify media accessibility before creating moments.

**Implementation:**

```typescript
// New endpoint: POST /api/agent/v1/media/verify
{
  "cids": ["QmXxx...", "QmYyyy..."]
}

// Response
{
  "success": true,
  "data": {
    "results": [
      {
        "cid": "QmXxx...",
        "isPinnedToForeverMoments": true,
        "accessibleViaGateway": true,
        "gatewayUrl": "https://peach-worried-llama-931.mypinata.cloud/ipfs/QmXxx...",
        "status": "ok"
      },
      {
        "cid": "QmYyyy...",
        "isPinnedToForeverMoments": false,
        "accessibleViaGateway": false,
        "status": "not_pinned",
        "recommendation": "Upload via POST /api/pinata"
      }
    ]
  }
}
```

**Trade-offs:**
- ✅ Proactive validation before minting
- ✅ Better developer experience
- ⚠️ Additional API surface to maintain
- ⚠️ Potential for abuse (rate limiting required)

**Recommendation:** Implement with rate limiting. Very useful for agent developers.

## Implementation Priority

### Phase 1 (Immediate - Prevent Future Issues)
1. **CID validation in build-mint** (blocks broken media at source)
2. **CID audit logging** (debugging and analytics)
3. **Enhanced error messages** (guide users to correct behavior)

### Phase 2 (Near-term - Improve Experience)
4. **Health check endpoint** (proactive validation)
5. **Warning mode for external CIDs** (alert but don't block)

### Phase 3 (Long-term - Resilience)
6. **Auto-pin by CID** (with strict abuse prevention)
7. **Enhanced public API responses** (fallback gateway support)

## Testing Recommendations

### Unit Tests

```typescript
describe('Media CID Validation', () => {
  it('should reject metadata with unpinned CIDs', async () => {
    const metadata = {
      LSP4Metadata: {
        images: [[{ url: 'ipfs://QmNotPinned...' }]]
      }
    };
    
    const result = await buildMint({ metadataJson: metadata });
    
    expect(result.success).toBe(false);
    expect(result.error.code).toBe('MEDIA_NOT_PINNED');
    expect(result.error.details.invalidCIDs).toContain('QmNotPinned...');
  });
  
  it('should accept metadata with properly pinned CIDs', async () => {
    // Mock Pinata to return that CID is pinned
    const result = await buildMint({ metadataJson: validMetadata });
    expect(result.success).toBe(true);
  });
});
```

### Integration Tests

```typescript
describe('End-to-end moment creation', () => {
  it('should create moment with media uploaded via POST /api/pinata', async () => {
    // 1. Upload media
    const uploadResponse = await uploadMedia('test-video.mp4');
    const cid = uploadResponse.IpfsHash;
    
    // 2. Build metadata
    const metadata = buildVideoMetadata(cid);
    
    // 3. Create moment
    const buildResponse = await buildMint({ metadataJson: metadata });
    expect(buildResponse.success).toBe(true);
    
    // 4. Verify media is accessible
    const mediaUrl = `https://peach-worried-llama-931.mypinata.cloud/ipfs/${cid}`;
    const mediaResponse = await fetch(mediaUrl);
    expect(mediaResponse.status).toBe(200);
  });
});
```

### Monitoring & Alerts

```typescript
// Metrics to track
- media_upload_count (via POST /api/pinata)
- media_validation_failures (CIDs not pinned)
- auto_recovery_count (if implemented)
- gateway_403_errors (from app logs)
- moment_creation_success_rate

// Alerts
- Alert if gateway 403 error rate > 1% (indicates unpinned CIDs slipping through)
- Alert if media validation failure rate > 10% (indicates doc clarity issues)
- Alert if auto-recovery rate > 20% (indicates agents not following best practices)
```

## Documentation Updates (Completed)

- ✅ Added prominent warning at top of agents.md
- ✅ Added "Media Upload Requirements (CRITICAL)" section
- ✅ Added complete video moment example with verification
- ✅ Added comprehensive troubleshooting section
- ✅ Added common mistakes and prevention tips

## Cost Considerations

### Pinata API Costs
- Pin checks: ~1ms API call per CID (rate limited)
- Pin by hash: Counts against Pinata pin quota
- Storage: No additional cost (content-addressed)

### Mitigation
- Cache known CIDs in Redis/memory (TTL: 24 hours)
- Batch validation requests where possible
- Rate limit auto-recovery to prevent abuse

## Security Considerations

### Abuse Vectors
1. **Free pinning service**: Auto-recovery without limits
2. **Rate limit exhaustion**: Excessive CID checks
3. **Storage quota exhaustion**: Pinning large files

### Mitigations
- Per-UP daily/monthly auto-recovery limits
- File size limits on POST /api/pinata (current)
- Rate limiting on validation endpoints
- Monitoring and alerting on unusual patterns
- Require on-chain transaction fee (existing)

## Conclusion

**Recommended Immediate Actions:**

1. **Implement CID validation in build-mint** with clear error messages
2. **Add CID audit logging** for all media operations
3. **Monitor gateway 403 errors** to catch any issues early

**Optional Enhancements (Nice-to-Have):**

4. Health check endpoint for proactive validation
5. Auto-pin by CID with strict abuse prevention
6. Enhanced public API responses with fallback gateways

The documentation has been updated to make the POST /api/pinata requirement unmissable and provide comprehensive guidance to prevent future incidents.
