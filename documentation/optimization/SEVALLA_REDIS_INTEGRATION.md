# Sevalla Redis Integration for Response Cache

## Current Redis Configuration in Sevalla

Your Sevalla environment already has Redis configured with these internal cluster variables:

```env
REDIS_HOST=pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local
REDIS_PORT=6379
REDIS_PASSWORD=Hw34343434@
REDIS_URL=redis://:Hw34343434@@pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local:6379/0
```

## Important: Redis URL Format

The current `REDIS_URL` in export.env has the correct internal cluster hostname for Sevalla:
- **Host**: `pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local`
- **Port**: `6379`
- **Database**: `0`

This is an internal Kubernetes service URL that works within the Sevalla cluster.

## How the Enhanced Cache Uses Redis

The response cache system will automatically:

1. **Detect Redis**: Check for `REDIS_URL` environment variable
2. **Connect to Redis**: Use the internal cluster URL
3. **Enable dual-layer caching**:
   - Redis for ultra-fast lookups (1-10ms)
   - PostgreSQL for persistence and analytics
4. **Store responses with links**: All cached responses include formatted source URLs

## Verification Steps

### 1. Check Redis Connection in Logs

When the app starts, look for:
```
✅ Redis connected for response caching
✅ Redis connected for Pinecone caching
```

### 2. Monitor Cache Performance

Watch for these log patterns:

**Redis Cache Hits (fastest)**:
```
⚡ Redis Cache HIT: "What is GaCounts..." (3ms)
```

**PostgreSQL Cache Hits (fallback)**:
```
✅ PostgreSQL Cache HIT (exact): "How to login..." (67ms)
```

**Cache Misses**:
```
❌ Cache MISS: "New question..." (2ms)
```

### 3. Redis Connection Issues

If Redis can't connect, you'll see:
```
⚠️ Redis connection failed, falling back to PostgreSQL cache
📝 Redis URL not configured, using PostgreSQL-only caching
```

## Cache Architecture in Sevalla

```
                    Sevalla Cluster
    ┌─────────────────────────────────────────┐
    │                                         │
    │  [Node.js App Pod]                      │
    │       ↓                                 │
    │  [Redis Service] ← Internal cluster    │
    │       ↓           connection           │
    │  [PostgreSQL Service]                  │
    │                                         │
    └─────────────────────────────────────────┘

Flow:
1. Check Redis first (1-10ms)
2. Fall back to PostgreSQL (50-100ms)
3. If both miss, run full RAG (8-16s)
4. Store in both Redis & PostgreSQL
```

## Response Cache Features with Redis

### Speed Comparison

| Cache Layer | Response Time | Use Case |
|-------------|--------------|-----------|
| Redis Hit | 1-10ms | Hot cache, frequent queries |
| PostgreSQL Hit | 50-100ms | Warm cache, less frequent |
| Cache Miss | 8-16 seconds | New queries, runs full RAG |

### Link Support

All cached responses include properly formatted links:

```javascript
{
  response: "GaCounts is UGA's grants system...",
  sources: [
    {
      text: "Source content...",
      title: "GaCounts Guide",
      url: "https://gacounts.uga.edu/docs",  // ✅ Full URL
      score: 0.95
    }
  ],
  cacheSource: "redis",  // Shows which cache layer served it
  responseTime: 3  // milliseconds
}
```

## Testing in Sevalla

### Quick Test Command

SSH into your Sevalla container and run:

```bash
# Test Redis connection
redis-cli -h pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local -p 6379 -a Hw34343434@ ping
# Should return: PONG

# Check Redis memory usage
redis-cli -h pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local -p 6379 -a Hw34343434@ info memory

# List response cache keys
redis-cli -h pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local -p 6379 -a Hw34343434@ keys "response:*"
```

### Application Test

```bash
# Run the cache test
node test-redis-response-cache.js
```

## Monitoring Redis Usage

### Check Redis Stats

```javascript
// In your app, you can check Redis stats
const cache = getResponseCache();
const stats = await cache.getStats();

console.log(stats.redis);
// Output:
// {
//   connected: true,
//   keysCount: 45,
//   memoryUsed: "2.3MB"
// }
```

### Clear Redis Cache (if needed)

```javascript
const cache = getResponseCache();
await cache.clearRedisCache();
// Clears all response:* keys
```

## Troubleshooting

### Issue: Redis Not Connecting

**Symptoms**:
- No "Redis connected" message
- All responses come from PostgreSQL
- Response times are 50-100ms instead of 1-10ms

**Solutions**:
1. Verify Redis pod is running in Sevalla
2. Check Redis password hasn't changed
3. Ensure internal DNS is resolving
4. Check Redis memory isn't full

### Issue: Cache Not Storing Links

**Symptoms**:
- Sources don't have URLs
- Links are missing or broken

**Solutions**:
1. Ensure sources have `url` or `metadata.url` field
2. Check `enrichSourcesWithLinks()` is being called
3. Verify base URLs are correct

## Best Practices

1. **Don't expose Redis externally** - Use internal cluster URLs only
2. **Monitor memory usage** - Redis stores data in memory
3. **Set appropriate TTLs** - Redis cache expires after 1 hour by default
4. **Use cache warming** - Pre-populate common queries
5. **Track cache metrics** - Monitor hit rates and response times

## Summary

Your Sevalla environment is already configured with Redis! The enhanced response cache will:

✅ Automatically detect and use the Redis service
✅ Provide 10-1000x faster response times
✅ Store and return properly formatted links
✅ Fall back gracefully if Redis is unavailable
✅ Track all metrics in PostgreSQL

No additional configuration needed - the existing `REDIS_URL` in your Sevalla environment will enable all these features automatically!