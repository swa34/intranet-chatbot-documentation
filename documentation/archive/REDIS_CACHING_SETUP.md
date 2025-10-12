# Redis Caching for Pinecone - Sevalla Setup Guide

## Overview

This guide explains how to set up Redis caching to eliminate Pinecone connection delays in serverless/containerized environments.

**Problem:** Sevalla's DB1 plan (0.25 CPU/0.25 GB RAM) restarts containers frequently, causing 4-second Pinecone reconnections on each request.

**Solution:** Use Sevalla's Redis add-on to cache Pinecone validation across container restarts.

## Performance Impact

**Without Redis (current):**
- First request after container start: ~4000ms Pinecone connection
- Subsequent requests (if container stays alive): 0ms
- Reality: Container restarts frequently, so most requests pay the 4s penalty

**With Redis:**
- First request ever: ~750ms Pinecone connection + Redis cache write
- All subsequent requests (even after container restarts): ~50ms Redis cache read
- **Result: 3-4 second improvement per request!**

## Setup Instructions

### 1. Enable Redis in Sevalla

1. Go to your Sevalla dashboard
2. Navigate to **Databases** → **New database**
3. Select **Redis** (version 8.x)
4. Configure:
   - **Database name:** `pincone-cache`
   - **Database password:** (auto-generated, save this!)
   - **Name:** `pincone-cache`
   - **Resource size:** DB1 (same as your app)
   - **Cost:** $5/month additional

5. Click **Create database**
6. Wait for provisioning to complete
7. Note the connection details:
   - **Host:** (provided by Sevalla)
   - **Port:** 6379 (default)
   - **Password:** (from step 4)

### 2. Configure Your Application

Add to your `.env` file:

```env
# Redis Configuration for Pinecone Caching
# Format: redis://username:password@host:port/database
# Replace with your actual Sevalla Redis credentials
REDIS_URL=redis://default:YOUR_PASSWORD@YOUR_HOST:6379/0

# Optional: Disable re-ranking during testing for even faster responses
ENABLE_LLM_RERANKING=false
```

### 3. Deploy the Updated Code

The application now includes:
- `src/rag/utils/pineconeClientWithRedis.js` - Redis-cached Pinecone client
- Automatic fallback to local caching if Redis is unavailable
- Graceful error handling

Deploy to Sevalla:
```bash
git add .
git commit -m "feat: Add Redis caching for Pinecone connections"
git push origin feature/performance-monitoring
```

### 4. Verify Redis is Working

After deployment, check your Sevalla logs for:

**First request (initial cache write):**
```
🔌 Initializing Pinecone client...
✅ Pinecone connected to index: uga-intranet-index (750ms)
✅ Redis connected for Pinecone caching
💾 Cached Pinecone validation in Redis (TTL: 3600s)
```

**Subsequent requests (cache hits):**
```
♻️ Using Redis-cached Pinecone validation
```

**If Redis fails (fallback to local):**
```
⚠️ Redis not available, using local cache only
```

## Monitoring

### Check Redis Status
```bash
# In your application logs
grep "Redis" /path/to/logs

# Expected output:
✅ Redis connected for Pinecone caching
♻️ Using Redis-cached Pinecone validation
```

### Performance Metrics to Track
- **pineconeConnect time:** Should drop from 4000ms to <100ms
- **Total retrieval time:** Should improve by 3-4 seconds
- **Overall response time:** Should drop from 8-10s to 4-6s

## Troubleshooting

### Redis Connection Errors

If you see: `⚠️ Redis connection failed, falling back to local cache`

1. Verify REDIS_URL is correct
2. Check Sevalla Redis status
3. Ensure Redis database is running
4. Check network connectivity

### Cache Not Persisting

If you see: `🔌 Initializing Pinecone client...` on every request

1. Verify Redis is actually connected (check for `✅ Redis connected`)
2. Check Redis memory usage (might be full)
3. Verify cache TTL isn't too short (default: 1 hour)

### Testing Locally

Test with Redis Cloud free tier:
1. Sign up at https://redis.com/try-free/
2. Create free database (30MB)
3. Add REDIS_URL to local .env
4. Run: `node test-redis-cache.js`

## Cost-Benefit Analysis

**Cost:** $5/month for Redis add-on

**Benefit:**
- 3-4 second faster response time per request
- Better user experience
- Higher user satisfaction
- Reduced OpenAI API timeouts

**Break-even:** If you serve >50 requests/day, the improved UX justifies the cost

## Alternative Solutions

If $5/month is too much:

1. **Accept current performance:** 8-10s responses aren't terrible
2. **Upgrade Sevalla plan:** Higher tiers have more persistent containers
3. **Use different hosting:** VPS with persistent Node.js process
4. **Disable re-ranking:** Saves 1-2s, free solution

## Summary

Redis caching is the best solution for your current Sevalla DB1 plan. It:
- Eliminates 4-second Pinecone reconnection delays
- Works across container restarts
- Costs only $5/month
- Requires minimal code changes
- Has automatic fallback if Redis fails

Expected improvement: **40-50% faster responses** (from 8-10s to 4-6s)