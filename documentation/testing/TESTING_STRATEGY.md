# Intranet Chatbot Testing Strategy

## Overview

This document outlines the comprehensive testing strategy for the Intranet Chatbot, including phased rollout, performance optimization strategies, and technical testing procedures. This is designed for developers, project managers, and technical stakeholders.

---

## Table of Contents

1. [Testing Phases & Timeline](#testing-phases--timeline)
2. [Performance Optimization Strategy](#performance-optimization-strategy)
3. [PostgreSQL Feedback System Testing](#postgresql-feedback-system-testing)

---

## Testing Phases & Timeline

### Phased Rollout Approach

This phased rollout ensures thorough training and refinement before organization-wide deployment. Each phase builds on learnings from the previous, with conservative timelines to allow for fixes and improvements.

### Phase 1: Internal Web Team Testing

**Status:** Currently Active
**Duration:** 2 weeks
**Participants:** 4 internal web team testers

**Focus:**
- Technical functionality validation
- Initial feedback system testing
- Core question/answer accuracy
- Bug identification and fixes

**Success Criteria:**
- Feedback system working correctly
- Basic navigation questions answered accurately
- No critical bugs or errors
- Initial learning data collected

### Phase 2: Limited External Testing (OIT Adjacent)

**Target Start:** 2 weeks from initial start (Friday after next)
**Duration:** 2 weeks
**Participants:** 3-5 testers from departments close to OIT

**Focus:**
- Test with tech-savvy but non-developer users
- Validate UI/UX for non-technical staff
- Gather feedback on response clarity
- Test department-specific questions outside web team

**Success Criteria:**
- 80% positive feedback rate
- Clear identification of knowledge gaps
- UI improvements implemented
- Feedback learning system showing improvement

**Between Phase 2-3:** 1 week for fixes and improvements

### Phase 3: Department Representative Testing

**Target Start:** 5 weeks from initial start
**Duration:** 3 weeks
**Participants:** 1-2 testers per department (10-15 departments)

**Focus:**
- Comprehensive department-specific question testing
- Follow the TESTING_GUIDE.md protocol
- 10-15 questions per department
- Include new employee perspective where possible

**Success Criteria:**
- All departments provide feedback
- 75%+ helpful response rate
- Department-specific knowledge gaps identified
- Feedback system has substantial training data

**Between Phase 3-4:** 2 weeks for major improvements and retraining

### Phase 4: Department Heads Review

**Target Start:** 10 weeks from initial start
**Duration:** 2 weeks
**Participants:** Department heads and senior management

**Focus:**
- Policy and procedure accuracy verification
- Approval of department-specific responses
- Strategic alignment check
- Change management preparation

**Success Criteria:**
- Department head approval/sign-off
- Policy accuracy verified
- Communication plan developed
- Training materials prepared

**Between Phase 4-5:** 2 weeks for final refinements

### Phase 5: Organization-Wide Beta

**Target Start:** 14 weeks from initial start (3.5 months from today)
**Duration:** 4 weeks monitored beta
**Participants:** All employees (opt-in initially, then full rollout)

**Focus:**
- Stress testing with full user load
- Monitor feedback trends
- Continuous learning enabled
- Support structure in place

**Success Criteria:**
- System handles concurrent users
- 70%+ positive feedback maintained
- Support tickets manageable
- Continuous learning improving responses

### Post-Beta: Full Production

**Target:** 18 weeks from initial start (4.5 months from today)
**Status:** General availability with continuous improvement

---

## Testing Gates & Milestones

### Gate 1: After Phase 1 (2 weeks)
- [ ] Feedback system fully operational
- [ ] Core functionality stable
- [ ] Basic knowledge base validated
- **Go/No-Go Decision for Phase 2**

### Gate 2: After Phase 2 (4 weeks)
- [ ] UI/UX validated by non-technical users
- [ ] Response clarity acceptable
- [ ] No major technical issues
- **Go/No-Go Decision for Phase 3**

### Gate 3: After Phase 3 (8 weeks)
- [ ] Department coverage complete
- [ ] 75%+ helpful rate achieved
- [ ] Major knowledge gaps addressed
- **Go/No-Go Decision for Phase 4**

### Gate 4: After Phase 4 (12 weeks)
- [ ] Executive approval received
- [ ] Policies and procedures verified
- [ ] Communication plan ready
- **Go/No-Go Decision for Phase 5**

### Gate 5: Beta Completion (16 weeks)
- [ ] Performance under load verified
- [ ] Support processes working
- [ ] Continuous learning effective
- **Go/No-Go Decision for Full Production**

---

## Risk Mitigation

### Conservative Timeline Rationale

- **Extra buffer time** between phases for unexpected issues
- **Learning curve** for feedback system optimization
- **Content updates** may require manual intervention
- **Department availability** for testing coordination
- **Integration issues** with existing systems
- **Change management** needs time for adoption

### Contingency Plans

- **If Phase 2 fails:** Extend internal testing by 2 weeks
- **If Phase 3 has low participation:** Extend by 1 week with follow-ups
- **If Phase 4 reveals major issues:** Add 2-week remediation phase
- **If Beta shows problems:** Can extend beta by 2-4 weeks

---

## Success Metrics Tracking

### Phase 1-2 Metrics
- Bug count and resolution time
- Response accuracy rate
- System uptime

### Phase 3-4 Metrics
- Feedback scores by department
- Question coverage percentage
- Knowledge gap identification rate

### Phase 5+ Metrics
- Daily active users
- Average feedback score
- Response time
- Escalation rate to human support

---

## Performance Optimization Strategy

### Overview

The chatbot configuration should be optimized differently during the testing phase versus production. During testing, prioritize collecting comprehensive feedback data. During production, prioritize speed and cost efficiency.

### Current Configuration (Testing Phase)

**Purpose:** Collect maximum feedback data

**Configuration:**
```env
MIN_SIMILARITY=0.40
ENABLE_LLM_RERANKING=true
RERANK_MODEL=gpt-5-nano
GPT5_REASONING_EFFORT=minimal
GPT5_VERBOSITY=low
```

**Why Low Similarity Threshold?**
- Retrieve MORE documents (even lower-quality matches)
- Collect feedback on a wider variety of sources
- Train the feedback analyzer with diverse data
- Learn which sources work best for different query types

**Trade-offs:**
- ✅ More comprehensive feedback data
- ✅ Better training for the feedback learning system
- ✅ Discover unexpected relevant sources
- ❌ Slower responses (~1-2s extra processing)
- ❌ More re-ranking triggered
- ❌ Higher API costs

### Performance Impact of Low Threshold

**Current (0.40 threshold):**
- Matches Retrieved: 8-12 documents
- Feedback Adjustment: 12ms (analyzing 8-12 sources)
- Re-ranking Frequency: ~40-50% of queries
- Response Time: 7-8s

**Production (0.70-0.75 threshold):**
- Matches Retrieved: 3-5 documents
- Feedback Adjustment: 5-8ms (analyzing 3-5 sources)
- Re-ranking Frequency: ~20-30% of queries
- Response Time: 5-6s (1-2s faster!)

---

## When to Switch to Production Mode

### Signs You're Ready:

1. **Feedback Volume**
   - Collected 200+ feedback entries
   - At least 50 thumbs down with comments
   - Diverse query patterns represented

2. **Analyzer Trained**
   - Source scores stabilized (check `/api/analytics`)
   - Query patterns identified
   - Top performers and low performers clear

3. **System Understanding**
   - Know which sources work well
   - Identified problematic content
   - Fixed major gaps in documentation

### Check Your Progress:

Visit your analytics dashboard: `https://hospitalitychatbot-r7f2j.sevalla.app/feedback-dashboard.html`

Look for:
- Total feedback count
- Source performance trends
- Common issues identified
- Pattern recognition working

---

## Optimization Migration Plan

### Phase 1: Testing (Current) - Weeks 1-4

**Goals:**
- Collect minimum 200 feedback entries
- Train feedback analyzer
- Identify content gaps

**Configuration:**
```env
MIN_SIMILARITY=0.40
ENABLE_LLM_RERANKING=true
RERANK_MODEL=gpt-5-nano
GPT5_REASONING_EFFORT=minimal
GPT5_VERBOSITY=low
```

**Expected Performance:**
- Response time: 7-8s
- High accuracy (catching edge cases)
- Learning continuously

### Phase 2: Optimization (Future) - Week 5+

**Goals:**
- Reduce response time
- Lower costs
- Maintain quality

**Configuration Changes:**
```env
# Increase threshold (fewer, higher-quality matches)
MIN_SIMILARITY=0.70

# Keep other optimizations
ENABLE_LLM_RERANKING=true
RERANK_MODEL=gpt-5-nano
GPT5_REASONING_EFFORT=minimal
GPT5_VERBOSITY=low
```

**Expected Performance:**
- Response time: 5-6s (25% faster!)
- Maintained accuracy (trained system knows good sources)
- Lower API costs

### Phase 3: Fine-Tuning (Optional) - Week 8+

**Advanced Optimizations:**

1. **Adjust re-ranking threshold:**
   ```env
   # Re-rank less often (only very uncertain)
   RERANK_SCORE_THRESHOLD=0.08
   ```

2. **Consider disabling re-ranking:**
   ```env
   # Feedback system is trained, might not need re-ranking
   ENABLE_LLM_RERANKING=false
   ```
   This saves another 400-600ms per query that triggers re-ranking.

3. **Dynamic threshold based on query type:**
   - Simple queries: 0.75 threshold
   - Complex queries: 0.65 threshold
   - (Requires code changes)

---

## Performance Comparison

### Scenario A: Current Testing Setup
```
Configuration:
- MIN_SIMILARITY=0.40
- ENABLE_LLM_RERANKING=true
- RERANK_MODEL=gpt-5-nano

Performance:
- Matches: 8-12
- Retrieval: 3-4s
- Generation: 3-4s
- Total: 7-8s
- Cost per query: ~$0.002
```

### Scenario B: After Training (Optimized)
```
Configuration:
- MIN_SIMILARITY=0.70
- ENABLE_LLM_RERANKING=true
- RERANK_MODEL=gpt-5-nano

Performance:
- Matches: 3-5
- Retrieval: 2-3s
- Generation: 3-4s
- Total: 5-6s (25% faster!)
- Cost per query: ~$0.0015 (25% cheaper!)
```

### Scenario C: Fully Optimized (Post-Training)
```
Configuration:
- MIN_SIMILARITY=0.75
- ENABLE_LLM_RERANKING=false (trained system doesn't need it)

Performance:
- Matches: 3-4
- Retrieval: 1.5-2s
- Generation: 3-4s
- Total: 4.5-5.5s (35% faster!)
- Cost per query: ~$0.001 (50% cheaper!)
```

---

## Testing the Threshold Change

### Gradual Approach (Recommended):

**Week 1-4: Testing Phase**
```env
MIN_SIMILARITY=0.40
```

**Week 5-6: Intermediate**
```env
MIN_SIMILARITY=0.55
```
Monitor: Did quality drop? Check feedback ratings.

**Week 7+: Production**
```env
MIN_SIMILARITY=0.70
```
Monitor: Stable quality + faster responses?

### A/B Testing Approach:

If you want to be scientific:
1. Keep 0.40 for 50% of queries
2. Use 0.70 for 50% of queries
3. Compare feedback ratings
4. Choose the winner

(Requires code changes to randomize threshold)

---

## Monitoring After Changes

### Key Metrics to Watch:

1. **Response Time**
   ```
   Performance Breakdown: {
     total: 'XXXms'  ← Should decrease
   }
   ```

2. **User Feedback**
   - Thumbs up/down ratio
   - "No results found" complaints
   - Quality of answers

3. **Top Score Average**
   ```
   topScore: 0.XXX ← Should increase (better matches)
   ```

4. **Re-ranking Frequency**
   ```
   rerankingApplied: true/false
   ```
   Should decrease with higher threshold

---

## Optimization Decision Matrix

| Feedback Count | Source Scores Stable | Recommendation |
|----------------|---------------------|----------------|
| < 100 | No | Stay at 0.40 |
| 100-200 | No | Stay at 0.40 |
| 200-500 | Yes | Try 0.60 |
| 500+ | Yes | Try 0.70 |
| 1000+ | Yes | Try 0.75 + consider disabling re-ranking |

---

## PostgreSQL Feedback System Testing

### Overview

This section covers testing procedures for the PostgreSQL feedback improvements branch, which adds enhanced analytics, automated triggers, and new database structures for better feedback tracking.

### Pre-Deployment Checklist

#### 1. Verify Branch
```bash
git branch  # Should show: postgres-feedback-improvements
git status  # Should be clean
```

#### 2. Environment Variables
Make sure `.env` on server has:
```env
USE_POSTGRES=true
DB_HOST=us-east1-001.proxy.kinsta.app
DB_PORT=30050
DB_DATABASE=chat-feedback-system
DB_USERNAME=puffin
DB_PASSWORD=[password]
```

---

### Test 1: Verify Database Tables & Columns

**Connect to PostgreSQL and run:**

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

**Success Criteria:** All tables, columns, and views exist

---

### Test 2: Test Chat Functionality (Baseline)

#### A. Start a Chat Session
```bash
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

#### B. Verify Conversation Saved to Postgres

```sql
SELECT session_id, question, answer, sources, timestamp
FROM conversations
WHERE session_id = 'test-session-001'
ORDER BY timestamp DESC
LIMIT 1;
```

**Success Criteria:** Row exists with your question and answer

---

### Test 3: Test Feedback Endpoint (NEW FUNCTIONALITY)

#### A. Submit Helpful Feedback
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

#### B. Verify Feedback in Postgres

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

#### C. Verify Auto-Trigger Updated source_scores

```sql
-- Check if source_scores was updated
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

**Success Criteria:** Auto-trigger working

---

### Test 4: Test Analytics Endpoint (NEW)

#### A. Query Analytics
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

**Success Criteria:** Analytics endpoint returns data without errors

#### B. Verify Data Quality

Check that:
- `overall_stats` shows reasonable numbers
- `daily_summary` has entries for recent days
- `top_sources` shows sources with scores
- Response time < 2 seconds

---

### Test 5: Test Homepage/Welcome Message Detection

#### A. Ask About Welcome Message
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

**Success Criteria:** Homepage detection working

---

### Test 6: Test Vague Question Clarification

#### A. Ask Vague Question
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

**Success Criteria:** Clarification mode working

---

### Test 7: Stress Test - Multiple Feedback

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

**Success Criteria:** No errors, all feedback saved

---

### Test 8: Check Google Sheets Backup

**Verify feedback still goes to Google Sheets:**
1. Open your Google Sheet
2. Check last few rows
3. Should see the test feedback entries

**Success Criteria:** Dual-storage working (Postgres + Sheets)

---

### Test 9: Query the New Views Directly

#### Daily Summary
```sql
SELECT * FROM daily_feedback_summary
WHERE date > NOW() - INTERVAL '7 days'
ORDER BY date DESC;
```

#### Top Sources
```sql
SELECT * FROM top_sources LIMIT 10;
```

#### Problematic Sources
```sql
SELECT * FROM problematic_sources LIMIT 5;
```

#### Session Satisfaction
```sql
SELECT * FROM session_satisfaction LIMIT 10;
```

**Success Criteria:** All views return data without errors

---

### Test 10: Verify Trigger is Working

#### A. Insert Test Feedback Manually
```sql
-- Get a conversation ID
SELECT id, session_id FROM conversations LIMIT 1;

-- Update its feedback
UPDATE conversations
SET feedback_rating = 1, feedback_comment = 'trigger test'
WHERE id = [ID_FROM_ABOVE];
```

#### B. Check Trigger Fired
```sql
-- Check source_scores was updated
SELECT source_key, helpful, total, updated_at
FROM source_scores
WHERE updated_at > NOW() - INTERVAL '1 minute';
```

**Expected:**
- `updated_at` timestamp should be recent
- `helpful` count should match

**Success Criteria:** Trigger auto-updating source_scores

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

## Success Criteria Summary

### All Tests Pass If:

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

### Ready to Merge to Main If:
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

---

## Communication Plan

### Phase 1-2
- Weekly updates to web team
- Bug tracking in project system

### Phase 3
- Email introduction to department testers
- Weekly progress reports to stakeholders

### Phase 4
- Executive briefings
- Department head meetings

### Phase 5
- Organization-wide announcement
- Training resources available
- Support documentation ready

---

**Note:** This timeline is intentionally conservative to ensure quality over speed. The chatbot will be well-trained and reliable before full deployment, reducing support burden and ensuring positive user adoption.
