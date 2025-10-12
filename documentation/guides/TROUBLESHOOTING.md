# Troubleshooting Guide
**Common issues and solutions for the CAES Intranet Help Bot**

This guide helps you diagnose and fix common problems during development and deployment.

## Table of Contents

- [Server Won't Start](#server-wont-start)
- [Database Connection Issues](#database-connection-issues)
- [Pinecone Connection Issues](#pinecone-connection-issues)
- [OpenAI API Errors](#openai-api-errors)
- [Performance Issues](#performance-issues)
- [Cache Issues](#cache-issues)
- [Document Ingestion Problems](#document-ingestion-problems)
- [Authentication Issues](#authentication-issues)

---

## Server Won't Start

### Error: Port Already in Use

```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solution 1 - Kill the process**:
```bash
# Find process using port 3000
lsof -ti:3000

# Kill it
kill -9 $(lsof -ti:3000)

# Windows
netstat -ano | findstr :3000
taskkill /PID <PID> /F
```

**Solution 2 - Use different port**:
```bash
PORT=3001 npm run dev
```

---

### Error: Missing Environment Variables

```
Error: Missing required environment variable: OPENAI_API_KEY
```

**Solution**:
1. Check `.env` file exists in root directory
2. Verify all required variables are set:
```env
OPENAI_API_KEY=sk-...
PINECONE_API_KEY=...
CHATBOT_API_KEY=...
```

---

### Error: Module Not Found

```
Error: Cannot find module 'express'
```

**Solution**:
```bash
# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

---

## Database Connection Issues

### Error: Connection Refused (PostgreSQL)

```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Causes**:
- PostgreSQL not running
- Wrong connection URL
- Firewall blocking connection

**Solutions**:

**Check if PostgreSQL is running**:
```bash
# macOS
brew services list | grep postgresql

# Ubuntu
sudo systemctl status postgresql

# Start if not running
brew services start postgresql  # macOS
sudo systemctl start postgresql # Ubuntu
```

**Test connection manually**:
```bash
psql -U postgres -d chatbot_db
```

**Check environment variable**:
```env
DATABASE_URL=postgresql://user:password@localhost:5432/chatbot_db
```

---

### Error: Database Does Not Exist

```
Error: database "chatbot_db" does not exist
```

**Solution**:
```bash
# Create database
createdb chatbot_db

# Or using psql
psql -U postgres
CREATE DATABASE chatbot_db;
```

---

### Error: Too Many Connections

```
Error: sorry, too many clients already
```

**Solution**:
Reduce connection pool size in code or increase PostgreSQL max_connections:

```javascript
// In your database client
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10 // Reduce from 20
});
```

---

## Pinecone Connection Issues

### Error: Invalid API Key

```
PineconeArgumentError: Invalid Pinecone API key
```

**Solution**:
1. Verify API key in `.env`:
```env
PINECONE_API_KEY=your-key-here
```

2. Check API key in Pinecone dashboard:
   - Go to https://app.pinecone.io/
   - Navigate to API Keys
   - Copy correct key

---

### Error: Index Not Found

```
Error: Index 'uga-intranet-index' not found
```

**Solution**:

**Option 1 - Create index manually** (Pinecone dashboard):
1. Go to https://app.pinecone.io/
2. Click "Create Index"
3. Settings:
   - Name: `uga-intranet-index`
   - Dimensions: 3072
   - Metric: cosine
   - Cloud: AWS
   - Region: us-east-1

**Option 2 - Let ingestion script create it**:
```bash
npm run ingest:dropbox -- --recreate-index
```

---

### Error: Dimension Mismatch

```
Error: Dimension mismatch: expected 3072, got 1536
```

**Cause**: Wrong embedding model or index configuration

**Solution**:
Ensure using `text-embedding-3-large`:
```env
EMBED_MODEL=text-embedding-3-large
```

If index has wrong dimensions, recreate it:
```bash
npm run ingest:dropbox -- --recreate-index
```

---

## OpenAI API Errors

### Error: Invalid API Key

```
Error: Incorrect API key provided
```

**Solution**:
1. Verify API key: https://platform.openai.com/api-keys
2. Update `.env`:
```env
OPENAI_API_KEY=sk-...
```

---

### Error: Rate Limit Exceeded

```
Error: Rate limit reached for requests
```

**Solution**:

**Short-term**:
- Wait 60 seconds
- Reduce concurrent requests

**Long-term**:
- Increase OpenAI tier (pay as you go)
- Implement request queue
- Add exponential backoff:

```javascript
async function retryWithBackoff(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (error.status === 429 && i < retries - 1) {
        await new Promise(r => setTimeout(r, 2 ** i * 1000));
        continue;
      }
      throw error;
    }
  }
}
```

---

### Error: Context Length Exceeded

```
Error: This model's maximum context length is 128000 tokens
```

**Solution**:

**Reduce context size**:
```javascript
// Limit number of document chunks
const matches = await retrieveRelevantChunks(query, 5); // Reduced from 8
```

**Or trim long documents**:
```javascript
const trimmedText = text.substring(0, 10000); // Max 10K chars
```

---

## Performance Issues

### Slow Response Times (>3s)

**Diagnose**:
```javascript
// Check timing breakdown in logs
console.log('⏱️ Retrieval Performance:', {
  embedding: `${embeddingTime}ms`,
  pineconeQuery: `${queryTime}ms`,
  generation: `${generationTime}ms`,
  total: `${totalTime}ms`
});
```

**Solutions**:

**If embedding is slow (>500ms)**:
- Check OpenAI API status
- Use smaller embedding model (not recommended)

**If Pinecone query is slow (>500ms)**:
- Reduce topK: `retrieveRelevantChunks(query, 5)`
- Check Pinecone dashboard for issues
- Consider upgrading Pinecone plan

**If generation is slow (>2s)**:
- Use faster model: `GEN_MODEL=gpt-4o-mini`
- Enable streaming for better UX
- Reduce max_tokens

**If re-ranking is slow**:
```env
# Disable if not needed
ENABLE_LLM_RERANKING=false
```

---

### High Memory Usage

**Symptom**: Node.js crashes with "out of memory"

**Solution 1 - Increase heap size**:
```bash
node --max-old-space-size=4096 src/server.js
```

**Solution 2 - Fix memory leaks**:
- Check for large object accumulation
- Clear conversation history periodically
- Monitor with `process.memoryUsage()`

---

## Cache Issues

### Cache Not Working

**Symptom**: Same questions not returning instant responses

**Diagnose**:
```bash
# Check cache stats
curl http://localhost:3000/admin/cache/stats \
  -H "x-api-key: your-key"
```

**Solutions**:

**If cache is disabled**:
```env
ENABLE_RESPONSE_CACHE=true
```

**If Redis not connected**:
```bash
# Test Redis
redis-cli ping
# Should return: PONG

# If not running
redis-server
```

**If PostgreSQL cache not working**:
```sql
-- Check if table exists
\dt cached_responses

-- Check entries
SELECT COUNT(*) FROM cached_responses WHERE is_active = TRUE;
```

---

### Cache Returning Wrong Answers

**Solution**: Clear cache
```bash
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: your-key" \
  -H "Content-Type: application/json" \
  -d '{"clearAll": true}'
```

---

## Document Ingestion Problems

### No Documents Found

```
warn: No files found. Nothing to ingest.
```

**Solution**:
Check documents exist in correct directory:
```bash
ls -la docs/dropbox/
```

If empty, process documents first:
```bash
npm run process:all-georgia-counts
```

---

### PDF Processing Fails

```
Error: Failed to read/parse document.pdf
```

**Solution**:

**Try Python processor** (better PDF support):
```bash
cd python
python document_processor.py "path/to/document.pdf"
```

**Or skip PDFs temporarily**:
```bash
npm run ingest:dropbox -- --skip-pdf
```

---

### Ingestion Stops Midway

**Causes**:
- OpenAI rate limit
- Out of memory
- Network issues

**Solutions**:

**Add error handling**:
```javascript
try {
  await processFile(file);
} catch (error) {
  console.error(`Failed to process ${file}:`, error);
  continue; // Skip and continue
}
```

**Process in smaller batches**:
```bash
# Process one source at a time
npm run ingest:dropbox
npm run ingest:olod
```

---

## Authentication Issues

### Login Fails

**Symptom**: "Invalid credentials" even with correct password

**Solution**:

**Check user exists**:
```sql
SELECT * FROM users WHERE username = 'your-username';
```

**Reset password**:
```javascript
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash('newpassword', 10);
console.log(hash);

// Update in database
UPDATE users SET password_hash = 'hash-from-above' WHERE username = 'your-username';
```

---

### Session Not Persisting

**Symptom**: Logged out immediately after login

**Solutions**:

**Check cookie settings**:
```javascript
app.use(session({
  cookie: {
    secure: false, // Set to false for development
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));
```

**Check sessions table**:
```sql
SELECT * FROM sessions;
```

---

## Common Error Messages

### "Chatbot not responding"

**Diagnose**:
1. Check server is running: `curl http://localhost:3000/health`
2. Check logs for errors
3. Check OpenAI API status
4. Test API directly:

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "test"}'
```

---

### "No sources found"

**Causes**:
- No documents ingested
- Low similarity scores
- Wrong namespace

**Solutions**:
```bash
# Check vector count
curl http://localhost:3000/debug-query?q=test \
  -H "x-api-key: your-key"

# Lower threshold temporarily
MIN_SIMILARITY=0.5
```

---

## Getting More Help

### Enable Debug Mode

```env
DEBUG_RAG=true
```

This prints detailed logs:
```
--- RAG DEBUG ---
Question: How do I report?
topScore: 0.89, threshold: 0.75
matches: [...]
------------------
```

### Check Application Logs

**View recent logs**:
```bash
# If using pm2
pm2 logs

# If running directly
# Logs appear in console
```

### Health Check

```bash
curl http://localhost:3000/health
```

Expected response:
```json
{
  "status": "ok",
  "timestamp": "2025-10-12T10:30:00Z",
  "environment": "development"
}
```

---

## Still Stuck?

1. **Check existing documentation**
2. **Search GitHub issues**
3. **Review error logs carefully**
4. **Test each component individually**
5. **Ask for help** with specific error messages and logs

---

## Related Documentation

- [Quick Start](QUICK_START.md)
- [Development Workflow](DEVELOPMENT_WORKFLOW.md)
- [API Reference](API_REFERENCE.md)
- [System Overview](../architecture/SYSTEM_OVERVIEW.md)

---

**Last Updated**: October 2025
