# Intranet Chatbot Setup Guide

## Overview

This chatbot has been adapted to crawl and index your internal intranet content, providing a RAG-powered Q&A system for your organization's internal knowledge base.

## Features

- **Authenticated Crawling**: Supports pAuth token authentication for ColdFusion sites
- **Content Extraction**: Intelligent content extraction using Readability
- **Vector Search**: OpenAI embeddings + Pinecone for semantic search
- **Incremental Updates**: Can crawl and re-index content as needed

## Prerequisites

1. Node.js 18+ installed
2. OpenAI API key
3. Pinecone account and API key
4. Authentication token for your intranet sites

## Configuration

### 1. Set Up Environment Variables

Update the `.env` file with your credentials:

```env
# OpenAI Configuration
OPENAI_API_KEY=your_openai_api_key_here
EMBED_MODEL=text-embedding-3-large
GEN_MODEL=gpt-4o-mini

# Pinecone Configuration
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX_NAME=uga-intranet-index
PINECONE_NAMESPACE=

# Crawler Configuration
CRAWL_TOKEN=YOUR_SECRET_TOKEN_HERE  # Replace with actual pAuth token
USE_HEADER_AUTH=true                 # Use header auth (recommended)
MAX_PAGES=100                        # Max pages per crawl
MAX_DEPTH=5                          # Max link depth
CRAWL_DELAY=500                      # Delay between requests (ms)

# Server Configuration
PORT=3000
CHATBOT_API_KEY=your_chatbot_api_key
MIN_SIMILARITY=0.40
DEBUG_RAG=true
```

### 2. Install Dependencies

```bash
npm install
```

## Usage

### Step 1: Crawl Your Intranet

#### Option A: Using npm script (simple)

```bash
npm run crawl:intranet https://your-intranet.com/section/ -- --domains your-intranet.com
```

#### Option B: Direct command with options

```bash
node src/rag/crawlIntranet.js https://your-intranet.com/personnel/ \
  --domains your-intranet.com,another-domain.com \
  --max-pages 50 \
  --max-depth 3
```

#### Authentication Methods

1. **Header Authentication (Default - Recommended)**:
   ```
   X-Crawl-Token: YOUR_SECRET_TOKEN_HERE
   ```

2. **URL Parameter (Testing only)**:
   ```bash
   node src/rag/crawlIntranet.js https://your-site.com/ --url-auth
   ```
   This appends `?crawl_token=YOUR_SECRET_TOKEN_HERE` to URLs

### Step 2: Process and Embed Content

After crawling, process the content and create embeddings:

```bash
npm run ingest:intranet
```

Options:
- `--dry`: Test run without uploading to Pinecone
- `--purge`: Clear existing vectors before ingesting
- `--recreate-index`: Delete and recreate the Pinecone index

### Step 3: Start the Chatbot Server

```bash
npm run dev
```

The server will start on `http://localhost:3000`

### Step 4: Test the Chatbot

Open `http://localhost:3000` in your browser or make API calls:

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your_chatbot_api_key" \
  -d '{"message": "What is the personnel policy on remote work?"}'
```

## File Structure

```
uga-intranet-Bot/
├── docs/
│   ├── web/         # Public web content (existing)
│   └── intranet/    # Crawled intranet content (new)
├── src/
│   ├── rag/
│   │   ├── crawlIntranet.js  # Intranet crawler with auth
│   │   ├── crawlBFS.js       # Original public crawler
│   │   ├── ingest.js          # Process & embed content
│   │   ├── retrieve.js        # Vector search
│   │   └── chunk.js           # Text chunking
│   └── server.js              # Main API server
└── public/                    # Web UI files
```

## Crawling Multiple Sites

### ColdFusion Sites (with pAuth)

```bash
# Personnel site
npm run crawl:intranet https://intranet.uga.edu/personnel/ -- --domains intranet.uga.edu

# Finance site
npm run crawl:intranet https://finance.uga.edu/policies/ -- --domains finance.uga.edu

# Multiple domains at once
npm run crawl:intranet https://site1.uga.edu/ https://site2.uga.edu/ -- --domains site1.uga.edu,site2.uga.edu
```

### WordPress Sites (Future)

WordPress crawling will require different authentication. Options:
1. Add REST API authentication
2. Use WordPress application passwords
3. Configure crawler user with read permissions

## Security Considerations

1. **Token Security**:
   - Never commit tokens to version control
   - Use environment variables for all secrets
   - Rotate tokens regularly

2. **Access Control**:
   - The chatbot API requires `X-API-Key` header
   - Consider adding IP whitelisting in production
   - Use HTTPS in production

3. **Data Privacy**:
   - Crawled content is stored locally in `docs/intranet/`
   - Embeddings are stored in Pinecone
   - Consider data retention policies

## Monitoring & Maintenance

### Check Crawl Results

After crawling, check the summary:

```bash
cat docs/intranet/crawl_summary.json
```

### Debug Vector Search

Test retrieval without the chatbot:

```bash
curl http://localhost:3000/debug-query?q=test \
  -H "X-API-Key: your_chatbot_api_key"
```

### View Pinecone Usage

```bash
npm run check-usage
```

## Troubleshooting

### Authentication Errors

If you see 401/403 errors:
1. Verify `CRAWL_TOKEN` in `.env`
2. Check if token is properly configured on the server
3. Try URL parameter auth for testing: `--url-auth`

### Empty Crawl Results

If no content is extracted:
1. Check selectors being removed in `crawlIntranet.js`
2. Verify Readability can parse your HTML structure
3. Check browser console for the actual page structure

### Low Similarity Scores

If chatbot says "couldn't find information":
1. Lower `MIN_SIMILARITY` in `.env` (try 0.30)
2. Check if content was properly chunked
3. Verify embeddings were created

## Next Steps

1. **Add WordPress Support**: Implement WordPress authentication
2. **Scheduled Crawling**: Set up cron jobs for regular updates
3. **Incremental Updates**: Only crawl changed pages
4. **Multi-namespace Support**: Separate namespaces per department
5. **Access Control**: Department-specific chatbot instances

## Support

For issues or questions:
- Check logs in the console
- Review `docs/intranet/crawl_summary.json`
- Test individual components (crawler, ingest, retrieve)