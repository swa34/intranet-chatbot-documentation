# Adding New Features
**Guide to extending the CAES Intranet Help Bot**

This document explains how to add common types of features to the system.

## Table of Contents

- [Adding a New Document Source](#adding-a-new-document-source)
- [Adding a New API Endpoint](#adding-a-new-api-endpoint)
- [Adding a New Metadata Filter](#adding-a-new-metadata-filter)
- [Adding a New Cache Strategy](#adding-a-new-cache-strategy)
- [Modifying the RAG Pipeline](#modifying-the-rag-pipeline)

---

## Adding a New Document Source

### Step 1: Create a Crawler

Create `src/rag/crawlers/crawlNewSource.js`:

```javascript
import fs from 'fs';
import path from 'path';
import axios from 'axios';
import turndown from 'turndown';

const OUTPUT_DIR = path.resolve('docs', 'newsource-site');

async function crawlNewSource() {
  // Create output directory
  if (!fs.existsSync(OUTPUT_DIR)) {
    fs.mkdirSync(OUTPUT_DIR, { recursive: true });
  }

  // Fetch pages
  const pages = await fetchPages(); // Your logic

  for (const page of pages) {
    const html = await axios.get(page.url);
    const markdown = turndownService.turndown(html.data);

    // Save as markdown
    const filename = `${page.slug}.md`;
    const content = `---\nurl: ${page.url}\ntitle: ${page.title}\n---\n\n${markdown}`;

    fs.writeFileSync(path.join(OUTPUT_DIR, filename), content);
  }

  console.log(`Crawled ${pages.length} pages`);
}

crawlNewSource().catch(console.error);
```

### Step 2: Add npm Script

In `package.json`:

```json
{
  "scripts": {
    "crawl:newsource": "node src/rag/crawlers/crawlNewSource.js",
    "ingest:newsource": "node --max-old-space-size=8192 src/rag/ingestion/ingest.js --source newsource"
  }
}
```

### Step 3: Update Ingestion Script

In `src/rag/ingestion/ingest.js`, add source mapping:

```javascript
if (source === 'newsource') {
  SOURCE_DIR = path.resolve('docs', 'newsource-site');
}
```

### Step 4: Run Crawler and Ingest

```bash
npm run crawl:newsource
npm run ingest:newsource
```

---

## Adding a New API Endpoint

### Step 1: Define the Endpoint

In `src/server.js`:

```javascript
// Add after existing endpoints

/**
 * New endpoint description
 * GET /api/new-feature
 */
app.get('/api/new-feature', requireApiKey, async (req, res) => {
  try {
    const { param1, param2 } = req.query;

    // Validate input
    if (!param1) {
      return res.status(400).json({
        error: 'Bad Request',
        message: 'param1 is required'
      });
    }

    // Your logic
    const result = await processNewFeature(param1, param2);

    // Return response
    res.json({
      success: true,
      data: result,
      timestamp: new Date().toISOString()
    });

  } catch (error) {
    console.error('Error in new-feature:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: NODE_ENV === 'development' ? error.message : 'Something went wrong'
    });
  }
});
```

### Step 2: Implement Logic

Create `src/features/newFeature.js`:

```javascript
export async function processNewFeature(param1, param2) {
  // Your implementation
  return { result: 'data' };
}
```

### Step 3: Document the Endpoint

In `documentation/guides/API_REFERENCE.md`, add:

```markdown
### Get New Feature Data

**Endpoint**: `GET /api/new-feature`

**Query Parameters**:
- `param1` (required): Description
- `param2` (optional): Description

**Response**:
\`\`\`json
{
  "success": true,
  "data": {...}
}
\`\`\`

**Example**:
\`\`\`bash
curl "http://localhost:3000/api/new-feature?param1=value" \
  -H "x-api-key: your-key"
\`\`\`
```

### Step 4: Test

```bash
curl "http://localhost:3000/api/new-feature?param1=test" \
  -H "x-api-key: your-key"
```

---

## Adding a New Metadata Filter

### Step 1: Enhance Metadata During Ingestion

In `src/rag/ingestion/ingest-enhanced.js`:

```javascript
export function enhanceMetadata(baseMetadata, filePath, textContent) {
  const enhanced = { ...baseMetadata };

  // Add your new metadata field
  if (textContent.includes('your-keyword')) {
    enhanced.newCategory = 'special_category';
    enhanced.priority = 10;
  }

  return enhanced;
}
```

### Step 2: Add Filter Detection

In `src/rag/vector-ops/retrieve.js`:

```javascript
function detectMetadataFilters(query) {
  const queryLower = query.toLowerCase();

  // Add your filter logic
  if (queryLower.includes('your keyword')) {
    return {
      filter: { newCategory: 'special_category' },
      reason: 'Detected your keyword → filtering by newCategory'
    };
  }

  // ... existing filters
}
```

### Step 3: Re-ingest Documents

```bash
npm run ingest:yoursource -- --purge
```

---

## Adding a New Cache Strategy

### Step 1: Implement Cache Logic

Create `src/rag/cache/newCacheStrategy.js`:

```javascript
export class NewCacheStrategy {
  constructor() {
    // Initialize
  }

  async get(key) {
    // Retrieval logic
  }

  async set(key, value, options) {
    // Storage logic
  }

  async invalidate(key) {
    // Deletion logic
  }
}
```

### Step 2: Integrate with Response Cache

In `src/rag/cache/responseCache.js`:

```javascript
import { NewCacheStrategy } from './newCacheStrategy.js';

class ResponseCache {
  constructor() {
    this.strategies = [
      new RedisCache(),
      new PostgreSQLCache(),
      new NewCacheStrategy() // Add your strategy
    ];
  }

  async get(question, sessionId) {
    // Try each strategy in order
    for (const strategy of this.strategies) {
      const result = await strategy.get(question);
      if (result) return result;
    }
    return null;
  }
}
```

---

## Modifying the RAG Pipeline

### Adding a Pre-Processing Step

In `src/rag/vector-ops/retrieve.js`:

```javascript
export async function retrieveRelevantChunks(userQuestion, topK = 8, sessionId = null) {
  // ... existing cache check ...

  // ADD YOUR PRE-PROCESSING HERE
  const preprocessedQuestion = await yourPreprocessingFunction(userQuestion);

  // Continue with existing pipeline
  const expandedQuery = acronymExpander.expandQuery(preprocessedQuestion);
  // ...
}
```

### Adding a Post-Processing Step

```javascript
// After response generation
let answer = response.choices[0].message.content;

// ADD YOUR POST-PROCESSING HERE
answer = await yourPostProcessingFunction(answer);

// Continue with existing post-processing
answer = answer.replace(/<a\s+(?:[^>]*?\s+)?href=(["'])(.*?)\1[^>]*>(.*?)<\/a>/gi, ...);
```

### Adding a Custom Scoring Algorithm

```javascript
// In src/rag/feedback/customScorer.js
export async function customScoring(matches, query) {
  return matches.map(match => {
    const baseScore = match.score;

    // Your custom scoring logic
    let boost = 0;
    if (match.metadata.priority > 8) boost += 0.05;
    if (match.metadata.recency < 30) boost += 0.03;

    return {
      ...match,
      adjustedScore: Math.min(baseScore + boost, 1.0)
    };
  });
}
```

Then integrate in `retrieve.js`:

```javascript
import { customScoring } from '../feedback/customScorer.js';

// After Pinecone query
matches = await customScoring(matches, userQuestion);
```

---

## Best Practices

1. **Test thoroughly**: Test new features in isolation before integrating
2. **Document as you go**: Update API docs and architecture docs
3. **Follow existing patterns**: Maintain consistency with existing code
4. **Add error handling**: Always handle potential errors gracefully
5. **Consider performance**: Profile and optimize if needed
6. **Update tests**: Add tests for new features

---

## Related Documentation

- [Code Recipes](../guides/CODE_RECIPES.md)
- [System Overview](SYSTEM_OVERVIEW.md)
- [RAG Pipeline](RAG_PIPELINE.md)
- [Development Workflow](../guides/DEVELOPMENT_WORKFLOW.md)

---

**Last Updated**: October 2025
