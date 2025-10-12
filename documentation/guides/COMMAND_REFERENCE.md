# Command Reference Guide

Complete reference for all CAES Chatbot commands, organized by category for quick access.

---

## Quick Start Commands

### Most Common Operations

```bash
# Start the chatbot server
npm start

# Update documentation (run after ANY content update)
npm run docs:publish

# Full update workflow for a website (example: OIT)
npm run crawl:oit && npm run ingest:oit && npm run docs:publish
```

---

## Documentation Management

### Publishing Documentation

```bash
# Generate, copy, commit, and push to GitHub Pages
npm run docs:publish

# Generate and copy to GitHub Pages repo (no commit)
npm run docs:update

# Generate documentation locally only
npm run docs:generate
```

**What they do:**
- `docs:publish` - Full workflow: generates docs, copies to GitHub Pages repo, commits and pushes
- `docs:update` - Generates docs and copies to GitHub Pages repo (no git operations)
- `docs:generate` - Only runs Python script to generate `GITPAGES/index.html`

**Output Location:**
- Local: `GITPAGES/index.html`
- Live: Pushed to `caes-chatbot-docs` repository

---

## Website Crawling & Ingestion

### CAES Internal Sites

#### CAES Intranet
```bash
npm run crawl:intranet
npm run ingest:intranet
# Combined workflow:
npm run crawl:intranet && npm run ingest:intranet && npm run docs:publish
```

#### Administrative Business Office (ABO)
```bash
npm run crawl:abo
npm run ingest:abo
# Combined workflow:
npm run crawl:abo && npm run ingest:abo && npm run docs:publish
```

#### Office of Information Technology (OIT)
```bash
npm run crawl:oit
npm run ingest:oit
# Combined workflow:
npm run crawl:oit && npm run ingest:oit && npm run docs:publish
```

#### Office of Learning & Organizational Development (OLOD)
```bash
npm run crawl:olod
npm run ingest:olod
# Combined workflow:
npm run crawl:olod && npm run ingest:olod && npm run docs:publish
```

#### Office of Marketing & Communications (OMC)
```bash
npm run crawl:omc
npm run ingest:omc
# Combined workflow:
npm run crawl:omc && npm run ingest:omc && npm run docs:publish
```

### CAES Public Sites

#### CAES Brand Guidelines
```bash
npm run crawl:brand
npm run ingest:brand
# Combined workflow:
npm run crawl:brand && npm run ingest:brand && npm run docs:publish
```

#### CAES Main Website
```bash
npm run crawl:caes-main
npm run ingest:caes-main
```

#### UGA Extension Website
```bash
npm run crawl:extension
npm run ingest:extension
```

---

## Dropbox Document Management

### Full Dropbox Update
```bash
# Fetch and process all Dropbox files
npm run fetch:dropbox
npm run process:all-georgia-counts
npm run ingest:dropbox
npm run ingest:dropbox-intranet
npm run docs:publish
```

### Dropbox Operations

```bash
# Fetch all Dropbox files
npm run fetch:dropbox

# Process all Georgia Counts documents
npm run process:all-georgia-counts

# Ingest main Dropbox folder
npm run ingest:dropbox

# Ingest Dropbox intranet files subfolder
npm run ingest:dropbox-intranet
```

### Dropbox Intranet Files Only
```bash
cd python
python dropbox_api_processor.py
cd ..
npm run ingest:dropbox-intranet && npm run docs:publish
```

---

## Other Content Sources

### ETS Resources
```bash
npm run ingest:ets
```

### TeamDynamix Knowledge Base
```bash
cd python
python crawl_teamdynamix.py
cd ..
npm run ingest:teamdynamix && npm run docs:publish
```

### WordPress Uploads
```bash
npm run process:wordpress
npm run ingest:wordpress
```

### GaCounts Application Pages
```bash
npm run ingest:gacounts
```

---

## Server & Development

### Starting the Server

```bash
# Start production server
npm start

# Start development server
npm run dev
```

### Cloudflare Tunnel

```bash
# Start tunnel in separate terminal
npm run tunnel

# Start both server and tunnel together
npm run dev:all
```

---

## Debugging & Verification

### Pinecone Vector Checks

```bash
# Check Dropbox vectors in Pinecone
node src/rag/vector-ops/checkDropboxVectors.js

# Check specific Dropbox document
node src/rag/vector-ops/checkSpecificDropboxDoc.js

# Search all vectors
node src/rag/vector-ops/searchAllVectors.js
```

### Usage & Statistics

```bash
# Check API usage
npm run check-usage

# Debug storage information
npm run debug-storage
```

### Feedback & Learning

```bash
# Sync feedback learning data
npm run sync-feedback
```

---

## Python Scripts

All Python scripts are located in the `python/` folder.

### Dropbox Processing Scripts

```bash
cd python

# Process Dropbox API content
python dropbox_api_processor.py

# Process all Dropbox files
python process_all_dropbox.py

# Process intranet-specific Dropbox files
python process_all_intranet_dropbox.py

# Fetch Dropbox links
python fetch_dropbox_links.py
```

### Website Crawling Scripts

```bash
cd python

# Crawl CAES main website
python crawl_caes_website.py

# Crawl UGA Extension website
python crawl_extension_website.py

# Crawl WordPress intranet
python crawl_wordpress_intranet.py

# Crawl TeamDynamix knowledge base
python crawl_teamdynamix.py
```

### Document Processing Scripts

```bash
cd python

# General document processor
python document_processor.py

# Process ETS documents
python process_ets_documents.py

# Process WordPress uploads
python process_wordpress_uploads.py
```

---

## Troubleshooting

### Documentation Issues

```bash
# Check if docs generated correctly
npm run docs:generate
# Then open: GITPAGES/index.html in browser
```

### Dropbox Verification

```bash
# Verify Dropbox ingestion
node src/rag/vector-ops/checkDropboxVectors.js
```

### Server Issues

```bash
# Server not starting? Check if port 3000 is in use
# Kill any existing process on port 3000
npm start
```

### Setup & Dependencies

```bash
# Setup Python dependencies
npm run setup:python

# Verify everything works
npm run docs:generate      # Should generate docs
npm start                   # Should start server on port 3000
```

---

## Common Workflows

### After Crawling a Website

```bash
# Standard workflow (example with OIT)
npm run crawl:oit && npm run ingest:oit && npm run docs:publish
```

### Update All Dropbox Content

```bash
npm run fetch:dropbox && npm run process:all-georgia-counts && npm run ingest:dropbox && npm run ingest:dropbox-intranet && npm run docs:publish
```

### Full Documentation Refresh

```bash
# Run after updating any crawled content
npm run docs:publish
```

---

## Cache Management

### Redis Cache Operations

The chatbot uses Redis for caching Pinecone queries and GPT responses. Cache operations are handled automatically but can be monitored through:

```bash
# Check cache statistics
npm run check-usage

# Debug storage and cache info
npm run debug-storage
```

---

## Testing Commands

### Manual Testing

```bash
# Start the server and test in browser
npm start
# Open http://localhost:3000

# Test with Cloudflare tunnel
npm run dev:all
```

### Vector Search Testing

```bash
# Search vectors to verify ingestion
node src/rag/vector-ops/searchAllVectors.js

# Check specific documents
node src/rag/vector-ops/checkSpecificDropboxDoc.js
```

---

## File Locations Reference

- **Generated documentation:** `GITPAGES/index.html`
- **Live documentation:** Published to `caes-chatbot-docs` GitHub repository
- **Crawled website data:** `docs/` folder
- **Dropbox documents:** `docs/dropbox/` folder
- **Scripts:** `scripts/` and `python/` folders
- **Vector operations:** `src/rag/vector-ops/` folder

---

## Pro Tips

1. **Always update documentation after content changes:**
   ```bash
   npm run docs:publish
   ```

2. **Chain commands efficiently:**
   ```bash
   npm run crawl:oit && npm run ingest:oit && npm run docs:publish
   ```

3. **Verify vector ingestion:**
   ```bash
   node src/rag/vector-ops/checkDropboxVectors.js
   ```

4. **Use combined workflows for speed:**
   ```bash
   npm run dev:all  # Starts server and tunnel together
   ```

5. **Remember the two-step process:**
   - Step 1: Crawl or fetch content
   - Step 2: Ingest into Pinecone
   - Step 3: Update documentation

---

## Command Categories Summary

| Category | Commands |
|----------|----------|
| **Documentation** | `docs:publish`, `docs:update`, `docs:generate` |
| **Web Crawling** | `crawl:intranet`, `crawl:abo`, `crawl:oit`, `crawl:olod`, `crawl:omc`, `crawl:brand`, `crawl:caes-main`, `crawl:extension` |
| **Ingestion** | `ingest:*` (matches crawl commands), `ingest:dropbox`, `ingest:dropbox-intranet`, `ingest:ets`, `ingest:teamdynamix`, `ingest:wordpress`, `ingest:gacounts` |
| **Dropbox** | `fetch:dropbox`, `process:all-georgia-counts`, `ingest:dropbox`, `ingest:dropbox-intranet` |
| **Server** | `start`, `dev`, `tunnel`, `dev:all` |
| **Debugging** | `check-usage`, `debug-storage`, `sync-feedback` |
| **Setup** | `setup:python` |

---

## Need Help?

1. Check server status: `npm start`
2. Verify documentation: `npm run docs:generate`
3. Check Pinecone vectors: `node src/rag/vector-ops/searchAllVectors.js`
4. Review logs in the console output
5. Ensure Python dependencies are installed: `npm run setup:python`
