# Cache Management Guide

## How to Clear Cache Entries

There are three ways to clear specific cache entries when you want the system to generate a fresh response:

### Method 1: Interactive Script (Easiest)

Run the cache management script:

```bash
node clear-cache-entry.js
```

This will show a menu:
1. **Search and clear by question** - Find and remove specific questions
2. **List recent cache entries** - See what's currently cached
3. **Clear all Redis cache** - Fast clear of memory cache
4. **Clear specific cache ID** - If you know the ID
5. **Clear all cache entries** - Complete reset

#### Example: Clearing the GaCounts Presentation Entry

```bash
node clear-cache-entry.js
# Choose option 1
# Enter: "GaCounts presentation"
# Select the entry to clear
```

### Method 2: API Endpoints

Use the admin API endpoints with your API key:

#### Clear by Question Text

```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I report a presentation in GaCounts"}'
```

#### Clear All Redis Cache (Fast)

```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"clearRedis": true}'
```

#### Clear Everything

```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"clearAll": true}'
```

#### Get Cache Statistics

```bash
curl -X GET http://localhost:3000/admin/cache/stats \
  -H "x-api-key: super-secret"
```

### Method 3: Direct Database Query (Advanced)

If you have database access:

```sql
-- Find cache entries about GaCounts
SELECT id, question_normalized, response_text, last_served
FROM cached_responses
WHERE question_normalized LIKE '%gacounts%presentation%'
   OR response_text ILIKE '%gacounts%presentation%';

-- Deactivate specific entry
UPDATE cached_responses
SET is_active = FALSE
WHERE id = 123;  -- Replace with actual ID

-- Clear all cache
UPDATE cached_responses
SET is_active = FALSE
WHERE is_active = TRUE;
```

## When to Clear Cache

Clear cache entries when:

1. **Information is outdated** - The cached answer has old information
2. **Missing links** - The response doesn't include proper URLs
3. **Wrong answer** - The cached response is incorrect
4. **Testing changes** - You've updated the system and want to test
5. **Poor quality** - The cached response needs improvement

## How Cache Clearing Works

### Two-Layer Cache System

1. **Redis (Fast Layer)**
   - Stores responses for 1 hour
   - Cleared immediately with `clearRedis`
   - Auto-expires anyway

2. **PostgreSQL (Persistent Layer)**
   - Stores responses for 30 days
   - Marked as `is_active = FALSE` when cleared
   - Kept for analytics

### What Happens After Clearing

1. Next query for that question will:
   - Miss the cache
   - Run full RAG pipeline (8-16 seconds)
   - Generate fresh response with current data
   - Store new response in cache with links

2. Subsequent queries will:
   - Hit the new cache entry
   - Return instantly with proper links

## Example: Fixing the GaCounts Question

```bash
# 1. Clear the old cached entry
node clear-cache-entry.js
# Choose 1, search "gacounts presentation", clear it

# 2. Ask the question again
curl -X POST http://localhost:3000/chat \
  -H "x-api-key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"message": "How do I report a presentation in GaCounts?"}'

# This will:
# - Take 8-16 seconds (full RAG)
# - Generate fresh response
# - Include proper GaCounts links
# - Cache the new response

# 3. Verify it's cached properly
curl -X GET http://localhost:3000/admin/cache/stats \
  -H "x-api-key: super-secret"
```

## Monitoring Cache Health

Check cache statistics regularly:

```javascript
{
  "summary": {
    "total_cached_responses": 150,
    "active_responses": 142,
    "success_rate_pct": 94.5
  },
  "redis": {
    "connected": true,
    "keysCount": 45,
    "memoryUsed": "2.3MB"
  }
}
```

## Best Practices

1. **Don't clear everything** unless necessary - cache improves performance
2. **Clear specific entries** when they need updating
3. **Monitor cache hit rate** - Should be >70% for common questions
4. **Let cache auto-expire** - Most entries clear themselves after 30 days
5. **Clear Redis first** - It's faster and often sufficient

## Troubleshooting

### "No cache entries found"
- The question might be phrased differently in cache
- Try searching with just key words: "gacounts" or "presentation"
- Check if cache is enabled: `ENABLE_RESPONSE_CACHE=true`

### Changes not taking effect
- Clear Redis cache first (it's the primary lookup)
- Make sure both Redis and PostgreSQL are cleared
- Verify the app restarted after code changes

### Cache fills up again with bad data
- The issue might be in source data (Pinecone vectors)
- Check if responses are being generated correctly
- Verify source URLs are included in retrieval