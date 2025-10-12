# Production Migration Plan
## Switching from Staging to Production WordPress Intranet

**Date:** 2025-10-08
**Status:** ✅ Ready to Execute

---

## Strategy: Separate Directories for Staging & Production

✅ **Implemented:** Files saved to separate directories, both kept in Pinecone

### Why Separate Directories?
1. **Clean separation:** `docs/intranet-staging/` vs `docs/intranet-production/`
2. **Re-ingest either:** Can re-crawl/re-ingest staging or production independently
3. **Safety:** Staging files preserved as backup
4. **Flexibility:** Easy to purge either environment from Pinecone

### What Changed (Already Done):
- ✅ Moved existing staging files: `docs/intranet/` → `docs/intranet-staging/`
- ✅ Crawler now outputs to environment-specific directories
- ✅ Ingestion supports `--source intranet-staging` and `--source intranet-production`
- ✅ Metadata includes `sourceEnvironment: 'production'` or `'staging'`
- ✅ Query filter automatically prefers production over staging

---

## Implementation Steps

### 1. Crawl Production Site (30-60 min)

```bash
cd python
python crawl_wordpress_intranet.py --prod
```

**What happens:**
- Crawls `https://intranet.caes.uga.edu/` (production)
- Uses production auth token (already configured)
- Saves to `docs/intranet-production/` with `Environment: production` metadata
- Safe: 6-second delay between requests (well under Kinsta limits)

**Expected output:**
- ~100-300 markdown files in `docs/intranet-production/`
- Each file has `**Environment:** production` in metadata
- Crawl summary in `docs/intranet-production/crawl_summary.json`

---

### 2. Ingest Production Content (10-20 min)

```bash
npm run ingest -- --source intranet-production
```

**What happens:**
- Chunks production markdown files from `docs/intranet-production/`
- Embeds with `text-embedding-3-large`
- Adds metadata:
  - `sourceType: 'wordpress_intranet'`
  - `sourceEnvironment: 'production'` (auto-detected from path)
  - `category: 'intranet_pages'`
  - `priority: 10` (highest)
- Upserts to Pinecone (same namespace as staging)

**Result:** Production + staging both in Pinecone, distinguished by `sourceEnvironment`

---

### 3. Auto-Filtering (Already Implemented)

**Query behavior** ([retrieve.js:298-309](src/rag/vector-ops/retrieve.js#L298-309)):
```javascript
filter: {
  $or: [
    { sourceEnvironment: 'production' },        // Prefer production
    { sourceEnvironment: { $exists: false } },  // Include non-intranet
    { sourceType: { $ne: 'wordpress_intranet' } } // Include other sources
  ]
}
```

**Effect:** Staging content is automatically excluded from queries (production preferred)

---

## Optional: Purge Staging Later

After confirming production works (wait 1-2 weeks), remove staging:

```bash
# Delete staging vectors from Pinecone
node -e "
import { getPineconeIndex } from './src/rag/utils/pineconeClient.js';
const index = await getPineconeIndex();
const ns = index.namespace('__default__');

// Find all staging vectors
const result = await ns.query({
  vector: Array(3072).fill(0),
  topK: 10000,
  filter: { sourceEnvironment: 'staging' },
  includeMetadata: true
});

const ids = result.matches.map(m => m.id);
console.log('Found', ids.length, 'staging vectors');

if (ids.length > 0) {
  await ns.deleteMany(ids);
  console.log('Deleted staging vectors');
}
"

# Optional: Delete staging files
rm -rf docs/intranet-staging
```

---

## Directory Structure

```
docs/
├── intranet-staging/          ✅ Existing staging crawl (moved from docs/intranet/)
│   ├── *.md                   (Source URLs: stg-intranetcaesugaedu-staging.kinsta.cloud)
│   ├── crawl_summary.json
│   └── crawl_inventory.csv
│
├── intranet-production/       🆕 New production crawl
│   ├── *.md                   (Source URLs: intranet.caes.uga.edu)
│   ├── crawl_summary.json
│   └── crawl_inventory.csv
│
└── [other sources...]
```

---

## Verification Checklist

After production ingestion:

- [ ] Check vector count increased: Visit Pinecone console
- [ ] Test query returns production URLs: Ask "what is the intranet?"
- [ ] Verify metadata: `node src/rag/vector-ops/sampleMetadata.js`
- [ ] Confirm staging is filtered out: Check response URLs (should be `intranet.caes.uga.edu`, not staging)
- [ ] Compare staging vs production: Count files in both directories

---

## Rollback Plan

If production has issues:

1. **Temporarily query staging:** Modify filter in `retrieve.js` to use staging
2. **Re-crawl production:** Just run crawler again (overwrites files)
3. **Delete production vectors:** Use query with `sourceEnvironment: 'production'` filter

---

## Timeline

| Step | Duration | Can Run Concurrently? |
|------|----------|----------------------|
| 1. Crawl production | 30-60 min | No (sequential) |
| 2. Ingest production | 10-20 min | No (after crawl) |
| 3. Test queries | 5 min | After ingest |
| **Total** | **45-85 min** | - |

---

## Cost Impact

**Pinecone storage:**
- Current: ~50K-100K vectors
- After production: ~60K-120K vectors (+10-20K for production intranet)
- Well within Starter plan limits (no additional cost)

**OpenAI embeddings:**
- ~100-300 pages × ~5 chunks/page = ~500-1500 embeddings
- Cost: ~$0.02-0.05 (negligible)

---

## Commands Reference

```bash
# Crawl production
cd python
python crawl_wordpress_intranet.py --prod

# Crawl staging (if needed later)
cd python
python crawl_wordpress_intranet.py  # defaults to staging

# Ingest production
npm run ingest -- --source intranet-production

# Ingest staging (if re-crawled)
npm run ingest -- --source intranet-staging

# Check production vectors
node src/rag/vector-ops/sampleMetadata.js

# Find production content
node src/rag/vector-ops/findByUrl.js "intranet.caes.uga.edu"

# Find staging content
node src/rag/vector-ops/findByUrl.js "stg-intranetcaesugaedu-staging.kinsta.cloud"

# Test query (via server)
curl -X POST http://localhost:3010/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "what is on the intranet?", "user": {"collegeid": "12345"}}'
```

---

## Questions?

**Q: Will production overwrite staging files?**
A: No! Separate directories: `docs/intranet-production/` vs `docs/intranet-staging/`

**Q: Can we re-crawl staging without affecting production?**
A: Yes! Just run crawler without `--prod` flag and re-ingest `--source intranet-staging`

**Q: What if production crawl fails?**
A: Staging still works. Staging files and vectors are untouched. Re-run the crawler to retry.

**Q: Should we purge staging immediately?**
A: No - wait 1-2 weeks to ensure production is working well, then purge staging vectors and files.

**Q: Will users see duplicate results (staging + production)?**
A: No - the filter excludes staging automatically when querying.

---

**Status:** ✅ Ready to execute
**Next step:** Run `cd python && python crawl_wordpress_intranet.py --prod`
