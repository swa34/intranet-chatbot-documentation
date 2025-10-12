# File System Cleanup Plan

## Current Structure Overview

### Two Main Applications:
1. **Chatbot Application** (Node.js/Express)
   - Location: `src/` directory
   - Main file: `src/server.js`
   - Purpose: Serves the chatbot interface and handles RAG queries

2. **Ingestion/Processing Application** (Python)
   - Location: `python/` directory
   - Purpose: Processes documents from various sources and ingests into Pinecone

## Files to Clean Up

### 1. Test Files in Root (Move to `tests/` directory)
- `test-abo.js`
- `test-crawl.js`
- `testExtractUrl.js`
- `test_date_priority.js`

### 2. Batch Scripts (Move to `scripts/` directory)
- `process_all_georgia_counts.bat`
- `process_dropbox_api.bat`
- `process_intranet_dropbox.bat`

### 3. Documentation Files (Keep in root but organize)
**Keep in root:**
- `README.md` - Main project documentation
- `INGESTED_CONTENT.md` - Tracking document
- `.env.example` - Environment template

**Move to `docs/documentation/`:**
- `AUTH_IMPLEMENTATION.md`
- `GOOGLE_SHEETS_SETUP.md`
- `GOOGLE_SHEETS_USER_TRACKING.md`
- `INTRANET_SETUP.md`
- `SEVALLA_GOOGLE_SHEETS_TROUBLESHOOTING.md`

### 4. Data Files (Move to `config/` or `data/`)
- `debtapp-471513-3606a8da1809.json` → `config/` (Google Sheets credentials)
- `RESOURCE_LINKS.json` → `data/`
- `Georgia_Counts_Dropbox_Links.txt` → `data/`
- `sitemap (brand).xml` → `data/`
- `sitemap.xml` → `data/`

### 5. Directories to Keep As-Is
- `src/` - Chatbot application
- `python/` - Ingestion scripts
- `docs/` - Processed documents
- `public/` - Static files for web interface
- `config/` - Configuration files
- `node_modules/` - Dependencies

### 6. Special Directory
- `uga-intranet-Bot/` - Check if this is needed or can be removed

## Proposed New Structure

```
intranetChatbot/
├── src/                    # Chatbot application
│   ├── rag/               # RAG implementation
│   ├── prompts/           # Chat prompts
│   └── server.js          # Main server
├── python/                 # Ingestion scripts
│   ├── dropbox_api_processor.py
│   └── [other processors]
├── docs/                   # Processed documents
│   ├── dropbox/
│   ├── abo-site/
│   └── [other sources]
├── tests/                  # Test files
│   ├── test-abo.js
│   └── [other tests]
├── scripts/                # Utility scripts
│   ├── process_dropbox_api.bat
│   └── [other scripts]
├── config/                 # Configuration
│   └── debtapp-*.json
├── data/                   # Static data files
│   ├── RESOURCE_LINKS.json
│   └── sitemaps/
├── documentation/          # Project documentation
│   ├── setup/
│   └── guides/
├── public/                 # Web interface
├── .env                    # Environment variables
├── .env.example           # Environment template
├── README.md              # Main documentation
├── INGESTED_CONTENT.md    # Ingestion tracking
├── package.json           # Node dependencies
└── .gitignore            # Git ignore rules
```

## Benefits of Cleanup
1. Clear separation between chatbot and ingestion apps
2. Easier navigation for new developers
3. Better organization of test files
4. Documentation grouped logically
5. Scripts and data files in appropriate locations