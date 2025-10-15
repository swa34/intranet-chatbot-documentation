# Redis Optimization Guide

## Overview

This document covers the Redis connection pooling optimization implemented for the UGA Intranet Chatbot. The optimization reduces connection overhead, standardizes configuration, and provides memory management recommendations.

## Architecture Changes

### Before Optimization
```
┌─────────────────┐
│  Redis Server   │
└─────────────────┘
    ↑  ↑  ↑  ↑  ↑
    │  │  │  │  │  5 separate connections!
    │  │  │  │  └── popularQueries.js (3 retries, 5s timeout)
    │  │  │  └────── sessionManager.js (10 retries, 10s timeout)
    │  │  └───────── conversationMemoryRedis.js (10 retries, 10s timeout)
    │  └──────────── responseCacheWithRedis.js (3 retries, 5s timeout)
    └─────────────── pineconeClientWithRedis.js (3 retries, 5s timeout)
```

**Issues:**
- 5 separate Redis connections (inefficient)
- Inconsistent retry/timeout configurations
- Duplicated connection code across modules
- No centralized configuration management

### After Optimization
```
┌─────────────────┐
│  Redis Server   │
└─────────────────┘
    ↑           ↑
    │           │  2 shared connections (pooled)
    │           │
    │           └── Cache Tier Client (5 retries, 7.5s timeout)
    │               ├── responseCacheWithRedis.js
    │               ├── pineconeClientWithRedis.js
    │               └── popularQueries.js
    │
    └────────────── Critical Tier Client (10 retries, 10s timeout)
                    ├── sessionManager.js
                    └── conversationMemoryRedis.js
```

**Benefits:**
- ✅ 60% reduction in connections (5 → 2)
- ✅ Standardized configuration by service tier
- ✅ Centralized management in `src/config/redisConfig.js`
- ✅ Shared utilities (scanKeys, isRedisAvailable)
- ✅ Better resource utilization

## Configuration Tiers

### Critical Tier (High Priority)
**Services:**
- User sessions (`sessionManager.js`)
- Conversation memory (`conversationMemoryRedis.js`)

**Configuration:**
```javascript
{
  retryStrategy: 10 retries with exponential backoff (max 5s delay),
  connectTimeout: 10000ms (10 seconds),
  maxRetriesPerRequest: 2,
  enableOfflineQueue: false
}
```

**Rationale:** These services directly impact user experience. Longer retry/timeout tolerance ensures sessions and conversations remain available even during transient Redis issues.

### Cache Tier (Lower Priority)
**Services:**
- Response cache (`responseCacheWithRedis.js`)
- Pinecone validation cache (`pineconeClientWithRedis.js`)
- Popular queries analytics (`popularQueries.js`)

**Configuration:**
```javascript
{
  retryStrategy: 5 retries with exponential backoff (max 3s delay),
  connectTimeout: 7500ms (7.5 seconds),
  maxRetriesPerRequest: 1,
  enableOfflineQueue: false
}
```

**Rationale:** These services can fail gracefully with fallbacks (PostgreSQL, local cache). Shorter retry/timeout prevents blocking requests when Redis is unavailable.

## Key Components

### 1. `src/config/redisConfig.js`
Centralized configuration for all Redis connections.

**Exports:**
- `REDIS_CONFIGS` - Configuration objects for each tier
- `getRedisConfig(tier)` - Get config for specific tier
- `REDIS_KEY_PREFIXES` - Standardized key prefixes
- `REDIS_TTL` - TTL constants for different data types
- `MEMORY_RECOMMENDATIONS` - Memory management best practices

**Example:**
```javascript
import { getRedisConfig, REDIS_TTL } from '../config/redisConfig.js';

const config = getRedisConfig('cache');
const ttl = REDIS_TTL.RESPONSE_CACHE; // 3600 seconds
```

### 2. `src/utils/redisClient.js`
Shared Redis client singleton with lifecycle management.

**Exports:**
- `getCriticalRedisClient()` - Get critical tier client
- `getCacheRedisClient()` - Get cache tier client
- `isRedisAvailable(client)` - Check if client is ready
- `scanKeys(client, pattern, count)` - Non-blocking key scanning
- `closeAllRedisConnections()` - Cleanup on shutdown
- `getRedisStats()` - Connection statistics

**Example:**
```javascript
import { getCacheRedisClient, isRedisAvailable, scanKeys } from '../utils/redisClient.js';

const redis = getCacheRedisClient();

if (isRedisAvailable(redis)) {
  const keys = await scanKeys(redis, 'response:*', 100);
  console.log(`Found ${keys.length} cached responses`);
}
```

### 3. Module Updates
All 5 Redis-using modules updated to use shared clients:

**Before:**
```javascript
// Each module created its own client
this.redis = new Redis(REDIS_URL, { ... });
```

**After:**
```javascript
// Import shared client
import { getCacheRedisClient } from '../utils/redisClient.js';

// Use shared client
this.redis = getCacheRedisClient();
```

## Memory Management

### Current Redis Configuration
Based on your environment, Redis is running on Kinsta's managed infrastructure:
```
Host: us-east1-001.proxy.kinsta.app:30088
```

### Recommended Settings

#### 1. Eviction Policy
**Recommended:** `allkeys-lru` (Least Recently Used)

```bash
# Set on Redis server
CONFIG SET maxmemory-policy allkeys-lru
```

**Why this policy?**
- Automatically evicts least-recently-used keys when memory limit reached
- Works across all keys (not just those with TTL)
- Best for cache-heavy workloads like this chatbot
- Ensures hot data (frequently accessed) stays in memory

**Alternatives:**
- `volatile-lru` - Only evicts keys with TTL set (alternative if you want to protect certain keys)
- `noeviction` - Returns errors when memory full ⚠️ NOT recommended for production

#### 2. Memory Limit
**Recommended:** `256mb` (adjust based on available resources)

```bash
# Set on Redis server
CONFIG SET maxmemory 256mb
```

**Current usage pattern:**
- Response cache: ~1-5 keys per unique question (1-10KB each)
- Session data: ~1 key per active user (~500 bytes each)
- Conversation history: ~2 keys per active conversation (~2-5KB each)
- Analytics: ~50-100 keys for query tracking (~1KB each)

**Estimated usage with 256MB:**
- ~25,000 cached responses (@ 10KB each)
- ~500,000 active sessions (@ 500 bytes each)
- ~50,000 conversation histories (@ 5KB each)

This should be more than sufficient for typical usage.

#### 3. Persistence
For a cache-heavy application, you may want to **disable persistence** to reduce overhead:

```bash
# Disable RDB snapshots
CONFIG SET save ""

# Disable AOF (Append-Only File)
CONFIG SET appendonly no
```

**Trade-off:** Faster performance, but cache lost on Redis restart (acceptable since PostgreSQL is source of truth).

### Key Expiration Strategy

All keys use TTL (Time To Live) for automatic cleanup:

| Data Type | TTL | Refresh Strategy |
|-----------|-----|------------------|
| Response Cache | 1 hour | Reset on each hit (keeps popular hot) |
| Pinecone Validation | 1 hour | Fixed TTL |
| User Sessions | 24 hours | Reset on each access |
| Conversation History | 24 hours | Reset on each message |
| Query Analytics | 30 days | Fixed TTL |

### Monitoring

Check Redis memory usage:
```javascript
import { getCacheRedisClient } from '../utils/redisClient.js';

const redis = getCacheRedisClient();
const info = await redis.info('memory');
console.log(info);
```

Key metrics:
- `used_memory_human` - Human-readable memory usage
- `maxmemory_human` - Configured memory limit
- `evicted_keys` - Number of keys evicted (should be low)
- `keyspace_hits` / `keyspace_misses` - Cache hit rate

## Testing

### Verify Shared Clients
```bash
npm start
```

Look for these log messages:
```
✅ Redis cache tier connected
✅ Redis critical tier connected
✓ Redis cache tier ready
✓ Redis critical tier ready
✓ Redis response cache initialized (shared client, cache tier)
```

### Check Connection Stats
The shared client automatically logs connection events:
```
✅ Redis cache tier connected
🔄 Redis cache tier reconnecting...
⚠️  Redis cache tier connection closed
```

### Test Cache Performance
```bash
# First request (cache miss)
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: super-secret" \
  -d '{"message":"test","sessionId":"test123"}'

# Second request (should be Redis cache hit)
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: super-secret" \
  -d '{"message":"test","sessionId":"test123"}'
```

Expected output:
```
⚡ Redis Cache HIT: "test..." (5-10ms)
```

## Migration Notes

### Breaking Changes
**None!** The changes are internal only:
- All external APIs remain the same
- No changes to how modules are imported/used
- Backward compatible with existing code

### Rollback Plan
If issues arise:
```bash
git checkout main
git merge --abort  # if mid-merge
npm start
```

The previous implementation remains available on `main` branch before merge.

## Best Practices

### 1. Don't Create New Redis Clients
Always use the shared clients:

**❌ Bad:**
```javascript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL); // Creates new connection!
```

**✅ Good:**
```javascript
import { getCacheRedisClient } from '../utils/redisClient.js';
const redis = getCacheRedisClient(); // Reuses shared connection
```

### 2. Always Check Availability
```javascript
import { getCacheRedisClient, isRedisAvailable } from '../utils/redisClient.js';

const redis = getCacheRedisClient();

if (isRedisAvailable(redis)) {
  // Safe to use Redis
  await redis.set('key', 'value');
} else {
  // Fallback logic
  console.warn('Redis unavailable, using fallback');
}
```

### 3. Use Shared Utilities
```javascript
import { scanKeys } from '../utils/redisClient.js';

// Don't implement your own SCAN loop
const keys = await scanKeys(redis, 'pattern:*', 100);
```

### 4. Choose Correct Tier
- **Critical tier** - User-facing data (sessions, conversations)
- **Cache tier** - Performance optimization data (caches, analytics)

## Troubleshooting

### Redis Connection Failed
**Symptom:** `⚠️ Redis cache connection failed, falling back to alternatives`

**Solution:**
1. Check `REDIS_URL` in `.env`
2. Verify Redis server is running
3. Check network connectivity
4. Review firewall rules

### High Memory Usage
**Symptom:** Redis using too much memory

**Solutions:**
1. Check eviction policy: `redis-cli CONFIG GET maxmemory-policy`
2. Set memory limit: `redis-cli CONFIG SET maxmemory 256mb`
3. Clear old data: `redis-cli FLUSHDB` (development only!)
4. Review TTL values in `src/config/redisConfig.js`

### Slow Performance
**Symptom:** Cache operations taking too long

**Solutions:**
1. Check network latency to Redis server
2. Verify using non-blocking SCAN instead of KEYS
3. Monitor `redis-cli --latency`
4. Consider increasing connection timeout in `redisConfig.js`

### Connection Leaks
**Symptom:** Too many Redis connections over time

**Solutions:**
1. Verify using shared clients (`getCacheRedisClient()` or `getCriticalRedisClient()`)
2. Don't create new `Redis()` instances directly
3. Check shutdown handlers are registered (automatic in `redisClient.js`)

## Performance Metrics

### Before Optimization
- Redis connections: 5
- Connection overhead: ~5MB (1MB per client)
- Config consistency: ❌ Inconsistent
- Code duplication: ✅ Yes (scanKeys in multiple files)

### After Optimization
- Redis connections: 2 (60% reduction)
- Connection overhead: ~2MB (1MB per client)
- Config consistency: ✅ Centralized
- Code duplication: ❌ None (shared utilities)

**Expected improvements:**
- Faster startup (fewer connections to establish)
- Lower memory usage (fewer client instances)
- Easier maintenance (one config file to update)
- Better reliability (standardized retry strategies)

## Related Files

### Core Implementation
- `src/config/redisConfig.js` - Configuration and constants
- `src/utils/redisClient.js` - Shared client singleton

### Using Modules
- `src/rag/cache/responseCacheWithRedis.js` - Response caching
- `src/rag/utils/pineconeClientWithRedis.js` - Pinecone validation cache
- `src/rag/analytics/popularQueries.js` - Query analytics
- `src/auth/sessionManager.js` - User sessions
- `src/storage/conversationMemoryRedis.js` - Conversation history

### Documentation
- `documentation/REDIS_OPTIMIZATION.md` - This file
- `documentation/REDIS_BLOCKING_FIX.md` - Previous SCAN optimization (if exists)

## Future Enhancements

### Potential Improvements
1. **Redis Cluster Support** - Horizontal scaling for high traffic
2. **Sentinel Configuration** - Automatic failover for HA
3. **Metrics Dashboard** - Real-time monitoring of cache performance
4. **Dynamic TTL** - Adjust TTL based on query popularity
5. **Cache Warming** - Pre-populate cache on startup

### Monitoring Integration
Consider adding to admin dashboard:
```javascript
// Admin endpoint to show Redis stats
app.get('/admin/redis-stats', async (req, res) => {
  const { getRedisStats } = await import('./utils/redisClient.js');
  const stats = getRedisStats();
  res.json(stats);
});
```

---

**Last Updated:** 2025-10-15
**Author:** Claude Code with sa69508
**Version:** 1.0
