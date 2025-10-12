# PostgreSQL Feedback System Analysis & Improvement Plan

## Current Database Schema

### Tables You Have:

1. **`conversations`** - Stores all chat interactions
   - `session_id`, `question`, `answer`, `timestamp`
   - `feedback_rating` (-1, 0, 1)
   - `feedback_comment`
   - `sources` (JSONB) - what sources were used
   - `response_time_ms`

2. **`conversation_sessions`** - Session metadata
   - `session_id`, `created_at`, `last_accessed_at`
   - `total_turns`
   - `user_agent`, `is_active`

3. **`pending_clarifications`** - Clarification requests
   - `session_id`, `original_question`, `main_topic`
   - `resolved`, `resolved_at`

4. **Feedback Learning Tables** (from feedbackLearningPostgres.js):
   - `source_scores` - Tracks which sources are helpful/not helpful
   - `query_patterns` - Tracks successful query patterns

## Current Data Flow

```
User Question
    ↓
[Retrieve from Pinecone] → sources returned
    ↓
[Generate Answer with LLM]
    ↓
[Save to conversations table] ← sources, response_time saved
    ↓
User gives feedback (optional)
    ↓
[Update feedback_rating in conversations]
    ↓
[Sync to Google Sheets] ← current feedback destination
    ↓
[FeedbackLearningPostgres reads from Sheets]
    ↓
[Updates source_scores & query_patterns in Postgres]
```

## 🚨 Current Issues

### 1. **Duplicate Storage**
- Feedback goes to **Google Sheets** first
- Then gets **re-synced back to Postgres**
- **WHY?** Historical reasons - Google Sheets was easier to set up initially

### 2. **Underutilized Conversation Data**
You're storing valuable data in `conversations` table but NOT using it:
- ❌ Not analyzing which questions get negative feedback
- ❌ Not analyzing which sources appear in successful vs failed conversations
- ❌ Not tracking user behavior patterns
- ❌ Not identifying common failure scenarios

### 3. **Google Sheets Bottleneck**
- Sheets API has rate limits
- Slow to query
- Hard to do complex analytics
- You already have a fast Postgres DB!

## ✅ Recommended Improvements

### **Phase 1: Direct Postgres Feedback (Immediate)**

Stop using Google Sheets as primary storage. Instead:

```sql
-- New feedback table (merge with conversations or keep separate)
CREATE TABLE feedback (
  id SERIAL PRIMARY KEY,
  conversation_id INTEGER REFERENCES conversations(id),
  rating VARCHAR(20) NOT NULL, -- 'helpful', 'not-helpful', 'neutral'
  comment TEXT,
  feedback_type VARCHAR(50), -- 'thumbs', 'comment', 'clarification_needed'
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  authenticated_user VARCHAR(100) -- from auth middleware
);

-- Index for fast lookups
CREATE INDEX idx_feedback_conversation ON feedback(conversation_id);
CREATE INDEX idx_feedback_rating ON feedback(rating);
CREATE INDEX idx_feedback_user ON feedback(authenticated_user);
```

### **Phase 2: Rich Analytics Tables**

```sql
-- Track question categories and success
CREATE TABLE question_analytics (
  id SERIAL PRIMARY KEY,
  question_normalized TEXT, -- lowercased, trimmed
  question_category VARCHAR(100), -- 'georgia_counts', 'intranet', 'policies', etc
  total_asks INTEGER DEFAULT 0,
  successful_answers INTEGER DEFAULT 0,
  failed_answers INTEGER DEFAULT 0,
  avg_response_time_ms INTEGER,
  last_asked TIMESTAMP,
  common_sources JSONB -- array of frequently returned sources
);

-- Track source performance
CREATE TABLE source_performance (
  id SERIAL PRIMARY KEY,
  source_file VARCHAR(500) NOT NULL UNIQUE,
  source_url TEXT,
  category VARCHAR(100),
  times_returned INTEGER DEFAULT 0,
  times_helpful INTEGER DEFAULT 0,
  times_not_helpful INTEGER DEFAULT 0,
  avg_score DECIMAL(3,2), -- calculated from feedback
  last_used TIMESTAMP,
  common_queries JSONB -- array of queries that trigger this source
);

-- Track user patterns (anonymous)
CREATE TABLE user_patterns (
  id SERIAL PRIMARY KEY,
  session_id VARCHAR(32) REFERENCES conversation_sessions(session_id),
  question_sequence JSONB, -- array of question types in order
  session_length_minutes INTEGER,
  total_questions INTEGER,
  satisfaction_score DECIMAL(3,2), -- avg of feedback ratings
  topics_discussed JSONB, -- array of topics
  ended_at TIMESTAMP
);
```

### **Phase 3: Real-Time Learning**

Use Postgres to automatically boost/lower source scores:

```sql
-- Function to update source scores after feedback
CREATE OR REPLACE FUNCTION update_source_scores()
RETURNS TRIGGER AS $$
BEGIN
  -- When feedback is given, update source_performance for all sources in that conversation
  IF NEW.rating IS NOT NULL THEN
    UPDATE source_performance sp
    SET
      times_helpful = times_helpful + CASE WHEN NEW.rating = 'helpful' THEN 1 ELSE 0 END,
      times_not_helpful = times_not_helpful + CASE WHEN NEW.rating = 'not-helpful' THEN 1 ELSE 0 END,
      avg_score = (times_helpful::DECIMAL / NULLIF(times_helpful + times_not_helpful, 0)),
      last_used = CURRENT_TIMESTAMP
    WHERE source_file IN (
      SELECT jsonb_array_elements(c.sources)->>'sourceFile'
      FROM conversations c
      WHERE c.id = NEW.id
    );
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger to run after feedback is updated
CREATE TRIGGER feedback_updates_sources
AFTER UPDATE OF feedback_rating ON conversations
FOR EACH ROW
EXECUTE FUNCTION update_source_scores();
```

## 🎯 Benefits of This Approach

### 1. **Faster Feedback Loop**
- Feedback → Postgres → Immediately affects next query
- No waiting for Sheets sync

### 2. **Rich Analytics**
Query examples:
```sql
-- What questions fail most often?
SELECT question, COUNT(*) as fails
FROM conversations
WHERE feedback_rating = -1
GROUP BY question
ORDER BY fails DESC
LIMIT 10;

-- Which sources are most/least helpful?
SELECT source_file,
       times_helpful,
       times_not_helpful,
       avg_score
FROM source_performance
ORDER BY avg_score DESC;

-- What time of day gets the worst feedback?
SELECT EXTRACT(HOUR FROM timestamp) as hour,
       AVG(feedback_rating) as avg_rating
FROM conversations
WHERE feedback_rating IS NOT NULL
GROUP BY hour
ORDER BY hour;
```

### 3. **Conversation Pattern Analysis**
```sql
-- Sessions that ended in frustration (multiple negative feedback)
SELECT session_id,
       COUNT(*) as negative_count,
       array_agg(question) as frustrated_questions
FROM conversations
WHERE feedback_rating = -1
GROUP BY session_id
HAVING COUNT(*) >= 2;
```

### 4. **Source Boosting in Retrieval**
Modify `retrieve.js` to query Postgres for source scores:

```javascript
// After getting matches from Pinecone
const sourceScores = await getSourceScoresFromPostgres(matches.map(m => m.metadata.sourceFile));

// Boost scores based on feedback
matches = matches.map(match => {
  const sourceScore = sourceScores[match.metadata.sourceFile];
  if (sourceScore) {
    match.adjustedScore = match.score * (1 + sourceScore.avg_score * 0.2); // 20% boost
  }
  return match;
});
```

## 📊 Migration Plan

### Step 1: Create new tables
Run SQL scripts to create feedback, analytics tables

### Step 2: Update feedback endpoint
Modify `/feedback` in server.js to write directly to Postgres instead of Google Sheets

### Step 3: Backfill from Sheets
One-time script to import all Google Sheets feedback into Postgres

### Step 4: Enable auto-learning
Add triggers and functions for real-time score updates

### Step 5: Update retrieve.js
Query source scores from Postgres and apply boosts

### Step 6: Add analytics dashboard endpoint
`GET /api/analytics` - return insights from Postgres

## 🔥 Quick Wins You Can Do NOW

1. **Start logging more in conversations table**
   - Add `retrieved_sources_count`, `top_source_score`, `was_clarification_needed`

2. **Query existing data for insights**
   ```sql
   -- See what your users are actually asking about
   SELECT question, COUNT(*) as frequency
   FROM conversations
   GROUP BY question
   HAVING COUNT(*) > 1
   ORDER BY frequency DESC
   LIMIT 20;
   ```

3. **Track failed queries**
   ```sql
   SELECT question, response_time_ms, sources
   FROM conversations
   WHERE feedback_rating = -1
   ORDER BY timestamp DESC
   LIMIT 50;
   ```

## Would You Like Me To:

- [ ] Create the SQL migration scripts for new tables?
- [ ] Update server.js to save feedback directly to Postgres?
- [ ] Write analytics queries/dashboard endpoint?
- [ ] Modify retrieve.js to use Postgres-based source boosting?
- [ ] Create a backfill script from Google Sheets?

Let me know what you'd like to tackle first!
