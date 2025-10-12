# Quick Reference Guide - All Commands

## 📚 Documentation Updates

### Update Documentation (Most Common)
```bash
npm run docs:publish
```
**What it does:** Generates docs → Copies to GitHub Pages repo → Commits & pushes

### Just Generate Documentation (No Publish)
```bash
npm run docs:update
```
**What it does:** Generates docs → Copies to GitHub Pages repo (no commit)

### Just Generate Locally (No Copy)
```bash
npm run docs:generate
```
**What it does:** Only runs the Python script to generate `GITPAGES/index.html`

---

## 🕷️ Crawling Websites

### CAES Intranet
```bash
npm run crawl:intranet
npm run ingest:intranet
```

### Administrative Business Office (ABO)
```bash
npm run crawl:abo
npm run ingest:abo
```

### Office of Information Technology (OIT)
```bash
npm run crawl:oit
npm run ingest:oit
```

### Office of Learning & Organizational Development (OLOD)
```bash
npm run crawl:olod
npm run ingest:olod
```

### Office of Marketing & Communications (OMC)
```bash
npm run crawl:omc
npm run ingest:omc
```

### CAES Brand Guidelines
```bash
npm run crawl:brand
npm run ingest:brand
```

### CAES Main Website
```bash
npm run crawl:caes-main
npm run ingest:caes-main
```

### UGA Extension Website
```bash
npm run crawl:extension
npm run ingest:extension
```

---

## 📦 Dropbox Documents

### Fetch & Process All Dropbox Files
```bash
npm run fetch:dropbox
npm run process:all-georgia-counts
```

### Process Dropbox Intranet Files
```bash
cd python
python dropbox_api_processor.py
```

### Ingest Dropbox into Pinecone
```bash
npm run ingest:dropbox              # Main Dropbox folder
npm run ingest:dropbox-intranet     # Intranet files subfolder
```

---

## 📄 Other Content Sources

### ETS Resources
```bash
npm run ingest:ets
```

### TeamDynamix Knowledge Base
```bash
cd python
python crawl_teamdynamix.py
npm run ingest:teamdynamix
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

## 🚀 Server & Development

### Start the Chatbot Server
```bash
npm start
# or
npm run dev
```

### Start with Cloudflare Tunnel
```bash
npm run tunnel              # In separate terminal
npm run dev:all             # Runs both server and tunnel
```

---

## 🔍 Debugging & Utilities

### Check Pinecone Vectors
```bash
# Check Dropbox vectors
node src/rag/vector-ops/checkDropboxVectors.js

# Check specific document
node src/rag/vector-ops/checkSpecificDropboxDoc.js

# Search all vectors
node src/rag/vector-ops/searchAllVectors.js
```

### Check Usage & Stats
```bash
npm run check-usage
npm run debug-storage
```

### Sync Feedback Learning
```bash
npm run sync-feedback
```

---

## 🎯 Common Workflows

### After Crawling a New Site
```bash
# Example: After crawling OIT
npm run crawl:oit
npm run ingest:oit
npm run docs:publish
```

### Update All Dropbox Content
```bash
npm run fetch:dropbox
npm run process:all-georgia-counts
npm run ingest:dropbox
npm run ingest:dropbox-intranet
npm run docs:publish
```

### Full Documentation Refresh
```bash
# After updating any crawled content
npm run docs:publish
```

---

## 📋 Python Scripts (Manual)

Located in `python/` folder:

```bash
cd python

# Dropbox Processing
python dropbox_api_processor.py
python process_all_dropbox.py
python process_all_intranet_dropbox.py
python fetch_dropbox_links.py

# TeamDynamix
python crawl_teamdynamix.py

# Website Crawling
python crawl_caes_website.py
python crawl_extension_website.py
python crawl_wordpress_intranet.py

# Document Processing
python document_processor.py
python process_ets_documents.py
python process_wordpress_uploads.py
```

---

## 🆘 Help

### Setup Python Dependencies
```bash
npm run setup:python
```

### Check if Everything Works
```bash
npm run docs:generate      # Should generate docs
npm start                   # Should start server on port 3000
```

---

## 💡 Pro Tips

1. **Always update docs after crawling:**
   ```bash
   npm run crawl:oit && npm run ingest:oit && npm run docs:publish
   ```

2. **Chain commands with `&&`:**
   ```bash
   npm run fetch:dropbox && npm run process:all-georgia-counts && npm run ingest:dropbox
   ```

3. **Check what's in Pinecone:**
   ```bash
   node src/rag/vector-ops/checkDropboxVectors.js
   ```

4. **Documentation is always at:**
   - Local: `GITPAGES/index.html`
   - Live: GitHub Pages repo after `npm run docs:publish`
