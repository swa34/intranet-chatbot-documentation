# WordPress Document Mapping System

## Overview

This system cross-references WordPress intranet pages with existing documents (Dropbox PDFs, help docs, etc.) to create a bidirectional knowledge map.

## The Problem We Solved

- **Before**: WordPress pages link to Dropbox documents, but we only had the documents ingested
- **Issue**: Users would find documents but not know which intranet page they came from
- **Solution**: Map WordPress pages to documents and vice versa during crawl

## How It Works

### 1. Document Mapper (`python/document_mapper.py`)

**Builds an index** of all existing documents:
- 923 documents indexed from `/docs/dropbox`, `/docs/abo-site`, etc.
- Creates 3 indexes:
  - **Filename index**: Matches by file name
  - **Title index**: Matches by document title
  - **URL index**: Matches by Dropbox/Google Drive URLs

**Matching Algorithm**:
1. **Exact URL match** (100% confidence): Dropbox link → local file
2. **Exact filename match** (95% confidence): PDF name → .md file
3. **Partial filename** (85% confidence): Fuzzy match on names
4. **Title match** (80% confidence): Link text → document title
5. **Fuzzy title** (70-80% confidence): Similar titles

### 2. Enhanced WordPress Crawler (`python/crawl_wordpress_intranet.py`)

**New Features**:
- Extracts document links from each page BEFORE cleaning HTML
- Matches links against existing documents using Document Mapper
- Generates relationship metadata for each page
- Saves metadata in markdown frontmatter

**Metadata Captured**:
```json
{
  "page_type": "navigation" | "content",
  "linked_documents": [...],
  "linked_document_count": 5,
  "high_confidence_matches": [...],
  "is_document_portal": true/false
}
```

**Embedded in Markdown**:
```markdown
**Source:** https://intranet...
**Page Type:** navigation
**Linked Documents:** 5

## Related Documents
- **Extension Calendar Guide**
  - External URL: https://dropbox.com/...
  - Local File: `../docs/dropbox/extension-calendar-guide.md`
  - Match Confidence: 95%
```

### 3. Crawl Summary (`docs/intranet/crawl_summary.json`)

Saves complete relationship data:
- Which pages link to which documents
- Confidence scores for each match
- Page types (navigation vs content)
- Document portal pages (3+ document links)

## Usage

### Crawl WordPress with Relationship Mapping

```bash
cd python
python crawl_wordpress_intranet.py
```

**Outputs**:
- Markdown files in `docs/intranet/` with embedded relationships
- `crawl_summary.json` with complete relationship map

### Check Document Relationships

Look in the crawl summary:
```json
{
  "url": "https://intranet...",
  "relationships": {
    "linked_documents": [
      {
        "document_url": "https://dropbox.com/...",
        "local_file": "../docs/dropbox/guide.md",
        "confidence": 0.95
      }
    ]
  }
}
```

## Next Steps

### Phase 1: Full WordPress Crawl ✓ (Current)
- Crawl all intranet pages (increase MAX_PAGES to 500+)
- Capture all document relationships
- Build complete site map

### Phase 2: Enhanced Ingestion (Next)
Update `src/rag/ingest.js` to:
1. Parse relationship metadata from markdown files
2. Add "parent_page_url" to document vectors
3. Add "child_documents" list to page vectors
4. Create bidirectional links in Pinecone metadata

### Phase 3: Smart Bot Responses (Future)
Update chatbot to:
1. When finding a document, also show its parent WordPress page
2. When finding a WordPress page, also show linked documents
3. Format: "Found in **[Page Title](url)** - see also: [Doc 1], [Doc 2]"

## Benefits

✅ **Bidirectional Navigation**: Find documents from pages, pages from documents
✅ **Context Preservation**: Know where documents live in the intranet structure
✅ **No Duplication**: Don't re-ingest documents, just link to existing ones
✅ **Clean Re-ingestion**: Can purge and re-ingest with full relationships intact
✅ **Confidence Scoring**: Only show high-confidence matches to users

## Configuration

Edit `python/crawl_wordpress_intranet.py`:
```python
MAX_PAGES = 500  # Increase for full crawl
MAX_DEPTH = 4    # How deep to follow links
CRAWL_DELAY = 0.5  # Rate limiting (seconds)
```

## Troubleshooting

**Low match rates?**
- Check confidence thresholds in `document_mapper.py`
- Add more matching patterns for your document naming conventions
- Verify Dropbox URLs are being extracted from existing docs

**Missing documents?**
- Ensure document directories are listed in `DOC_DIRS` in `document_mapper.py`
- Run `python document_mapper.py` to test matching

**Slow crawling?**
- Adjust `CRAWL_DELAY` (but be respectful to the server)
- Run in background: `python crawl_wordpress_intranet.py &`

## Files Modified

- ✅ `python/document_mapper.py` - NEW: Document matching system
- ✅ `python/crawl_wordpress_intranet.py` - ENHANCED: Added relationship mapping
- 📝 `src/rag/ingest.js` - TODO: Parse and use relationships
- 📝 `src/server.js` - TODO: Show bidirectional links in responses