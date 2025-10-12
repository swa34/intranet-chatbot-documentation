# Response Cache - Command Reference

Quick reference guide for managing the response caching system.

---

## 📊 View Cache Status

**See current cache performance, hit rate, and statistics**

```bash
node view_cache_stats.js
```

**When to run:**
- Anytime you want to check cache performance
- After populating/refreshing cache
- To monitor hit rate and time saved

**Output:**
```
Total cached responses: 4
Active responses: 4
Hit rate: 34.48%
Time saved today: 164.9s
```

---

## 🔄 Populate/Refresh Cache

### Preview (Dry Run)

**See what questions will be cached WITHOUT actually caching them**

```bash
node src/rag/cache/generateFAQCache.js --dry-run
```

**When to run:**
- Before first cache generation
- To preview what questions qualify
- To test with different parameters

---

### Generate Cache (Real)

**Actually cache the top FAQs**

```bash
node src/rag/cache/generateFAQCache.js
```

**When to run:**
- Initial setup (first time)
- Weekly/monthly to refresh with new popular questions
- After clearing cache
- After conversation data grows

**Default behavior:**
- Finds questions asked ≥2 times
- Requires ≥50% success rate (if feedback exists)
- Caches top 50 questions
- Generates responses using RAG
- Calculates confidence scores

---

### Custom Options

**Cache more questions:**
```bash
node src/rag/cache/generateFAQCache.js --limit 100
```

**Require more asks before caching:**
```bash
node src/rag/cache/generateFAQCache.js --min-asks 3
```

**Higher success rate requirement:**
```bash
node src/rag/cache/generateFAQCache.js --min-success-rate 0.7
```

**Combine options:**
```bash
node src/rag/cache/generateFAQCache.js --limit 100 --min-asks 3 --min-success-rate 0.7
```

---

## 🗑️ Clear Cache

**Delete ALL cached responses (use with caution!)**

```bash
node -e "
import('dotenv/config');
import('pg').then(({ default: pg }) => {
  const { Pool } = pg;
  const pool = new Pool({
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 5432,
    database: process.env.DB_DATABASE,
    user: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    ssl: false
  });
  pool.query('DELETE FROM cached_responses').then(() => {
    console.log('✅ All cache cleared');
    pool.end();
  });
});
"
```

**When to run:**
- Cache contains bad/test data
- Before fresh regeneration
- **WARNING:** Don't run in production unless intentional!

**Alternative (safer):** Delete specific entries via SQL:
```sql
DELETE FROM cached_responses WHERE id = 123;
```

---

## 🧪 Test Cache

**Verify cache is working correctly**

```bash
node src/rag/cache/testCache.js
```

**When to run:**
- After initial setup
- After code changes
- To verify read/write operations
- After database migrations

**Tests:**
- Set cache entry
- Get exact match
- Get variation match
- Cache miss
- Feedback updates
- Statistics retrieval

---

## 🧹 Cleanup Expired Entries

**Remove expired cache entries based on TTL**

```bash
node -e "
import('./src/rag/cache/responseCache.js').then(async ({ getResponseCache }) => {
  const cache = getResponseCache();
  const count = await cache.cleanupExpired();
  console.log(\`Cleaned \${count} expired entries\`);
  await cache.destroy();
});
"
```

**When to run:**
- Daily/weekly cron job
- When cache grows too large
- Before cache refresh
- During maintenance windows

**Alternative:** PostgreSQL function:
```sql
SELECT deactivate_expired_cache();
```

---

## 📋 Typical Workflows

### Initial Setup (One Time)

```bash
# 1. Create database tables
node src/rag/cache/migrations/run_migration.js

# 2. Populate cache with FAQs
node src/rag/cache/generateFAQCache.js

# 3. Verify it worked
node view_cache_stats.js
```

---

### Weekly Maintenance

```bash
# 1. Refresh cache with new popular questions
node src/rag/cache/generateFAQCache.js

# 2. Check performance
node view_cache_stats.js

# 3. (Optional) Cleanup expired entries
# (runs automatically via trigger, but can force it)
```

---

### Fix Bad/Incorrect Cache

```bash
# 1. Clear all cache entries
node -e "..." (use DELETE command above)

# 2. Regenerate from scratch
node src/rag/cache/generateFAQCache.js

# 3. Verify new cache
node view_cache_stats.js
```

---

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

---

## ⚙️ Configuration

### Environment Variables

Add to your `.env` file:

```bash
# Cache Control
ENABLE_RESPONSE_CACHE=true          # Enable/disable cache (default: true)
CACHE_MIN_CONFIDENCE=0.75           # Min confidence to serve cached response
CACHE_TTL_DAYS=30                   # How long to keep cache entries (days)

# Semantic Cache (Future - requires pgvector)
ENABLE_SEMANTIC_CACHE=false         # Enable semantic similarity matching
SEMANTIC_CACHE_THRESHOLD=0.92       # Min similarity score for match
```

### Disable Cache Temporarily

```bash
# Set in .env
ENABLE_RESPONSE_CACHE=false
```

Or remove the variable to use default (enabled).

---

## 📁 Important Files

### Keep These Files:

- `view_cache_stats.js` - Statistics viewer
- `src/rag/cache/generateFAQCache.js` - FAQ generator
- `src/rag/cache/testCache.js` - Test suite
- `src/rag/cache/responseCache.js` - Core cache manager
- `src/rag/cache/README.md` - Full documentation
- `src/rag/cache/migrations/` - Database setup

### Database Tables:

- `cached_responses` - Main cache storage
- `cache_analytics` - Daily metrics
- `cache_hit_log` - Detailed hit tracking
- `cache_performance_summary` (view) - Real-time stats

---

## 🔍 SQL Queries (Direct Database Access)

### View all cached questions:
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

### Today's cache performance:
```sql
SELECT * FROM cache_analytics
WHERE date = CURRENT_DATE;
```

### Top performing cache entries:
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

### Recent cache hits:
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

### Cache performance summary:
```sql
SELECT * FROM cache_performance_summary;
```

---

## 🚨 Troubleshooting

### Cache not working?

1. **Check if enabled:**
   ```bash
   grep ENABLE_RESPONSE_CACHE .env
   ```
   Should be `true` or not set (defaults to true)

2. **Verify tables exist:**
   ```sql
   \dt cache*
   ```
   Should show: `cache_analytics`, `cache_hit_log`, `cached_responses`

3. **Check cache entries:**
   ```bash
   node view_cache_stats.js
   ```

4. **Review server logs:**
   Look for: `✅ Cache HIT` or `❌ Cache MISS`

### Low hit rate?

1. **Populate more questions:**
   ```bash
   node src/rag/cache/generateFAQCache.js --limit 100
   ```

2. **Lower confidence threshold:**
   ```bash
   # In .env
   CACHE_MIN_CONFIDENCE=0.6
   ```

3. **Add more variations manually:**
   ```sql
   UPDATE cached_responses
   SET question_variations = ARRAY['variation1', 'variation2']
   WHERE id = 123;
   ```

### Stale responses?

1. **Reduce TTL:**
   ```bash
   # In .env
   CACHE_TTL_DAYS=7
   ```

2. **Invalidate specific entry:**
   ```sql
   UPDATE cached_responses
   SET is_active = FALSE
   WHERE id = 123;
   ```

3. **Regenerate cache:**
   ```bash
   node src/rag/cache/generateFAQCache.js
   ```

---

## 📈 Performance Expectations

### Cache Hit:
- Response time: **50-300ms**
- Cost: **$0**
- Load: Minimal (database lookup)

### Cache Miss:
- Response time: **~16 seconds** (normal RAG)
- Cost: **$0.002-0.01**
- Load: Full (vector search + LLM)

### Improvement:
- **56x faster** for cache hits
- **100% cost savings** for cached responses
- **Growing effectiveness** as usage increases

---

## 📞 Support

For more details, see:
- [Full Documentation](src/rag/cache/README.md)
- [Implementation Summary](CACHE_IMPLEMENTATION_SUMMARY.md)

---

Last Updated: October 9, 2025
