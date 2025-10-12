# Quick Start Guide
**Get the CAES Intranet Help Bot running in 5 minutes**

This guide will help you set up a local development environment as quickly as possible.

## Prerequisites

Before you begin, ensure you have:

- ✅ **Node.js 20.16+** - [Download here](https://nodejs.org/)
- ✅ **npm 10.0+** - Comes with Node.js
- ✅ **Git** - [Download here](https://git-scm.com/)
- ✅ **PostgreSQL** - [Download here](https://www.postgresql.org/download/) (optional for local dev)
- ✅ **Python 3.8+** - [Download here](https://www.python.org/) (optional, for document processing)

**Required API Keys** (you'll need these):
- OpenAI API key
- Pinecone API key

## 5-Minute Setup

### Step 1: Clone the Repository (30 seconds)

```bash
git clone https://github.com/swa34/CAES-INTRANET-HELP-BOT.git
cd CAES-INTRANET-HELP-BOT
```

Or if using SSH:
```bash
git clone git@github.com:swa34/CAES-INTRANET-HELP-BOT.git
cd CAES-INTRANET-HELP-BOT
```

### Step 2: Install Dependencies (1 minute)

```bash
# Install Node.js dependencies
npm install

# (Optional) Install Python dependencies for document processing
cd python
pip install -r requirements.txt
cd ..
```

### Step 3: Configure Environment Variables (2 minutes)

Create a `.env` file in the root directory:

```bash
# Copy the example (if it exists) or create new file
cp .env.example .env
```

Edit `.env` with your configuration:

```env
# ============================================
# REQUIRED: Core Configuration
# ============================================

# OpenAI API Configuration
OPENAI_API_KEY=sk-your-openai-api-key-here
GEN_MODEL=gpt-4o                          # Main chat model
EMBED_MODEL=text-embedding-3-large        # Embedding model

# Pinecone Configuration
PINECONE_API_KEY=your-pinecone-api-key-here
PINECONE_INDEX_NAME=uga-intranet-index
PINECONE_NAMESPACE=__default__

# API Security
CHATBOT_API_KEY=your-secure-api-key-here  # Create a strong random key

# Server Configuration
PORT=3000
NODE_ENV=development

# ============================================
# OPTIONAL: Database Configuration
# ============================================

# PostgreSQL (for conversation memory, cache, feedback)
# If not provided, some features will be limited
DATABASE_URL=postgresql://user:password@localhost:5432/chatbot_db

# Redis (for performance caching - optional but recommended)
REDIS_URL=redis://localhost:6379

# ============================================
# OPTIONAL: Feature Flags
# ============================================

# Enable/disable features
DEBUG_RAG=false                           # Enable detailed RAG logging
ENABLE_CLARIFICATION_MODE=true            # Ask clarifying questions
ENABLE_RESPONSE_CACHE=true                # Cache responses
ENABLE_LLM_RERANKING=true                 # Re-rank search results

# Similarity thresholds
MIN_SIMILARITY=0.75                       # Minimum similarity for matches

# ============================================
# OPTIONAL: Google Sheets (for feedback backup)
# ============================================

# GOOGLE_SHEETS_ID=your-spreadsheet-id
# GOOGLE_SERVICE_ACCOUNT_EMAIL=your-service-account@project.iam.gserviceaccount.com
# GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# ============================================
# OPTIONAL: GPT-5 Configuration (if using GPT-5 models)
# ============================================

# GPT5_REASONING_EFFORT=minimal            # minimal, low, medium, high
# GPT5_VERBOSITY=low                       # low, medium, high
```

**Quick API Key Setup:**

1. **OpenAI API Key**:
   - Go to https://platform.openai.com/api-keys
   - Create new secret key
   - Copy to `OPENAI_API_KEY`

2. **Pinecone API Key**:
   - Go to https://app.pinecone.io/
   - Create account (free tier available)
   - Copy API key to `PINECONE_API_KEY`
   - Create an index named `uga-intranet-index` with:
     - Dimensions: 3072
     - Metric: cosine
     - Cloud: AWS
     - Region: us-east-1

3. **Chatbot API Key**:
   - Generate a random secure key: `openssl rand -hex 32`
   - Or use: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

### Step 4: Start the Server (30 seconds)

```bash
npm run dev
```

You should see:
```
 Intranet Chatbot server running on port 3000
Environment: development
Debug mode: false
Clarification mode: ENABLED
Local URL: http://localhost:3000
Health check: http://localhost:3000/health
🚀 Pre-initializing Pinecone connection...
✅ Pinecone pre-initialized and cached for all future requests
```

### Step 5: Test the Chatbot (1 minute)

**Option A: Using curl**

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-secure-api-key-here" \
  -d '{"message": "Hello, how can you help me?"}'
```

**Option B: Using the Web UI**

1. Open browser to http://localhost:3000
2. You'll be redirected to login (if auth is enabled)
3. Or you can directly access the chat interface

**Option C: Using a REST client**

Import this into Postman/Insomnia:
```json
{
  "method": "POST",
  "url": "http://localhost:3000/chat",
  "headers": {
    "Content-Type": "application/json",
    "x-api-key": "your-secure-api-key-here"
  },
  "body": {
    "message": "How do I report in GaCounts?"
  }
}
```

Expected response:
```json
{
  "answer": "...",
  "sources": [...],
  "responseTime": 1234,
  "sessionId": "..."
}
```

## 🎉 Success!

If you see a response, you're ready to develop! The chatbot is now running locally.

---

## Next Steps

### 1. Load Some Documents (Optional for Testing)

If you want to test with real documents:

```bash
# Process GaCounts documents from Dropbox
npm run process:all-georgia-counts

# Ingest into Pinecone
npm run ingest:dropbox
```

This will take 5-10 minutes depending on the number of documents.

### 2. Set Up PostgreSQL (Recommended)

For full features (conversation memory, caching, feedback):

```bash
# Create a database
createdb chatbot_db

# Run migrations
npm run migrate:auth

# Update .env with connection string
DATABASE_URL=postgresql://localhost/chatbot_db
```

### 3. Set Up Redis (Optional but Recommended)

For better performance:

```bash
# Install Redis
# macOS: brew install redis
# Ubuntu: sudo apt-get install redis-server
# Windows: Use WSL or Docker

# Start Redis
redis-server

# Update .env
REDIS_URL=redis://localhost:6379
```

### 4. Explore the Codebase

- **Main server**: `src/server.js`
- **RAG retrieval**: `src/rag/vector-ops/retrieve.js`
- **Document ingestion**: `src/rag/ingestion/ingest.js`
- **Web UI**: `public/index.html`

See [Code Organization](../architecture/CODE_ORGANIZATION.md) for complete structure.

### 5. Read the Full Documentation

- [Developer Guide](../DEVELOPER_GUIDE.md) - Complete development guide
- [API Reference](API_REFERENCE.md) - All API endpoints
- [Development Workflow](DEVELOPMENT_WORKFLOW.md) - Daily development practices

---

## Common Issues

### Port 3000 Already in Use

```bash
# Find and kill the process
# macOS/Linux:
lsof -ti:3000 | xargs kill -9

# Windows:
netstat -ano | findstr :3000
taskkill /PID <PID> /F

# Or use a different port
PORT=3001 npm run dev
```

### Missing API Keys

```
Error: Missing required environment variable: OPENAI_API_KEY
```

**Solution**: Double-check your `.env` file has all required keys.

### Cannot Connect to Pinecone

```
Error: Failed to connect to Pinecone
```

**Solution**:
1. Verify your Pinecone API key is correct
2. Check that the index name matches: `uga-intranet-index`
3. Ensure the index exists in your Pinecone dashboard

### Module Not Found Errors

```
Error: Cannot find module 'express'
```

**Solution**: Reinstall dependencies
```bash
rm -rf node_modules package-lock.json
npm install
```

For more troubleshooting, see [Troubleshooting Guide](TROUBLESHOOTING.md).

---

## Development Commands Cheat Sheet

```bash
# Start development server
npm run dev

# Start with tunneling (for external access)
npm run dev:all

# Process documents
npm run process:all-georgia-counts
npm run crawl:olod
npm run crawl:oit

# Ingest into Pinecone
npm run ingest:dropbox
npm run ingest:olod

# Database migrations
npm run migrate:auth

# Test cache
curl http://localhost:3000/admin/cache/stats \
  -H "x-api-key: your-key"

# Clear cache
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: your-key" \
  -H "Content-Type: application/json" \
  -d '{"clearAll": true}'
```

---

## What's Next?

Now that you have the chatbot running:

1. **Understand the architecture**: [System Overview](../architecture/SYSTEM_OVERVIEW.md)
2. **Learn the RAG pipeline**: [RAG Pipeline](../architecture/RAG_PIPELINE.md)
3. **Explore the API**: [API Reference](API_REFERENCE.md)
4. **Start developing**: [Development Workflow](DEVELOPMENT_WORKFLOW.md)
5. **Add new features**: [Adding New Features](../architecture/ADDING_NEW_FEATURES.md)

---

## Getting Help

- **Troubleshooting**: [Troubleshooting Guide](TROUBLESHOOTING.md)
- **Documentation**: [Developer Guide](../DEVELOPER_GUIDE.md)
- **GitHub Issues**: [Report a bug](https://github.com/swa34/CAES-INTRANET-HELP-BOT/issues)

Happy coding! 🚀
