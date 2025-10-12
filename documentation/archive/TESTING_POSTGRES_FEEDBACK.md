# Testing Guide: Postgres Feedback Improvements Branch

## Pre-Deployment Checklist

### 1. Verify Branch
```bash
git branch  # Should show: postgres-feedback-improvements
git status  # Should be clean
```

### 2. Environment Variables
Make sure `.env` on server has:
```
USE_POSTGRES=true
DB_HOST=us-east1-001.proxy.kinsta.app
DB_PORT=30050
DB_DATABASE=chat-feedback-system
DB_USERNAME=puffin
DB_PASSWORD=Hw34343434@
```

---

## Test 1: Verify Database Tables & Columns

**Connect to Kinsta PostgreSQL and run:**

```sql
-- Check new columns in conversations table
SELECT column_name, data_type, column_default
FROM information_schema.columns
WHERE table_name = 'conversations'
AND column_name IN ('authenticated_user', 'feedback_type');

-- Expected: Should see 2 rows
-- authenticated_user | character varying | NULL
-- feedback_type      | character varying | 'thumbs'::character varying
```

```sql
-- Check new columns in source_scores table
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'source_scores'
AND column_name IN ('source_type', 'category', 'confidence_score', 'last_helpful', 'last_not_helpful');

-- Expected: Should see 5 rows
```

```sql
-- Check new tables exist
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
AND table_name IN ('question_analytics', 'user_patterns');

-- Expected: 2 rows
```

```sql
-- Check new views exist
SELECT table_name FROM information_schema.views
WHERE table_schema = 'public'
AND table_name IN ('top_sources', 'problematic_sources', 'question_success_rates', 'daily_feedback_summary', 'session_satisfaction');

-- Expected: 5 rows
```

**✅ PASS:** All tables, columns, and views exist

---

## Test 2: Test Chat Functionality (Baseline)

### A. Start a Chat Session
```bash
# From your test client or browser
POST https://your-test-server.com/chat
Headers:
  x-api-key: YOUR_API_KEY
  Content-Type: application/json
Body:
{
  "question": "What is the CAES welcome message?",
  "session_id": "test-session-001"
}
```

**Expected Response:**
- Should get an answer
- Response should include `session_id`
- Response should include `sources` array

**Save for later:**
- Note the `session_id` returned
- Check if answer mentions the welcome message

### B. Verify Conversation Saved to Postgres

```sql
SELECT session_id, question, answer, sources, timestamp
FROM conversations
WHERE session_id = 'test-session-001'
ORDER BY timestamp DESC
LIMIT 1;
```

**✅ PASS:** Row exists with your question and answer

---

## Test 3: Test Feedback Endpoint (NEW FUNCTIONALITY)

### A. Submit Helpful Feedback
```bash
POST https://your-test-server.com/feedback
Headers:
  x-api-key: YOUR_API_KEY
  Content-Type: application/json
Body:
{
  "session_id": "test-session-001",
  "rating": "helpful",
  "comment": "This was very helpful!",
  "question": "What is the CAES welcome message?",
  "answer": "Welcome to...",
  "sources": []
}
```

**Expected Response:**
```json
{
  "success": true,
  "message": "Feedback recorded",
  "location": "Postgres + Google Sheets"
}
```

**Check Server Logs:**
Should see:
```
✅ Feedback saved to Postgres
✅ Updated feedback for session test-session-001: rating=1, user=anonymous
📝 Feedback received: {...}
```

### B. Verify Feedback in Postgres

```sql
-- Check feedback was saved to conversations
SELECT
  session_id,
  question,
  feedback_rating,
  feedback_comment,
  authenticated_user,
  feedback_type
FROM conversations
WHERE session_id = 'test-session-001';
```

**Expected:**
- `feedback_rating` = 1 (helpful)
- `feedback_comment` = 'This was very helpful!'
- `authenticated_user` = 'anonymous'
- `feedback_type` = 'comment' (because comment was provided)

**✅ PASS:** Feedback saved correctly

### C. Verify Auto-Trigger Updated source_scores

```sql
-- Check if source_scores was updated
-- (Only works if sources were returned in the conversation)
SELECT
  source_key,
  helpful,
  not_helpful,
  total,
  score,
  confidence_score,
  last_helpful
FROM source_scores
WHERE last_helpful > NOW() - INTERVAL '5 minutes'
ORDER BY last_helpful DESC
LIMIT 5;
```

**Expected:**
- If sources were in the conversation, you should see updated rows
- `helpful` count should have increased by 1
- `last_helpful` timestamp should be recent
- `score` should be recalculated

**✅ PASS:** Auto-trigger working

---

## Test 4: Test Analytics Endpoint (NEW)

### A. Query Analytics
```bash
GET https://your-test-server.com/api/analytics
Headers:
  x-api-key: YOUR_API_KEY
```

**Expected Response Structure:**
```json
{
  "success": true,
  "generated_at": "2025-10-08T...",
  "period": "Last 30 days",
  "overall_stats": {
    "total_sessions": 123,
    "total_conversations": 456,
    "total_feedback": 89,
    "helpful_count": 67,
    "not_helpful_count": 22,
    "avg_response_time_ms": "1234"
  },
  "daily_summary": [...],
  "top_sources": [...],
  "problematic_sources": [...],
  "question_success": [...],
  "session_satisfaction": [...]
}
```

**✅ PASS:** Analytics endpoint returns data without errors

### B. Verify Data Quality

Check that:
- `overall_stats` shows reasonable numbers
- `daily_summary` has entries for recent days
- `top_sources` shows sources with scores
- Response time < 2 seconds

---

## Test 5: Test Homepage/Welcome Message Detection (FROM EARLIER WORK)

### A. Ask About Welcome Message
```bash
POST /chat
{
  "question": "What is the CAES welcome message?",
  "session_id": "test-welcome-001"
}
```

**Expected:**
- Should detect "welcome message" keyword
- Should filter to `isHomepage: true` sources
- Should return content from `index.md`
- Answer should include "Welcome to the UGA College of Agricultural and Environmental Sciences"

**✅ PASS:** Homepage detection working

---

## Test 6: Test Vague Question Clarification (FROM EARLIER WORK)

### A. Ask Vague Question
```bash
POST /chat
{
  "question": "How do I send an email?",
  "session_id": "test-vague-001"
}
```

**Expected Response:**
Should trigger clarification:
```
I'd like to help answer your question, but I need a bit more context. Which system or area are you asking about?

Common systems:
- Georgia Counts (reporting system)
- Intranet (internal resources)
- ABO (business office)
- WordPress (websites)
- Extension (programs & resources)
- Or specify another system
```

**✅ PASS:** Clarification mode working

---

## Test 7: Stress Test - Multiple Feedback

Submit 5 feedback entries for different sessions:

```bash
for i in {1..5}; do
  # Submit chat
  POST /chat {"question": "Test $i", "session_id": "stress-test-$i"}

  # Submit feedback
  POST /feedback {"session_id": "stress-test-$i", "rating": "helpful"}
done
```

**Then check:**
```sql
-- Should see 5 new conversations with feedback
SELECT COUNT(*)
FROM conversations
WHERE session_id LIKE 'stress-test-%';
-- Expected: 5
```

**✅ PASS:** No errors, all feedback saved

---

## Test 8: Check Google Sheets Backup

**Verify feedback still goes to Google Sheets:**
1. Open your Google Sheet: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID
2. Check last few rows
3. Should see the test feedback entries

**✅ PASS:** Dual-storage working (Postgres + Sheets)

---

## Test 9: Query the New Views Directly

### Daily Summary
```sql
SELECT * FROM daily_feedback_summary
WHERE date > NOW() - INTERVAL '7 days'
ORDER BY date DESC;
```

### Top Sources
```sql
SELECT * FROM top_sources LIMIT 10;
```

### Problematic Sources
```sql
SELECT * FROM problematic_sources LIMIT 5;
```

### Session Satisfaction
```sql
SELECT * FROM session_satisfaction LIMIT 10;
```

**✅ PASS:** All views return data without errors

---

## Test 10: Verify Trigger is Working

### A. Insert Test Feedback Manually
```sql
-- Get a conversation ID
SELECT id, session_id FROM conversations LIMIT 1;

-- Update its feedback
UPDATE conversations
SET feedback_rating = 1, feedback_comment = 'trigger test'
WHERE id = [ID_FROM_ABOVE];
```

### B. Check Trigger Fired
```sql
-- Check source_scores was updated
SELECT source_key, helpful, total, updated_at
FROM source_scores
WHERE updated_at > NOW() - INTERVAL '1 minute';
```

**Expected:**
- `updated_at` timestamp should be recent
- `helpful` count should match

**✅ PASS:** Trigger auto-updating source_scores

---

## Rollback Plan (If Something Breaks)

### Option 1: Quick Rollback
```bash
git checkout main
# Re-deploy main branch
```

### Option 2: Database Rollback
```sql
-- Drop new tables (keeps existing data safe)
DROP TABLE IF EXISTS user_patterns;
DROP TABLE IF EXISTS question_analytics;

-- Remove new columns
ALTER TABLE conversations DROP COLUMN IF EXISTS authenticated_user;
ALTER TABLE conversations DROP COLUMN IF EXISTS feedback_type;

ALTER TABLE source_scores DROP COLUMN IF EXISTS source_type;
ALTER TABLE source_scores DROP COLUMN IF EXISTS category;
ALTER TABLE source_scores DROP COLUMN IF EXISTS confidence_score;
ALTER TABLE source_scores DROP COLUMN IF EXISTS last_helpful;
ALTER TABLE source_scores DROP COLUMN IF EXISTS last_not_helpful;

-- Drop trigger
DROP TRIGGER IF EXISTS conversation_feedback_update_sources ON conversations;
DROP FUNCTION IF EXISTS update_source_scores_from_conversation();

-- Drop views
DROP VIEW IF EXISTS top_sources;
DROP VIEW IF EXISTS problematic_sources;
DROP VIEW IF EXISTS question_success_rates;
DROP VIEW IF EXISTS daily_feedback_summary;
DROP VIEW IF EXISTS session_satisfaction;
```

---

## Success Criteria

### ✅ All Tests Pass If:

1. ✅ Database has all new tables, columns, views
2. ✅ Chat functionality works normally
3. ✅ Feedback saves to Postgres (check conversations table)
4. ✅ Feedback also saves to Google Sheets (backup)
5. ✅ Auto-trigger updates source_scores
6. ✅ Analytics endpoint returns data
7. ✅ Homepage detection works
8. ✅ Clarification mode works
9. ✅ No errors in server logs
10. ✅ Response times are normal

### 🚀 Ready to Merge to Main If:
- All 10 tests pass
- No performance degradation
- Server logs show no errors
- You've tested for at least 30 minutes with real usage

---

## Monitoring After Deploy

### Watch These Metrics:
```sql
-- Check feedback is being saved
SELECT COUNT(*), DATE(timestamp)
FROM conversations
WHERE feedback_rating IS NOT NULL
GROUP BY DATE(timestamp)
ORDER BY DATE(timestamp) DESC
LIMIT 7;

-- Check source scores are updating
SELECT COUNT(*) as sources_with_feedback,
       AVG(total) as avg_feedback_per_source,
       MAX(updated_at) as last_update
FROM source_scores;
```

### Watch Server Logs For:
- ✅ "Feedback saved to Postgres"
- ✅ "Updated feedback for session..."
- ❌ Any Postgres connection errors
- ❌ Any "Failed to save to Postgres" warnings

---

## Questions to Answer During Testing:

1. **Does feedback save faster?** (Should, no Sheets API wait)
2. **Are analytics useful?** (Check the data quality)
3. **Does source boosting improve answers?** (Compare before/after)
4. **Any performance issues?** (Monitor response times)

Good luck! 🚀
