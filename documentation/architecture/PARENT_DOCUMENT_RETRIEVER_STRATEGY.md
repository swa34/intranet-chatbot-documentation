# Parent Document Retriever Strategy

## Overview

This document outlines the strategy for implementing a **Parent Document Retriever** model. This approach enhances the RAG pipeline by searching over small, specific "child" chunks of text while providing larger, more context-rich "parent" chunks to the LLM for answer generation.

This strategy was proposed as a potential enhancement to be implemented concurrently with the Hybrid Search feature, as both require a full re-ingestion of the source data.

## Core Benefit

This strategy solves a common RAG problem:

1.  **Precise Search:** Small, granular chunks are better for vector search as they provide a more specific match to the user's query.
2.  **Rich Context:** Large chunks of text give the LLM the surrounding context it needs to generate high-quality, comprehensive answers, preventing responses that are "cut off" or lack background information.

By combining these, we get the best of both worlds.

## Implementation Plan

The implementation involves modifying both the ingestion and retrieval stages of the RAG pipeline.

### 1. Update Chunking Logic (`src/rag/ingestion/chunk.js`)

The existing `chunkText` function will be supplemented with a new two-layer chunking process:

1.  **Create Parent Chunks:** The entire document text is first split into large, overlapping chunks (e.g., 1500-2000 characters). These splits should ideally occur along natural boundaries like paragraphs (\n\n).
2.  **Create Child Chunks:** Each parent chunk is then split again into smaller, overlapping child chunks (e.g., 300-400 characters). These child chunks are what will be vectorized.

### 2. Update Ingestion Script (`src/rag/ingestion/ingest.js`)

The ingestion script will be modified to handle the parent-child relationship.

*   For each document, the new two-layer chunking logic is applied.
*   A loop will iterate through each **child chunk**.
*   For each child chunk, we will generate its dense (OpenAI) and sparse (keyword) vectors.
*   When creating the vector object to `upsert` to Pinecone, the metadata will be critically important. It will store the full, original text of the **parent chunk**.

**Example Vector Structure:**

```json
{
  "id": "doc1-child3",
  "values": [0.1, 0.2, ...], // Dense vector from the child chunk
  "sparseValues": { ... },   // Sparse vector from the child chunk
  "metadata": {
    "parentText": "The full text of the large parent chunk...",
    "childText": "The specific text of this small child chunk...",
    "parentChunkId": "doc1-parent1",
    "source": "source_document.pdf"
  }
}
```

### 3. Update Retrieval Script (`src/rag/vector-ops/retrieve.js`)

The retrieval logic will be updated to leverage this new structure.

1.  **Search (No Change):** The initial hybrid search query to Pinecone remains the same. It searches for the most relevant **child chunks**.
2.  **Post-Retrieval Step (New):** After the re-ranking step, a new processing step is added:
    *   Take the final, ordered list of child chunk results.
    *   Instead of using their own text, extract the `parentText` from the metadata of each result.
    *   De-duplicate this list to ensure the same parent chunk isn't sent to the LLM multiple times.
3.  **Generation (Input Change):** The final, de-duplicated list of large parent chunks is what gets passed to the LLM for generating the answer.

### Interaction with Other Systems

This strategy complements the existing advanced features:

*   **Hybrid Search:** Provides the initial, high-quality list of child chunks.
*   **LLM Re-ranking:** Re-orders the child chunks for maximum relevance before the switch to their parents.

The final, sequential data flow in the pipeline will be:

`Search (Child Chunks) -> Re-Rank (Child Chunks) -> Switch to Parents -> Generate Answer (from Parents)`

## Next Steps (When You Return)

1.  Create a new branch from `feature-hybrid-search` (e.g., `feature-parent-retriever`).
2.  Modify `src/rag/ingestion/chunk.js` to implement the two-layer chunking logic.
3.  Update `src/rag/ingestion/ingest.js` to process documents using the new chunking and store the `parentText` in the metadata.
4.  Update `src/rag/vector-ops/retrieve.js` to add the post-retrieval step for switching from child results to parent context.
5.  Perform a full data re-ingestion into a test namespace to validate the end-to-end process.

## Implementation Status (As of October 14, 2025)

All code modifications for the Parent Document Retriever strategy have been completed.

*   **[COMPLETED]** A new Pinecone namespace (`HYBRID_SEARCH_PARENT`) has been configured in the local `.env` file.
*   **[COMPLETED]** The chunking logic in `src/rag/ingestion/chunk.js` has been updated to create parent-child chunk pairs.
*   **[COMPLETED]** The ingestion script `src/rag/ingestion/ingest.js` has been modified to handle the new chunk structure and store `parentText` in the vector metadata.
*   **[COMPLETED]** The retrieval script `src/rag/vector-ops/retrieve.js` has been updated to replace child chunks with their parent chunks before sending them to the language model.
*   **[COMPLETED]** Fixed namespace bug in `ingest.js` line 438 to use `.namespace(NAMESPACE).upsert()` syntax.
*   **[COMPLETED]** Removed invalid `alpha` parameter from retrieve.js (Pinecone SDK v6 auto-balances).
*   **[COMPLETED]** Vector structure verified: All vectors have dense values (3072D), sparse values (16-26 indices), and parent-child metadata.

### Critical Issue: Pinecone Index Metric Incompatibility

**Problem:** The existing Pinecone index `uga-intranet-index` was created with `metric: 'cosine'`, which **does not support hybrid search** (sparse vectors). Pinecone requires `metric: 'dotproduct'` for hybrid search.

**Resolution Options:**

#### Option 1: Create New Index (Recommended for Testing)
Create a brand new index with a different name for testing hybrid search:

1. Update `.env`:
   ```
   PINECONE_INDEX_NAME=uga-intranet-hybrid  # New index name
   PINECONE_NAMESPACE=HYBRID_SEARCH_PARENT
   ```

2. Run ingestion with `--recreate-index` flag:
   ```bash
   npm run ingest:brand -- --recreate-index
   ```
   This will create a new index with `dotproduct` metric and ingest test data.

3. Test retrieval, then decide whether to migrate production data.

#### Option 2: Recreate Existing Index (⚠️ Deletes All Data)
**WARNING:** This will delete all 27,495 vectors in your production index!

1. Backup your data first
2. Run: `npm run ingest:brand -- --recreate-index`
3. Re-ingest all sources using `restore-all-data.bat`

#### Option 3: Disable Hybrid Search (Keep Parent-Child Only)
If you want to keep your existing index and data:

1. Remove sparse vector creation from `ingest.js` (lines 415-426)
2. Remove sparse vector from `retrieve.js` (line 440)
3. Parent-child retrieval will still work with semantic search only

### Test Results

**Tested successfully:**
- ✅ Parent-child chunking (1,200 char parents → 400 char children)
- ✅ Sparse vector creation (16-26 keyword indices per vector)
- ✅ Dense vector embedding (3,072 dimensions)
- ✅ Metadata structure (hasParentText, text, parentText)
- ✅ Namespace isolation (HYBRID_SEARCH_PARENT)

**Blocked by index metric:**
- ❌ Hybrid search queries (requires dotproduct index)

### Recommended Next Steps

1. **For production deployment:** Create new index `uga-intranet-hybrid` with Option 1
2. **Test with brand data:** Verify hybrid search + parent retrieval works end-to-end
3. **Gradual migration:** Ingest all sources to new index, test thoroughly
4. **Cutover:** Update `.env` to point to new index when ready
5. **Cleanup:** Delete old cosine index after confirming new index works