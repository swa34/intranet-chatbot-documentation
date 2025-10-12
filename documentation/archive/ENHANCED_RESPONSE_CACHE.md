# Enhanced Response Cache with Redis & Link Support

## Overview

The response cache system has been upgraded to use Redis for ultra-fast lookups while maintaining PostgreSQL for persistence and analytics. This dual-layer approach provides:

- **⚡ Sub-millisecond lookups** via Redis (primary cache)
- **💾 Persistent storage** via PostgreSQL (backup & analytics)
- **🔗 Full link support** for source citations
- **📊 Comprehensive analytics** for cache performance

## Architecture

```
User Query
    ↓
[Redis Cache] ← 1-10ms lookup
    ↓ (miss)
[PostgreSQL Cache] ← 50-100ms lookup
    ↓ (miss)
[Full RAG Pipeline] ← 8-16 seconds
    ↓
Store in both Redis & PostgreSQL
    ↓
Return Response with Links
```

## Performance Improvements

| Cache Type | Response Time | Improvement |
|------------|--------------|-------------|
| Redis Hit | 1-10ms | 1000x faster |
| PostgreSQL Hit | 50-100ms | 100x faster |
| Cache Miss (RAG) | 8-16 seconds | Baseline |

## Features

### 1. **Dual-Layer Caching**
- Redis for hot cache (1-hour TTL)
- PostgreSQL for persistent cache (30-day TTL)
- Automatic fallback if Redis is unavailable

### 2. **Link Support**
All cached responses now properly store and return source links:

```javascript
sources: [
  {
    text: "Content excerpt...",
    title: "Document Title",
    url: "https://intranet.caes.uga.edu/resource",
    score: 0.95
  }
]
```

### 3. **Intelligent Link Formatting**
- Relative URLs are converted to absolute
- Intranet paths are properly prefixed
- Invalid URLs are cleaned up

### 4. **Cache Key Generation**
- MD5 hash of normalized question
- Session-specific caching support
- Variation matching

## Configuration

### Environment Variables

```env
# Redis Configuration (Required for enhanced caching)
REDIS_URL=redis://default:password@host:6379/0

# Cache Settings
ENABLE_RESPONSE_CACHE=true        # Enable/disable caching
ENABLE_SEMANTIC_CACHE=false       # Semantic similarity (requires pgvector)
CACHE_MIN_CONFIDENCE=0.75         # Minimum confidence to cache
CACHE_TTL_DAYS=30                 # PostgreSQL cache TTL
SEMANTIC_CACHE_THRESHOLD=0.92     # Similarity threshold
```

## Usage

### Basic Implementation

```javascript
import { getResponseCache } from './src/rag/cache/responseCache.js';

// Get cache instance (automatically uses Redis if configured)
const cache = getResponseCache();

// Check cache
const cached = await cache.get(question, sessionId);
if (cached) {
  return {
    response: cached.response,
    sources: cached.sources,  // Includes formatted links
    cached: true,
    cacheSource: cached.cacheSource  // 'redis' or 'postgresql'
  };
}

// Cache new response
await cache.set(question, response, sources, {
  confidence: 0.95,
  variations: ["alternative phrasing"],
  ttlDays: 30
});
```

## How Links Work

### Source Enrichment
When caching responses, sources are automatically enriched with proper links:

```javascript
// Input sources (from RAG)
[
  {
    metadata: {
      url: "/gacounts",
      text: "GaCounts info..."
    }
  }
]

// Enriched output (in cache)
[
  {
    text: "GaCounts info...",
    title: "GaCounts Guide",
    url: "https://intranet.caes.uga.edu/gacounts",
    score: 0.95
  }
]
```

### URL Formatting Rules

1. **Relative paths** → Absolute intranet URLs
   - `/resource` → `https://intranet.caes.uga.edu/resource`

2. **Missing protocol** → HTTPS added
   - `example.com` → `https://example.com`

3. **Already formatted** → Preserved
   - `https://example.com` → `https://example.com`

## Cache Statistics

### Real-time Metrics

```javascript
const stats = await cache.getStats();

// Returns:
{
  summary: {
    total_cached_responses: 150,
    active_responses: 142,
    success_rate_pct: 94.5,
    used_last_7_days: 89
  },
  redis: {
    connected: true,
    keysCount: 45,
    memoryUsed: "2.3MB"
  },
  daily: [
    // Daily analytics...
  ]
}
```

## Testing

Run the test script to verify the enhanced caching:

```bash
node test-redis-response-cache.js
```

Expected output:
- Redis cache hits: <10ms
- PostgreSQL cache hits: 50-100ms
- Link formatting verification
- Cache statistics display

## Monitoring

### Cache Performance Logs

```
⚡ Redis Cache HIT: "What is GaCounts..." (3ms)
✅ PostgreSQL Cache HIT (exact): "How to login..." (67ms)
❌ Cache MISS: "Random question..." (2ms)
💾 Cached response for "New question..." (ID: 42)
```

### Redis Connection Status

```
✅ Redis connected for response caching
⚠️ Redis not available, using PostgreSQL cache only
```

## Maintenance

### Clear Redis Cache
```javascript
await cache.clearRedisCache();
// Clears all response:* keys from Redis
```

### Invalidate Specific Response
```javascript
await cache.invalidate(cacheId);
// Marks as inactive in PostgreSQL & clears Redis
```

### Cleanup Expired Entries
```javascript
await cache.cleanupExpired();
// Deactivates expired entries & syncs Redis
```

## Benefits

1. **Speed**: 100-1000x faster response times for cached queries
2. **Reliability**: Dual-layer ensures cache survives Redis failures
3. **Analytics**: PostgreSQL tracks all cache performance metrics
4. **Link Support**: Sources always include proper URLs for citations
5. **Cost Efficiency**: Reduces OpenAI API calls significantly

## Migration from Old Cache

The new cache is backward compatible. To migrate:

1. Add `REDIS_URL` to environment variables
2. Deploy the updated code
3. Existing PostgreSQL cache continues working
4. New cache hits are automatically stored in Redis
5. Links are enriched on new cache entries

## Troubleshooting

### Redis Not Connecting
- Verify `REDIS_URL` is correct
- Check network connectivity
- System falls back to PostgreSQL automatically

### Links Not Appearing
- Ensure sources have `url` or `metadata.url` field
- Check enrichSourcesWithLinks() formatting
- Verify intranet base URL is correct

### Cache Misses
- Check normalization is consistent
- Verify confidence thresholds
- Review question variations

## Future Enhancements

1. **Semantic Caching** - Using pgvector for similarity search
2. **Smart Invalidation** - Auto-invalidate on negative feedback
3. **Distributed Caching** - Redis Cluster support
4. **Cache Warming** - Pre-populate common queries
5. **A/B Testing** - Compare cached vs fresh responses