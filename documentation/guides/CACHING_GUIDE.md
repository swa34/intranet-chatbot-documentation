# Response Caching System - User Guide

Complete guide for setting up, managing, and maintaining the chatbot's response caching system.

---

## Table of Contents

1. [Overview](#overview)
2. [Initial Setup](#initial-setup)
3. [Daily Commands](#daily-commands)
4. [Cache Management](#cache-management)
5. [Maintenance Workflows](#maintenance-workflows)
6. [Troubleshooting](#troubleshooting)
7. [Configuration](#configuration)
8. [Best Practices](#best-practices)

---

## Overview

The response caching system dramatically improves chatbot performance by storing pre-computed answers to frequently asked questions. This reduces response times from ~16 seconds to under 10ms for cached queries.

### Performance Benefits

| Cache Type | Response Time | Improvement |
|------------|--------------|-------------|
| Redis Hit | 1-10ms | 1000x faster |
| PostgreSQL Hit | 50-100ms | 100x faster |
| Cache Miss (RAG) | 8-16 seconds | Baseline |

### Key Features

- **Dual-layer caching**: Redis for speed, PostgreSQL for persistence
- **Link support**: All cached responses include properly formatted source URLs
- **Automatic fallback**: System continues working if Redis fails
- **Analytics tracking**: Comprehensive performance metrics
- **Smart matching**: Exact, variation, and semantic similarity matching

---

## Initial Setup

### Prerequisites

- PostgreSQL database (already configured)
- Redis instance (optional but recommended)
- Node.js environment

### Step 1: Enable Redis (Recommended)

Add Redis URL to your `.env` file for ultra-fast caching:

```env
# Redis Configuration (Required for enhanced caching)
REDIS_URL=redis://default:YOUR_PASSWORD@YOUR_HOST:6379/0

# Cache Settings (Optional - these are defaults)
ENABLE_RESPONSE_CACHE=true        # Master switch
CACHE_MIN_CONFIDENCE=0.75         # Minimum confidence to cache
CACHE_TTL_DAYS=30                 # PostgreSQL cache TTL
```

#### Sevalla Redis Setup

1. Go to Sevalla dashboard
2. Navigate to **Databases** > **New database**
3. Select **Redis** (version 8.x)
4. Configure:
   - Database name: `response-cache`
   - Resource size: DB1
   - Cost: $5/month
5. Copy connection details to `.env`

#### Benefits of Redis

- **Sub-10ms lookups**: 10x faster than PostgreSQL alone
- **Session support**: Can cache per-session responses
- **Automatic fallback**: PostgreSQL continues working if Redis fails
- **Cost-effective**: $5/month for thousands of cached responses

**Without Redis:** System uses PostgreSQL-only caching (50-100ms lookups, still 100x faster than full RAG).

### Step 2: Create Database Tables

Run the migration to create cache tables:

```bash
node src/rag/cache/migrations/run_migration.js
```

This creates:
- `cached_responses` - Main cache storage
- `cache_analytics` - Daily metrics
- `cache_hit_log` - Detailed hit tracking
- `cache_performance_summary` - Real-time stats view

### Step 3: Populate Initial Cache

Generate cache for frequently asked questions:

```bash
# Preview what will be cached (dry run)
node src/rag/cache/generateFAQCache.js --dry-run

# Generate cache for top 50 questions
node src/rag/cache/generateFAQCache.js
```

### Step 4: Verify Setup

Check that caching is working:

```bash
# View cache statistics
node view_cache_stats.js

# Run test suite
node src/rag/cache/testCache.js
```

Expected output:
```
Total cached responses: 4
Active responses: 4
Hit rate: 34.48%
Time saved today: 164.9s
✅ Redis connected for response caching
```

---

## Daily Commands

### View Cache Status

See current cache performance, hit rate, and statistics:

```bash
node view_cache_stats.js
```

**When to run:**
- Anytime you want to check cache performance
- After populating/refreshing cache
- To monitor hit rate and time saved

**Output:**
```
Total cached responses: 150
Active responses: 142
Hit rate: 72.5%
Time saved today: 1,245.3s

Redis Status:
✅ Connected
Keys: 45
Memory: 2.3MB
```

### Refresh Cache

Update cache with new popular questions:

```bash
# Standard refresh (top 50 questions)
node src/rag/cache/generateFAQCache.js

# Cache more questions
node src/rag/cache/generateFAQCache.js --limit 100

# Require more asks before caching
node src/rag/cache/generateFAQCache.js --min-asks 3

# Higher success rate requirement
node src/rag/cache/generateFAQCache.js --min-success-rate 0.7

# Combine options
node src/rag/cache/generateFAQCache.js --limit 100 --min-asks 3 --min-success-rate 0.7
```

**When to run:**
- Weekly/monthly for regular refresh
- After conversation data grows
- When you notice new common questions
- After clearing cache

**Default behavior:**
- Finds questions asked ≥2 times
- Requires ≥50% success rate (if feedback exists)
- Caches top 50 questions
- Generates responses using RAG
- Calculates confidence scores

---

## Cache Management

### Clear Specific Cache Entry

There are three methods to clear individual cache entries:

#### Method 1: Interactive Script (Easiest)

```bash
node clear-cache-entry.js
```

Menu options:
1. **Search and clear by question** - Find and remove specific questions
2. **List recent cache entries** - See what's currently cached
3. **Clear all Redis cache** - Fast clear of memory cache
4. **Clear specific cache ID** - If you know the ID
5. **Clear all cache entries** - Complete reset

Example workflow:
```bash
node clear-cache-entry.js
# Choose option 1
# Enter: "GaCounts presentation"
# Select the entry to clear
```

#### Method 2: API Endpoints

Use admin API endpoints with your API key:

**Clear by question text:**
```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I report a presentation in GaCounts"}'
```

**Clear Redis cache (fast):**
```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"clearRedis": true}'
```

**Clear everything:**
```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"clearAll": true}'
```

**Get cache statistics:**
```bash
curl -X GET http://localhost:3000/admin/cache/stats \
  -H "x-api-key: super-secret"
```

#### Method 3: Direct Database Query

If you have database access:

```sql
-- Find cache entries
SELECT id, question_normalized, response_text, last_served
FROM cached_responses
WHERE question_normalized LIKE '%keyword%'
   OR response_text ILIKE '%keyword%';

-- Deactivate specific entry
UPDATE cached_responses
SET is_active = FALSE
WHERE id = 123;

-- Clear all cache
UPDATE cached_responses
SET is_active = FALSE
WHERE is_active = TRUE;
```

### When to Clear Cache

Clear cache entries when:

1. **Information is outdated** - The cached answer has old information
2. **Missing links** - The response doesn't include proper URLs
3. **Wrong answer** - The cached response is incorrect
4. **Testing changes** - You've updated the system and want to test
5. **Poor quality** - The cached response needs improvement

### How Cache Clearing Works

The system uses a two-layer cache:

1. **Redis (Fast Layer)**
   - Stores responses for 1 hour
   - Cleared immediately with `clearRedis`
   - Auto-expires anyway

2. **PostgreSQL (Persistent Layer)**
   - Stores responses for 30 days
   - Marked as `is_active = FALSE` when cleared
   - Kept for analytics

**What happens after clearing:**

1. Next query for that question will:
   - Miss the cache
   - Run full RAG pipeline (8-16 seconds)
   - Generate fresh response with current data
   - Store new response in cache with links

2. Subsequent queries will:
   - Hit the new cache entry
   - Return instantly with proper links

### Cleanup Expired Entries

Remove expired cache entries based on TTL:

```bash
# Via Node.js
node -e "
import('./src/rag/cache/responseCache.js').then(async ({ getResponseCache }) => {
  const cache = getResponseCache();
  const count = await cache.cleanupExpired();
  console.log(\`Cleaned \${count} expired entries\`);
  await cache.destroy();
});
"
```

**Or via SQL:**
```sql
SELECT deactivate_expired_cache();
```

**When to run:**
- Daily/weekly cron job
- When cache grows too large
- Before cache refresh
- During maintenance windows

---

## Maintenance Workflows

### Initial Setup (One Time)

```bash
# 1. Create database tables
node src/rag/cache/migrations/run_migration.js

# 2. Add Redis URL to .env (optional)
# REDIS_URL=redis://default:password@host:6379/0

# 3. Populate cache with FAQs
node src/rag/cache/generateFAQCache.js

# 4. Verify it worked
node view_cache_stats.js
```

### Weekly Maintenance

```bash
# 1. Check current performance
node view_cache_stats.js

# 2. Refresh cache with new popular questions
node src/rag/cache/generateFAQCache.js

# 3. Review performance improvement
node view_cache_stats.js

# 4. (Optional) Cleanup expired entries
# Runs automatically via trigger, but can force it
```

### Fix Bad/Incorrect Cache

```bash
# 1. Clear specific entry or all cache
node clear-cache-entry.js

# 2. Regenerate from scratch
node src/rag/cache/generateFAQCache.js

# 3. Verify new cache
node view_cache_stats.js
```

### Test After Deployment

```bash
# 1. Run test suite
node src/rag/cache/testCache.js

# 2. Check stats
node view_cache_stats.js

# 3. Test in chatbot UI with these questions:
# - "How do I add an event to the extension calendar?"
# - "What is the CAES welcome message?"
# - "How do download the CAES logo?"
# - "How do I report a presentation in GaCounts?"
```

### Monitor Cache Health

Check cache statistics regularly:

```bash
curl -X GET http://localhost:3000/admin/cache/stats \
  -H "x-api-key: super-secret"
```

Expected output:
```json
{
  "summary": {
    "total_cached_responses": 150,
    "active_responses": 142,
    "success_rate_pct": 94.5,
    "used_last_7_days": 89
  },
  "redis": {
    "connected": true,
    "keysCount": 45,
    "memoryUsed": "2.3MB"
  },
  "daily": [
    {
      "date": "2025-10-10",
      "cache_hits": 245,
      "cache_misses": 89,
      "hit_rate_pct": 73.4,
      "time_saved_ms": 3429000
    }
  ]
}
```

---

## Troubleshooting

### Cache Not Working

**1. Check if enabled:**
```bash
grep ENABLE_RESPONSE_CACHE .env
```
Should be `true` or not set (defaults to true).

**2. Verify tables exist:**
```sql
\dt cache*
```
Should show: `cache_analytics`, `cache_hit_log`, `cached_responses`

**3. Check cache entries:**
```bash
node view_cache_stats.js
```

**4. Review server logs:**
Look for: `✅ Cache HIT` or `❌ Cache MISS`

### Redis Not Connecting

**Symptoms:**
```
⚠️ Redis not available, using PostgreSQL cache only
```

**Solutions:**

1. Verify REDIS_URL is correct in `.env`
2. Check Sevalla Redis status in dashboard
3. Ensure Redis database is running
4. Test connectivity:
```bash
node test-redis-response-cache.js
```

**Note:** System automatically falls back to PostgreSQL if Redis fails.

### Low Hit Rate

**Symptoms:** Hit rate below 30%

**Solutions:**

1. **Populate more questions:**
```bash
node src/rag/cache/generateFAQCache.js --limit 100
```

2. **Lower confidence threshold:**
```env
# In .env
CACHE_MIN_CONFIDENCE=0.6
```

3. **Add more variations manually:**
```sql
UPDATE cached_responses
SET question_variations = ARRAY['variation1', 'variation2']
WHERE id = 123;
```

### Stale Responses

**Symptoms:** Cached answers contain outdated information

**Solutions:**

1. **Reduce TTL:**
```env
# In .env
CACHE_TTL_DAYS=7
```

2. **Invalidate specific entry:**
```bash
node clear-cache-entry.js
# Choose option 1, search for question
```

3. **Regenerate cache:**
```bash
node src/rag/cache/generateFAQCache.js
```

### Links Not Appearing in Cache

**Symptoms:** Cached responses missing source URLs

**Solutions:**

1. Clear and regenerate cache (links added automatically):
```bash
node clear-cache-entry.js
# Clear the entry
# Ask question again to regenerate with links
```

2. Verify sources have URLs in metadata
3. Check enrichSourcesWithLinks() formatting

### "No cache entries found"

**Symptoms:** Search can't find cached questions

**Solutions:**

1. Try searching with just keywords: "gacounts" or "presentation"
2. Check if cache is enabled: `ENABLE_RESPONSE_CACHE=true`
3. List all cache entries:
```bash
node clear-cache-entry.js
# Choose option 2
```

### Changes Not Taking Effect

**Symptoms:** Updates don't appear in responses

**Solutions:**

1. Clear Redis cache first (it's the primary lookup):
```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"clearRedis": true}'
```

2. Make sure both Redis and PostgreSQL are cleared
3. Verify app restarted after code changes

---

## Configuration

### Environment Variables

```env
# Redis Configuration (Optional but Recommended)
REDIS_URL=redis://default:password@host:6379/0

# Cache Control
ENABLE_RESPONSE_CACHE=true          # Enable/disable cache (default: true)
CACHE_MIN_CONFIDENCE=0.75           # Min confidence to serve cached response
CACHE_TTL_DAYS=30                   # How long to keep cache entries (days)

# Semantic Cache (Future - requires pgvector)
ENABLE_SEMANTIC_CACHE=false         # Enable semantic similarity matching
SEMANTIC_CACHE_THRESHOLD=0.92       # Min similarity score for match
```

### Disable Cache Temporarily

```env
# Set in .env
ENABLE_RESPONSE_CACHE=false
```

Or remove the variable to use default (enabled).

### Adjust Cache Sensitivity

**More aggressive caching (lower quality threshold):**
```env
CACHE_MIN_CONFIDENCE=0.6
```

**More conservative caching (higher quality threshold):**
```env
CACHE_MIN_CONFIDENCE=0.9
```

---

## Best Practices

### Cache Management

1. **Don't clear everything** unless necessary - cache improves performance
2. **Clear specific entries** when they need updating
3. **Monitor cache hit rate** - Should be >70% for common questions
4. **Let cache auto-expire** - Most entries clear themselves after 30 days
5. **Clear Redis first** - It's faster and often sufficient

### Performance Optimization

1. **Use Redis** for production environments
2. **Populate cache proactively** with common questions
3. **Monitor hit rates** and adjust thresholds accordingly
4. **Review low-performing entries** and improve them
5. **Set appropriate TTLs** based on content freshness needs

### Monitoring

1. **Check stats weekly** to track improvements
2. **Review cache analytics** for usage patterns
3. **Monitor Redis memory** usage
4. **Track response times** for cache hits vs misses
5. **Alert on low hit rates** (below 50%)

### Content Quality

1. **Review cached responses** for accuracy
2. **Ensure links are present** in cached answers
3. **Update stale content** regularly
4. **Remove poor-quality entries** promptly
5. **Test after major updates** to source data

---

## SQL Queries for Analysis

### View All Cached Questions

```sql
SELECT
  id,
  question_normalized,
  cache_confidence,
  times_served,
  is_active
FROM cached_responses
WHERE is_active = TRUE
ORDER BY times_served DESC;
```

### Today's Cache Performance

```sql
SELECT * FROM cache_analytics
WHERE date = CURRENT_DATE;
```

### Top Performing Cache Entries

```sql
SELECT
  question_normalized,
  times_served,
  positive_feedback,
  negative_feedback,
  cache_confidence
FROM cached_responses
WHERE is_active = TRUE
ORDER BY times_served DESC
LIMIT 10;
```

### Recent Cache Hits

```sql
SELECT
  cr.question_normalized,
  chl.hit_type,
  chl.response_time_ms,
  chl.created_at
FROM cache_hit_log chl
JOIN cached_responses cr ON cr.id = chl.cached_response_id
ORDER BY chl.created_at DESC
LIMIT 20;
```

### Cache Performance Summary

```sql
SELECT * FROM cache_performance_summary;
```

---

## Performance Expectations

### Cache Hit

- **Response time:** 50-300ms (Redis: 1-10ms)
- **Cost:** $0
- **Load:** Minimal (database lookup)

### Cache Miss

- **Response time:** ~16 seconds (normal RAG)
- **Cost:** $0.002-0.01
- **Load:** Full (vector search + LLM)

### Overall Improvement

- **56x faster** for cache hits
- **100% cost savings** for cached responses
- **Growing effectiveness** as usage increases
- **Better UX** with instant responses

### Expected Hit Rates

| Usage Pattern | Expected Hit Rate | Performance Gain |
|--------------|------------------|------------------|
| New deployment | 10-20% | 10-20% faster avg |
| After 1 month | 40-60% | 40-60% faster avg |
| Mature system | 70-80% | 70-80% faster avg |

---

## Important Files Reference

### Keep These Files

- `view_cache_stats.js` - Statistics viewer
- `clear-cache-entry.js` - Interactive cache management
- `src/rag/cache/generateFAQCache.js` - FAQ generator
- `src/rag/cache/testCache.js` - Test suite
- `src/rag/cache/responseCache.js` - Core cache manager
- `src/rag/cache/migrations/` - Database setup

### Database Tables

- `cached_responses` - Main cache storage
- `cache_analytics` - Daily metrics
- `cache_hit_log` - Detailed hit tracking
- `cache_performance_summary` (view) - Real-time stats

---

## Support

For technical details and architecture information, see:
- [Caching Architecture Guide](../architecture/CACHING_ARCHITECTURE.md)
- [Full Implementation Details](src/rag/cache/README.md)

---

Last Updated: October 10, 2025
