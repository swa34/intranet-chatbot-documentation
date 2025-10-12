# Feedback Learning System - PostgreSQL Implementation

## Overview

The feedback learning system uses user feedback from Google Sheets to improve search result rankings. The analyzed data is stored in PostgreSQL for fast access in production.

## Architecture

```
Google Sheets (Feedback) → Analyze → PostgreSQL (Learning Cache) → Retrieve.js (Score Adjustment)
```

1. **Feedback Collection**: Users provide feedback (helpful/not helpful) via the chatbot
2. **Storage**: Feedback stored in Google Sheets
3. **Analysis**: Script analyzes feedback and calculates source scores
4. **Cache**: Results stored in PostgreSQL for fast lookup
5. **Retrieval**: Search results are adjusted based on cached scores

## Database Schema

### Tables

#### `source_scores`
Tracks performance of each content source.

```sql
CREATE TABLE source_scores (
    id SERIAL PRIMARY KEY,
    source_key VARCHAR(500) UNIQUE NOT NULL,
    helpful INT DEFAULT 0,
    not_helpful INT DEFAULT 0,
    helpful_with_issues INT DEFAULT 0,
    total INT DEFAULT 0,
    score DECIMAL(5,2) DEFAULT 0,
    issue_types JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### `query_patterns`
Tracks which sources work well for specific question patterns.

```sql
CREATE TABLE query_patterns (
    id SERIAL PRIMARY KEY,
    pattern_key VARCHAR(500) UNIQUE NOT NULL,
    count INT DEFAULT 0,
    sources JSONB DEFAULT '[]',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### `feedback_cache_meta`
Metadata about the last sync.

```sql
CREATE TABLE feedback_cache_meta (
    id SERIAL PRIMARY KEY,
    last_updated TIMESTAMP DEFAULT NOW(),
    sources_analyzed INT DEFAULT 0,
    patterns_identified INT DEFAULT 0,
    total_feedback INT DEFAULT 0,
    llm_analyzed INT DEFAULT 0
);
```

## Environment Variables

Required in `.env` and Sevalla environment:

```env
# PostgreSQL Database
DB_HOST=your-db-host
DB_PORT=5432
DB_DATABASE=chat-feedback-system
DB_USERNAME=puffin
DB_PASSWORD=your_password
DB_URL=postgresql://user:pass@host:5432/dbname

# Google Sheets (for feedback)
GOOGLE_SHEETS_ID=your_sheet_id
GOOGLE_SHEETS_RANGE=Sheet1!A:J
GOOGLE_SERVICE_ACCOUNT_EMAIL=your_email
GOOGLE_PRIVATE_KEY=your_private_key
```

## Usage

### Sync Feedback Data (Manual)

Run this on Sevalla or any server with database access:

```bash
npm run sync-feedback
```

This will:
1. Load all feedback from Google Sheets
2. Analyze source performance
3. Calculate scores
4. Store results in PostgreSQL

### Automatic Usage in Production

The chatbot automatically uses the PostgreSQL cache when retrieving results:

```javascript
// In retrieve.js
const feedbackLearning = new FeedbackLearningPostgres();

// Adjusts scores based on feedback
const adjustedMatches = await feedbackLearning.adjustRetrievalScores(matches, query);
```

## How It Works

### Score Calculation

```javascript
score = (helpful - (notHelpful * 2)) / total
```

- Helpful feedback: +1 to score
- Not helpful feedback: -2 to score
- Normalized by total feedback count

### Score Adjustment

```javascript
adjustedScore = originalScore * (1 + (feedbackScore * 0.1))
```

- Maximum 10% boost/penalty based on feedback
- Pattern matches get additional 20% boost

### Example

```
Original Score: 0.82
Feedback Score: 0.5 (positive feedback)
Adjustment: 0.82 * (1 + 0.05) = 0.861

Pattern Match: 0.861 * 1.2 = 1.033
```

## Deployment

### Local Development

The feedback system will use fallback (no adjustments) if PostgreSQL is not accessible locally.

### Sevalla Production

1. Database credentials are auto-injected by Sevalla
2. Run `npm run sync-feedback` after deploying to populate cache
3. Set up a cron job or scheduled task to run sync periodically

### Scheduled Sync (Recommended)

Add a cron job on Sevalla to sync daily:

```bash
# Run every day at 2 AM
0 2 * * * cd /app && npm run sync-feedback
```

## Monitoring

### Check Cache Status

```javascript
// Get report
const report = await feedbackLearning.getSourceReport();

console.log('Top performers:', report.topPerformers);
console.log('Low performers:', report.lowPerformers);
```

### Query Patterns

```javascript
const patterns = await feedbackLearning.getQueryPatterns();

console.log('Successful patterns:', patterns);
```

## Troubleshooting

### Connection Issues

**Error**: `ENOTFOUND` or connection timeout

- **Local**: Normal - Sevalla's internal DNS not accessible locally
- **Production**: Check environment variables are set correctly

### No Score Adjustments

**Issue**: All `adjustedScore` equals `originalScore`

- Run `npm run sync-feedback` to populate database
- Check database has data: `SELECT COUNT(*) FROM source_scores;`

### Stale Data

**Issue**: Old feedback not reflecting in results

- Run `npm run sync-feedback` to refresh cache
- Set up automated sync schedule

## Migration from File-Based Cache

The old `FeedbackLearning.js` (file-based) is replaced by `FeedbackLearningPostgres.js` (database-based).

**Benefits**:
- ✅ Works in production (no file system dependency)
- ✅ Faster lookups (indexed queries)
- ✅ Centralized data (all servers access same cache)
- ✅ Better scaling (handles growth)

**Migration Steps**:
1. ✅ Database tables created
2. ✅ `retrieve.js` updated to use PostgreSQL version
3. ⏳ Run `npm run sync-feedback` on production
4. ⏳ Set up scheduled sync

## API Reference

### `FeedbackLearningPostgres`

#### Methods

**`initialize()`**
- Connects to PostgreSQL and Google Sheets
- Must be called before other methods

**`analyzeFeedback()`**
- Loads feedback from Google Sheets
- Analyzes and stores in PostgreSQL
- Returns: `{ sourcesAnalyzed, patternsIdentified, totalFeedback, llmAnalyzed }`

**`adjustRetrievalScores(results, query)`**
- Adjusts search result scores based on feedback
- Params: `results` (array), `query` (string)
- Returns: Sorted array with `adjustedScore` field

**`getSourceReport()`**
- Returns performance report
- Returns: `{ topPerformers, lowPerformers, allSources }`

**`getQueryPatterns()`**
- Returns successful query patterns
- Returns: Array of `{ pattern, successCount, topSources }`

**`close()`**
- Closes database connection pool

## Next Steps

1. Deploy code to Sevalla
2. Run `npm run sync-feedback` on production server
3. Verify database is populated
4. Test search result improvements
5. Set up automated daily sync
