# Common Commands - Quick Copy/Paste

## 🎯 Most Used Commands

### Update Documentation (Use This Most!)
```bash
npm run docs:publish
```

### Start the Chatbot
```bash
npm start
```

---

## 🕷️ Crawl & Update a Website

### OIT (Office of IT)
```bash
npm run crawl:oit && npm run ingest:oit && npm run docs:publish
```

### ABO (Administrative Business Office)
```bash
npm run crawl:abo && npm run ingest:abo && npm run docs:publish
```

### OLOD (Learning & Org Development)
```bash
npm run crawl:olod && npm run ingest:olod && npm run docs:publish
```

### OMC (Marketing & Communications)
```bash
npm run crawl:omc && npm run ingest:omc && npm run docs:publish
```

### Brand Guidelines
```bash
npm run crawl:brand && npm run ingest:brand && npm run docs:publish
```

### CAES Intranet
```bash
npm run crawl:intranet && npm run ingest:intranet && npm run docs:publish
```

---

## 📦 Update Dropbox Content

### All Dropbox Files
```bash
npm run fetch:dropbox && npm run process:all-georgia-counts && npm run ingest:dropbox && npm run docs:publish
```

### Just Intranet Files
```bash
cd python
python dropbox_api_processor.py
cd ..
npm run ingest:dropbox-intranet && npm run docs:publish
```

---

## 🔧 TeamDynamix Knowledge Base

```bash
cd python
python crawl_teamdynamix.py
cd ..
npm run ingest:teamdynamix && npm run docs:publish
```

---

## ✅ After ANY Update

**Always run this to update the documentation site:**
```bash
npm run docs:publish
```

---

## 💡 Simple Workflow

1. **Crawl or update content** (pick one from above)
2. **Run `npm run docs:publish`**
3. **Done!** ✅

---

## 🆘 Troubleshooting

### Check if docs generated correctly
```bash
npm run docs:generate
# Then open: GITPAGES/index.html in browser
```

### Check Dropbox ingestion
```bash
node src/rag/vector-ops/checkDropboxVectors.js
```

### Server not starting?
```bash
# Check if port 3000 is in use
# Kill any existing process on port 3000
npm start
```

---

## 📂 File Locations

- **Generated docs:** `GITPAGES/index.html`
- **Live docs:** Push to `caes-chatbot-docs` repo
- **Crawled data:** `docs/` folder
- **Scripts:** `scripts/` and `python/` folders
