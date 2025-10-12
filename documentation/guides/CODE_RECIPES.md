# Code Recipes
**Common code patterns and examples for the CAES Intranet Help Bot**

This guide provides practical code examples for common development tasks.

## Table of Contents

- [Adding a New API Endpoint](#adding-a-new-api-endpoint)
- [Querying the Database](#querying-the-database)
- [Working with Pinecone](#working-with-pinecone)
- [Calling OpenAI API](#calling-openai-api)
- [Caching Responses](#caching-responses)
- [Processing Documents](#processing-documents)

---

## Adding a New API Endpoint

### Basic GET Endpoint

```javascript
// In src/server.js

app.get('/api/my-endpoint', requireApiKey, async (req, res) => {
  try {
    const { param1, param2 } = req.query;

    // Your logic here
    const result = await yourFunction(param1, param2);

    res.json({
      success: true,
      data: result,
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    console.error('Error in my-endpoint:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: NODE_ENV === 'development' ? error.message : 'Something went wrong'
    });
  }
});
```

### POST Endpoint with Validation

```javascript
app.post('/api/my-endpoint', requireApiKey, async (req, res) => {
  try {
    const { requiredField, optionalField } = req.body;

    // Validate input
    if (!requiredField) {
      return res.status(400).json({
        error: 'Bad Request',
        message: 'requiredField is required'
      });
    }

    // Process request
    const result = await processData(requiredField, optionalField);

    res.json({
      success: true,
      data: result
    });
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

---

## Querying the Database

### Simple Query

```javascript
import { getConversationMemory } from './storage/conversationMemoryPostgres.js';

const memory = getConversationMemory();
await memory.initialize();

const result = await memory.pool.query(
  'SELECT * FROM conversations WHERE session_id = $1',
  [sessionId]
);

console.log(result.rows);
```

### Query with Aggregation

```javascript
const stats = await memory.pool.query(`
  SELECT
    COUNT(*) as total_conversations,
    COUNT(CASE WHEN feedback_rating = 1 THEN 1 END) as helpful_count,
    ROUND(AVG(response_time_ms)) as avg_response_time
  FROM conversations
  WHERE timestamp > NOW() - INTERVAL '7 days'
`);

console.log(stats.rows[0]);
```

### Insert with Returning

```javascript
const result = await memory.pool.query(`
  INSERT INTO conversations (session_id, question, answer, sources)
  VALUES ($1, $2, $3, $4)
  RETURNING id, timestamp
`, [sessionId, question, answer, JSON.stringify(sources)]);

const insertedId = result.rows[0].id;
```

---

## Working with Pinecone

### Query Vectors

```javascript
import { getPineconeIndex } from './rag/utils/pineconeClient.js';

const index = await getPineconeIndex();
const ns = index.namespace('__default__');

// Generate embedding
const embedRes = await openai.embeddings.create({
  model: 'text-embedding-3-large',
  input: 'your query text'
});

const queryVector = embedRes.data[0].embedding;

// Search
const result = await ns.query({
  vector: queryVector,
  topK: 10,
  includeMetadata: true,
  filter: {
    category: 'ga_counts_app'
  }
});

console.log(result.matches);
```

### Upsert Vectors

```javascript
const vectors = [
  {
    id: 'doc1_chunk0',
    values: embedding, // 3072-dim array
    metadata: {
      source: 'document.md',
      text: 'Document text...',
      url: 'https://...'
    }
  }
];

await index.upsert(vectors, '__default__');
```

### Delete by ID

```javascript
await index.namespace('__default__').deleteOne('doc1_chunk0');
```

---

## Calling OpenAI API

### Generate Embeddings

```javascript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await openai.embeddings.create({
  model: 'text-embedding-3-large',
  input: 'Text to embed'
});

const embedding = response.data[0].embedding; // 3072-dim vector
```

### Chat Completion (GPT-4o)

```javascript
const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Hello!' }
  ],
  temperature: 0.7,
  max_tokens: 1000
});

const answer = response.choices[0].message.content;
```

### Responses API (GPT-5)

```javascript
const response = await openai.responses.create({
  model: 'gpt-5',
  instructions: 'System instructions here',
  input: 'User message',
  reasoning: { effort: 'minimal' },
  text: { verbosity: 'low' }
});

const answer = response.output_text;
```

### Streaming Response

```javascript
const stream = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Tell me a story' }],
  stream: true
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content || '';
  process.stdout.write(content);
}
```

---

## Caching Responses

### Store in Cache

```javascript
import { getResponseCache } from './rag/cache/responseCache.js';

const cache = getResponseCache();
await cache.initialize();

await cache.set(
  'How do I report in GaCounts?',
  'To report in GaCounts...',
  sources,
  {
    confidence: 0.92,
    ttlDays: 30,
    createdBy: 'auto'
  }
);
```

### Retrieve from Cache

```javascript
const cachedResult = await cache.get('How do I report in GaCounts?', sessionId);

if (cachedResult) {
  console.log('Cache hit!');
  return cachedResult.response;
}
```

### Clear Cache

```javascript
// Clear specific entry
await cache.invalidate(cacheId);

// Clear all Redis cache
if (cache.clearRedisCache) {
  await cache.clearRedisCache();
}
```

---

## Processing Documents

### Read and Chunk Document

```javascript
import fs from 'fs';
import { chunkText } from './rag/ingestion/chunk.js';

const text = fs.readFileSync('document.md', 'utf-8');
const chunks = chunkText(text, 1200, 200);

console.log(`Created ${chunks.length} chunks`);
```

### Extract Text from PDF (Node.js)

```javascript
import pdfParse from 'pdf-parse';
import fs from 'fs';

const dataBuffer = fs.readFileSync('document.pdf');
const data = await pdfParse(dataBuffer);

console.log(data.text);
```

### Process with Python

```python
# python/document_processor.py usage
import sys
from document_processor import extract_text_from_pdf

pdf_path = sys.argv[1]
text = extract_text_from_pdf(pdf_path)
print(text)
```

```bash
# Call from Node.js
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

const { stdout } = await execAsync(`python python/document_processor.py "${pdfPath}"`);
console.log(stdout);
```

---

## Working with Sessions

### Create Session

```javascript
import { getConversationMemory } from './storage/conversationMemoryPostgres.js';

const memory = getConversationMemory();
const sessionId = memory.createSessionId();

console.log(sessionId); // sess_abc123xyz
```

### Add Conversation Turn

```javascript
await memory.addTurn(
  sessionId,
  'How do I report?',
  'To report in GaCounts...',
  sources,
  1456 // response time ms
);
```

### Get Conversation History

```javascript
const history = await memory.getHistory(sessionId);

for (const turn of history) {
  console.log(`Q: ${turn.question}`);
  console.log(`A: ${turn.answer}`);
}
```

---

## Error Handling Patterns

### Try-Catch with Logging

```javascript
async function riskyOperation() {
  try {
    const result = await externalAPI();
    return result;
  } catch (error) {
    console.error('Operation failed:', {
      error: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString()
    });
    throw error; // Re-throw if caller needs to handle
  }
}
```

### Retry with Exponential Backoff

```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
const result = await retryWithBackoff(() => callUnreliableAPI());
```

---

## Testing Helpers

### Mock OpenAI Response

```javascript
// For testing without calling API
const mockOpenAI = {
  embeddings: {
    create: async () => ({
      data: [{ embedding: new Array(3072).fill(0.1) }]
    })
  },
  chat: {
    completions: {
      create: async () => ({
        choices: [{ message: { content: 'Mock response' } }]
      })
    }
  }
};
```

### Test Database Query

```javascript
async function testQuery() {
  const memory = getConversationMemory();
  await memory.initialize();

  const result = await memory.pool.query('SELECT NOW()');
  console.log('Database connected:', result.rows[0]);

  await memory.destroy();
}
```

---

## Related Documentation

- [API Reference](API_REFERENCE.md)
- [Adding New Features](../architecture/ADDING_NEW_FEATURES.md)
- [Development Workflow](DEVELOPMENT_WORKFLOW.md)

---

**Last Updated**: October 2025
