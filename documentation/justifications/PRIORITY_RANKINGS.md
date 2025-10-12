# Document Priority Rankings System

## Overview
Priority rankings determine which documents are preferred when multiple sources contain similar information. Higher priority documents will appear first in search results when relevance scores are similar.

## Priority Scale: 1-10 (10 = Highest)

### Priority 10 - Direct Source Documents
- **Dropbox Intranet Files** (`docs/dropbox/intranet-files/`)
  - Original administrative documents
  - Policy documents
  - Training materials
  - Financial reports
  - *These are the actual source documents*

### Priority 9 - Official Help Documentation
- **Georgia Counts Help** (`docs/dropbox/`)
  - Official GaCounts documentation PDFs
  - Step-by-step guides

- **TeamDynamix KB** (`docs/teamdynamix/`)
  - Official HR help articles
  - Financial system guides
  - Benefits documentation
  - Payroll procedures

### Priority 8 - Training & Internal Documentation
- **ETS Training Documents** (`docs/ets/`)
  - Training manuals
  - Session guides
  - Conference materials

- **Extension Calendar Docs** (`docs/intranet/`)
  - Calendar guides
  - Manually created documentation

### Priority 7 - WordPress Crawled Sites
- **ABO Website** (`docs/abo-site/`)
  - Administrative policies
  - Business procedures

- **OIT Website** (`docs/oit-site/`)
  - IT help articles
  - Software guides

- **Brand Guidelines** (`docs/brand-site/`)
  - Brand standards
  - Logo usage

- **OLOD Website** (`docs/olod-site/`)
  - Leadership development
  - Training programs

- **OMC Website** (`docs/omc-site/`)
  - Marketing resources
  - Communication guidelines

### Priority 6 - Web Applications
- **GaCounts Web App** (`docs/gacounts-site/`)
  - Application interface documentation
  - Authenticated crawl content

### Priority 5 - General Web Content
- **General Web Crawls** (`docs/web/`)
  - Other web sources
  - Miscellaneous content

## How Priority Works

1. **During Search**: When the RAG system retrieves documents, it considers both:
   - **Relevance Score**: How well the document matches the query
   - **Priority**: The importance/authority of the source

2. **Sorting Logic**:
   ```javascript
   // When relevance scores are similar (within 0.02)
   if (Math.abs(docA.score - docB.score) < 0.02) {
     // Use priority to determine order
     return docB.priority - docA.priority;
   }
   // Otherwise, use relevance score
   return docB.score - docA.score;
   ```

3. **Date-Based Prioritization**:
   - Within the same priority level, newer documents (based on filename dates) are preferred
   - Example: "Foundations 2024 Schedule" would be preferred over "Foundations 2023 Schedule"

## Viewing Priority in Metadata

Each document in Pinecone has metadata including:
```json
{
  "sourceType": "teamdynamix_kb",
  "category": "hr_financial_help",
  "priority": 9,
  "url": "https://uga.teamdynamix.com/...",
  "ingestionDate": "2025-09-24",
  ...
}
```

## Why This Matters

- **Authoritative Sources First**: Official documents (Priority 10) are preferred over summaries
- **Help Docs Over Crawls**: TeamDynamix and GaCounts help (Priority 9) are preferred over general web crawls
- **Avoid Duplicates**: When WordPress pages just embed documents, the original document (Priority 10) wins
- **User Trust**: Users get the most authoritative source for their questions

## Checking Document Priority

To see what priority a document has:
1. Check which folder it's in under `docs/`
2. Reference this guide to see the priority level
3. Or check the metadata in Pinecone directly

## Future Adjustments

Priority levels can be adjusted in:
- `src/rag/ingest-enhanced.js` - for new ingestions
- Individual document metadata can be updated in Pinecone if needed