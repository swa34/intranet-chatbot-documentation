# Code Organization
**Understanding the CAES Intranet Help Bot directory structure and module organization**

This document explains where everything lives in the codebase and how the code is organized.

## Table of Contents

- [Project Structure Overview](#project-structure-overview)
- [Directory Breakdown](#directory-breakdown)
- [Key Files Reference](#key-files-reference)
- [Module Dependencies](#module-dependencies)
- [Naming Conventions](#naming-conventions)
- [Import/Export Patterns](#importexport-patterns)

---

## Project Structure Overview

```
intranetChatbot/
├── src/                          # Node.js application source code
│   ├── admin/                    # Admin dashboard features
│   ├── analyzers/                # AI-powered analysis modules
│   ├── auth/                     # Authentication system
│   ├── middleware/               # Express middleware
│   ├── migrations/               # Database migrations
│   ├── prompts/                  # LLM system prompts
│   ├── rag/                      # RAG pipeline (core logic)
│   ├── routes/                   # API route handlers
│   ├── storage/                  # Data storage modules
│   ├── streaming/                # Streaming response handlers
│   ├── utils/                    # Utility functions
│   └── server.js                 # Main Express server
│
├── python/                       # Python document processors
│   ├── document_processor.py     # PDF/Office file processing
│   ├── dropbox_api_processor.py  # Dropbox API integration
│   ├── crawl_*.py                # Website crawlers
│   └── requirements.txt          # Python dependencies
│
├── public/                       # Frontend web files
│   ├── index.html                # Main chat interface
│   ├── login.html                # Login page
│   ├── admin.html                # Admin dashboard
│   ├── styles.css                # CSS styles
│   └── script.js                 # Frontend JavaScript
│
├── docs/                         # Processed documents (1000+ files)
│   ├── dropbox/                  # Dropbox documents
│   ├── abo-site/                 # ABO website content
│   ├── olod-site/                # OLOD website content
│   ├── oit-site/                 # OIT website content
│   ├── brand-site/               # Brand guidelines
│   └── [other sources]/          # Additional document sources
│
├── documentation/                # Project documentation (YOU ARE HERE)
│   ├── guides/                   # User-facing guides
│   ├── architecture/             # Technical architecture docs
│   ├── testing/                  # Testing documentation
│   ├── optimization/             # Performance optimization docs
│   ├── justifications/           # Business justifications
│   ├── setup/                    # Setup and configuration
│   └── DEVELOPER_GUIDE.md        # Main developer guide
│
├── tests/                        # Test files
├── scripts/                      # Utility scripts
├── config/                       # Configuration files
├── feedback/                     # User feedback data
├── data/                         # Static data files
├── .env                          # Environment variables (not in git)
├── .gitignore                    # Git ignore rules
├── package.json                  # Node.js dependencies and scripts
└── README.md                     # Project overview
```

---

## Directory Breakdown

### `/src` - Main Application Code

The Node.js application source code. All backend logic lives here.

#### `/src/admin` - Admin Dashboard
**Purpose**: Admin-only features for managing the chatbot

```
admin/
├── middleware/
│   └── adminAuth.js              # Admin authentication middleware
├── routes/
│   ├── cacheRoutes.js            # Cache management endpoints
│   ├── conversationRoutes.js     # Conversation history endpoints
│   └── maintenanceRoutes.js      # System maintenance endpoints
└── services/
    └── variationGenerator.js     # Question variation generator
```

**Key Responsibilities**:
- Cache management and statistics
- Conversation history viewing
- Analytics dashboards
- System maintenance operations

#### `/src/analyzers` - AI Analysis Modules
**Purpose**: LLM-powered analysis of user input and feedback

```
analyzers/
├── commentAnalyzer.js            # Analyze user feedback comments
└── questionReframer.js           # Reframe follow-up questions
```

**Key Responsibilities**:
- Detect ambiguous questions
- Generate clarification responses
- Reframe context-dependent questions
- Analyze sentiment and issues in feedback

#### `/src/auth` - Authentication System
**Purpose**: User authentication and session management

```
auth/
├── authRoutes.js                 # Auth endpoints (login, logout, register)
├── authUtils.js                  # Auth utility functions
└── sessionManager.js             # Session management logic
```

**Key Responsibilities**:
- User login/logout/registration
- Session creation and validation
- Password hashing and verification
- Session token generation

#### `/src/middleware` - Express Middleware
**Purpose**: Request processing middleware

```
middleware/
├── authMiddleware.js             # Basic API key auth (legacy)
└── sessionAuthMiddleware.js      # Session-based auth (current)
```

**Key Responsibilities**:
- Authenticate requests
- Validate API keys
- Check session cookies
- Set user context on request object

#### `/src/migrations` - Database Migrations
**Purpose**: Database schema changes

```
migrations/
└── 001_create_users_table.js     # Initial user authentication tables
```

**Key Responsibilities**:
- Create database tables
- Modify schema
- Seed initial data

#### `/src/prompts` - LLM System Prompts
**Purpose**: System prompts for AI models

```
prompts/
└── systemPrompt.txt              # Main chatbot system prompt
```

**Key Responsibilities**:
- Define chatbot behavior
- Set response formatting rules
- Provide context guidelines

#### `/src/rag` - RAG Pipeline (CORE)
**Purpose**: The heart of the application - retrieval and generation logic

```
rag/
├── analytics/
│   └── popularQueries.js         # Track popular questions (Redis)
├── cache/
│   ├── responseCache.js          # Response caching (PostgreSQL)
│   ├── responseCacheWithRedis.js # Response caching with Redis
│   ├── generateFAQCache.js       # Pre-cache FAQ responses
│   ├── feedbackQualityControl.js # Quality control for cached responses
│   ├── testCache.js              # Cache testing utilities
│   └── migrations/               # Cache schema migrations
├── crawlers/
│   ├── crawlIntranet.js          # Generic intranet crawler
│   ├── crawlABO_XML.js           # ABO website (XML sitemap)
│   ├── crawlOLOD.js              # OLOD website
│   ├── crawlOIT.js               # OIT website
│   ├── crawlOMC.js               # OMC website
│   ├── crawlBrand.js             # Brand guidelines site
│   ├── crawlBFS.js               # BFS website
│   └── crawlGaCounts.js          # GaCounts help pages
├── feedback/
│   ├── feedbackLearning.js       # Legacy feedback learning
│   ├── feedbackLearningPostgres.js # PostgreSQL-based learning
│   ├── feedbackLearningSheets.js # Google Sheets learning
│   ├── feedbackAnalyzer.js       # Modern feedback analyzer (ACTIVE)
│   ├── feedback-analyzer.js      # Alternate analyzer
│   └── commentScorer.js          # Score comment quality
├── ingestion/
│   ├── ingest.js                 # Main document ingestion (CRITICAL)
│   ├── ingest-enhanced.js        # Enhanced metadata extraction
│   ├── chunk.js                  # Text chunking logic
│   └── testChunking.js           # Test chunking algorithm
├── utils/
│   ├── pineconeClient.js         # Pinecone connection (basic)
│   ├── pineconeClientWithRedis.js # Pinecone with Redis caching
│   ├── acronymExpander.js        # Expand acronyms (GaCounts → Georgia Counts)
│   ├── dropboxFetcher.js         # Fetch files from Dropbox
│   ├── extractLinks.js           # Extract links from HTML
│   ├── extractABOResources.js    # Parse ABO sitemap
│   ├── extractOITResources.js    # Parse OIT sitemap
│   └── scrapeFromSitemap.js      # Generic sitemap scraper
└── vector-ops/
    ├── retrieve.js               # Main retrieval logic (CRITICAL)
    ├── checkDocument.js          # Verify document in Pinecone
    ├── deleteDocument.js         # Remove document from Pinecone
    ├── findByUrl.js              # Search by URL
    ├── inspectVector.js          # Inspect vector data
    ├── searchText.js             # Text search in vectors
    ├── searchAllVectors.js       # Search all vectors
    ├── sampleMetadata.js         # Sample metadata
    ├── countBySource.js          # Count vectors by source
    ├── purgeIntranetVectors.js   # Delete intranet vectors
    ├── purgeABOVectors.js        # Delete ABO vectors
    └── verifyProductionTags.js   # Verify production tags
```

**Key Responsibilities**:
- Document ingestion and processing
- Vector search and retrieval
- Response caching
- Feedback learning
- Analytics tracking
- Web crawling
- Metadata extraction

**Most Important Files**:
- `ingestion/ingest.js:1` - Document ingestion pipeline
- `vector-ops/retrieve.js:228` - RAG retrieval (THE CORE)
- `cache/responseCache.js:1` - Response caching

#### `/src/routes` - API Route Handlers
**Purpose**: Specialized route handlers

```
routes/
└── chatStreaming.js              # Streaming chat endpoint
```

**Key Responsibilities**:
- Handle streaming responses (SSE)
- Hybrid cache/stream logic

#### `/src/storage` - Data Storage
**Purpose**: Database and storage abstractions

```
storage/
├── conversationMemory.js         # In-memory conversation storage (legacy)
├── conversationMemoryPostgres.js # PostgreSQL conversation storage (ACTIVE)
├── feedbackStorage.js            # Feedback data storage
└── googleSheetsStorage.js        # Google Sheets integration
```

**Key Responsibilities**:
- Conversation history persistence
- Feedback data storage
- Session management
- Analytics data

#### `/src/streaming` - Streaming Services
**Purpose**: Server-sent events (SSE) streaming

```
streaming/
└── chatStreamService.js          # Streaming chat service
```

**Key Responsibilities**:
- Stream LLM responses in real-time
- Handle SSE connections
- Backpressure management

#### `/src/utils` - Utility Functions
**Purpose**: Shared utility functions

```
utils/
└── performanceMetrics.js         # Performance monitoring
```

**Key Responsibilities**:
- Track request performance
- Record metrics
- Generate reports

#### `/src/server.js` - Main Application
**Purpose**: Express server and application entry point

**File**: `src/server.js:1` (1600+ lines)

**Key Responsibilities**:
- Initialize Express server
- Define all API endpoints
- Mount middleware
- Connect to databases
- Handle requests
- Error handling
- Graceful shutdown

**Major Endpoints**:
- `POST /chat` - Main chat endpoint
- `POST /chat/stream` - Streaming chat
- `POST /feedback` - Feedback submission
- `GET /api/analytics` - Analytics dashboard
- `GET /api/comment-insights` - Comment analysis
- `GET /api/popular-questions` - Popular queries
- `POST /admin/cache/clear` - Clear cache
- `GET /admin/cache/stats` - Cache statistics
- `GET /admin/conversations` - View conversations

---

### `/python` - Document Processing

Python scripts for advanced document extraction and crawling.

```
python/
├── document_processor.py         # Main document processor
├── dropbox_api_processor.py      # Dropbox API integration
├── process_all_dropbox.py        # Batch process Dropbox docs
├── crawl_caes_website.py         # CAES website crawler
├── crawl_extension_website.py    # Extension website crawler
├── process_wordpress_uploads.py  # WordPress document processor
└── requirements.txt              # Python dependencies
```

**Key Responsibilities**:
- Extract text from PDFs (better than Node.js libraries)
- Process Word documents (DOCX/DOC)
- Process PowerPoint files (PPTX/PPT)
- Process Excel spreadsheets (XLSX/XLS)
- Fetch files from Dropbox API
- Crawl websites with authentication

**Usage**:
```bash
cd python
pip install -r requirements.txt
python document_processor.py "https://dropbox.com/..."
```

---

### `/public` - Frontend

Static files served to users.

```
public/
├── index.html                    # Main chat interface
├── login.html                    # Login page
├── admin.html                    # Admin dashboard
├── styles.css                    # Styles
└── script.js                     # Frontend JavaScript
```

**Key Responsibilities**:
- Render chat UI
- Handle user authentication
- Display admin dashboard
- Make API requests
- Handle streaming responses

---

### `/docs` - Document Repository

Processed documents ready for ingestion. **Not in git** (too large).

```
docs/
├── dropbox/                      # GaCounts help documents (65 files)
├── abo-site/                     # ABO policies and procedures
├── olod-site/                    # Leadership development content
├── oit-site/                     # IT help articles
├── omc-site/                     # Marketing resources
├── brand-site/                   # Brand guidelines and logos
├── gacounts-site/                # GaCounts system pages
├── wordpress-uploads-processed/  # Processed WordPress files
├── caes-main-site/               # Main CAES website
├── extension-site/               # Extension website
├── teamdynamix/                  # TeamDynamix knowledge base
└── web/                          # Miscellaneous web content
```

**Structure**: Each document is a `.md` file with:
```markdown
---
url: https://source-url.com
title: Document Title
---

Document content here...
```

---

### `/documentation` - Project Documentation

All project documentation (you're reading this now!).

See [DEVELOPER_GUIDE.md](../DEVELOPER_GUIDE.md) for the full documentation map.

---

## Key Files Reference

### Critical Files (Must Understand)

| File | Lines | Purpose | When to Modify |
|------|-------|---------|----------------|
| `src/server.js:1` | 1600+ | Main Express server, all endpoints | Adding endpoints, changing logic |
| `src/rag/vector-ops/retrieve.js:228` | 549 | RAG retrieval pipeline | Changing search logic, reranking |
| `src/rag/ingestion/ingest.js:1` | 445 | Document ingestion | Adding document sources, changing chunking |
| `src/storage/conversationMemoryPostgres.js:1` | 600+ | Conversation storage | Database schema changes |
| `src/rag/cache/responseCache.js:1` | 400+ | Response caching | Cache behavior changes |

### Important Configuration Files

| File | Purpose |
|------|---------|
| `package.json` | Dependencies, scripts, metadata |
| `.env` | Environment variables (not in git) |
| `.gitignore` | Files to ignore in git |
| `src/prompts/systemPrompt.txt` | Chatbot behavior instructions |

### Entry Points

| Entry Point | Purpose | Command |
|-------------|---------|---------|
| `src/server.js:1` | Start web server | `npm run dev` |
| `src/rag/ingestion/ingest.js:1` | Ingest documents | `npm run ingest:dropbox` |
| `python/document_processor.py` | Process documents | `python python/document_processor.py` |
| Web crawlers | Crawl websites | `npm run crawl:olod` |

---

## Module Dependencies

### Core Dependencies

```
Express Server (server.js)
    ├── RAG Retrieval (rag/vector-ops/retrieve.js)
    │   ├── Pinecone Client (rag/utils/pineconeClient.js)
    │   ├── OpenAI Embeddings
    │   ├── Acronym Expander (rag/utils/acronymExpander.js)
    │   ├── Feedback Analyzer (rag/feedback/feedbackAnalyzer.js)
    │   └── Response Cache (rag/cache/responseCache.js)
    │
    ├── Conversation Memory (storage/conversationMemoryPostgres.js)
    │   └── PostgreSQL
    │
    ├── Question Reframer (analyzers/questionReframer.js)
    │   └── OpenAI API
    │
    ├── Authentication (middleware/sessionAuthMiddleware.js)
    │   ├── Session Manager (auth/sessionManager.js)
    │   └── PostgreSQL
    │
    └── Admin Routes (admin/routes/*.js)
        ├── Cache Routes
        ├── Conversation Routes
        └── Maintenance Routes
```

### Data Flow

```
User Question
    ↓
Express Server (server.js)
    ↓
Authentication Middleware
    ↓
Cache Check (responseCache.js)
    ├─ Cache Hit → Return cached response
    └─ Cache Miss → Continue
         ↓
Question Reframer (questionReframer.js)
    ↓
Acronym Expander (acronymExpander.js)
    ↓
Vector Search (retrieve.js)
    ↓
Pinecone Query
    ↓
Feedback Adjustment (feedbackAnalyzer.js)
    ↓
LLM Re-ranking (if needed)
    ↓
Context Building
    ↓
OpenAI Response Generation
    ↓
Response Caching
    ↓
Conversation Memory Storage
    ↓
Return Response to User
```

---

## Naming Conventions

### Files

- **camelCase**: JavaScript files (`conversationMemory.js`)
- **PascalCase**: Classes (`FeedbackAnalyzer.js`)
- **kebab-case**: Configuration files (`package.json`)
- **SCREAMING_SNAKE_CASE**: Documentation (`README.md`)

### Directories

- **lowercase**: Package directories (`src/`, `python/`)
- **kebab-case**: Multi-word directories (`rag/vector-ops/`)

### Variables

```javascript
// camelCase for variables and functions
const responseTime = 1234;
function calculateScore() {}

// PascalCase for classes
class FeedbackAnalyzer {}

// SCREAMING_SNAKE_CASE for constants
const MAX_RETRIES = 3;
const API_ENDPOINT = 'https://api.example.com';
```

### Functions

```javascript
// Verb-first naming
function getUserData() {}
function processDocument() {}
function retrieveRelevantChunks() {}

// Boolean predicates start with is/has/should
function isAuthenticated() {}
function hasPermission() {}
function shouldRerank() {}
```

---

## Import/Export Patterns

### ES Modules (Current Standard)

All JavaScript files use ES modules (`import`/`export`).

**`package.json`:**
```json
{
  "type": "module"
}
```

**Importing:**
```javascript
// Named imports
import { retrieveRelevantChunks } from './rag/vector-ops/retrieve.js';

// Default imports
import express from 'express';

// Namespace imports
import * as utils from './utils/helpers.js';

// Dynamic imports (for lazy loading)
const { getResponseCache } = await import('./rag/cache/responseCache.js');
```

**Exporting:**
```javascript
// Named exports (preferred)
export function myFunction() {}
export const MY_CONSTANT = 42;

// Default export
export default class MyClass {}

// Multiple exports
export { functionA, functionB, CONSTANT_C };
```

### File Extensions

**Always include `.js` extension in imports:**
```javascript
// ✅ Correct
import { retrieve } from './rag/vector-ops/retrieve.js';

// ❌ Wrong (will not work with ES modules)
import { retrieve } from './rag/vector-ops/retrieve';
```

---

## Design Patterns

### Singleton Pattern

Used for shared resources (database connections, caches):

```javascript
// conversationMemoryPostgres.js
let memoryInstance = null;

export function getConversationMemory() {
  if (!memoryInstance) {
    memoryInstance = new ConversationMemory();
  }
  return memoryInstance;
}
```

### Factory Pattern

Used for creating configured instances:

```javascript
// pineconeClient.js
export async function getPineconeIndex() {
  const pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
  return pinecone.index(process.env.PINECONE_INDEX_NAME);
}
```

### Middleware Pattern

Used throughout Express:

```javascript
// authMiddleware.js
export default function authMiddleware(req, res, next) {
  // Authenticate
  if (authenticated) {
    next(); // Continue
  } else {
    res.status(401).json({ error: 'Unauthorized' });
  }
}
```

---

## Code Style Guidelines

### Formatting

- **Indentation**: 2 spaces
- **Quotes**: Single quotes for strings (`'hello'`)
- **Semicolons**: Used at end of statements
- **Line length**: ~100 characters max (soft limit)

### Comments

```javascript
// Single-line comments for brief explanations

/**
 * Multi-line JSDoc comments for functions
 * @param {string} query - The user's question
 * @param {number} topK - Number of results to return
 * @returns {Promise<Object>} Retrieved chunks
 */
export async function retrieveRelevantChunks(query, topK = 8) {
  // Implementation
}
```

### Error Handling

```javascript
// Always use try-catch for async operations
try {
  const result = await someAsyncOperation();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  throw error; // Re-throw if caller needs to handle
}
```

---

## Next Steps

Now that you understand the code organization:

1. **Explore key files**: Start with `src/server.js:1` and `src/rag/vector-ops/retrieve.js:228`
2. **Read the architecture docs**: [System Overview](SYSTEM_OVERVIEW.md)
3. **Understand the RAG pipeline**: [RAG Pipeline](RAG_PIPELINE.md)
4. **Learn the API**: [API Reference](../guides/API_REFERENCE.md)
5. **Start developing**: [Development Workflow](../guides/DEVELOPMENT_WORKFLOW.md)

---

## Related Documentation

- [Developer Guide](../DEVELOPER_GUIDE.md) - Main developer documentation
- [System Overview](SYSTEM_OVERVIEW.md) - Architecture diagrams
- [RAG Pipeline](RAG_PIPELINE.md) - How RAG works
- [API Reference](../guides/API_REFERENCE.md) - API documentation

---

**Last Updated**: October 2025
