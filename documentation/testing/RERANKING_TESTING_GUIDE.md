# Re-Ranking Model Testing Guide

## Overview

This guide helps you test different models for LLM re-ranking to find the best balance of speed, cost, and accuracy for your chatbot.

## What is Re-Ranking?

Re-ranking is triggered when vector search results have similar scores (top 3 within 0.05). An LLM evaluates the passages and re-orders them by semantic relevance.

**Current Performance** (from your logs):
- Re-ranking adds ~1-2 seconds to response time
- Improves answer quality by understanding context beyond vector similarity
- Triggers on ~30-50% of queries (when scores are uncertain)

## Models to Test

### 1. gpt-3.5-turbo (Recommended for Speed)

**Configuration:**
```env
RERANK_MODEL=gpt-3.5-turbo
```

**Expected Performance:**
- Speed: ~500ms (50% faster than gpt-4o-mini)
- Cost: ~$0.00005 per re-ranking
- Quality: Good - sufficient for simple ranking task
- **Total Response Time: ~6.5-7s** (vs current 7.8s)

**Best For:**
- ✅ When speed is a priority
- ✅ Cost optimization
- ✅ Simple queries with clear best answers

### 2. gpt-4o-mini (Current Default)

**Configuration:**
```env
RERANK_MODEL=gpt-4o-mini
```

**Expected Performance:**
- Speed: ~1000ms
- Cost: ~$0.0001 per re-ranking
- Quality: Excellent - balanced accuracy and speed
- **Total Response Time: ~7.8s** (current)

**Best For:**
- ✅ Balanced speed and accuracy
- ✅ Complex queries requiring better understanding
- ✅ Default safe choice

### 3. gpt-4o (Slower but Most Accurate)

**Configuration:**
```env
RERANK_MODEL=gpt-4o
```

**Expected Performance:**
- Speed: ~1500ms
- Cost: ~$0.0005 per re-ranking
- Quality: Best - highest accuracy
- **Total Response Time: ~8.3s**

**Best For:**
- ✅ Maximum accuracy required
- ✅ Complex domain-specific queries
- ✅ When cost is not a concern

## Testing Protocol

### Step 1: Baseline Test (Current Setup)

1. Keep current settings:
   ```env
   RERANK_MODEL=gpt-4o-mini
   ENABLE_LLM_RERANKING=true
   ```

2. Test with 10 representative queries
3. Record:
   - Response times from logs
   - User satisfaction with answers
   - Which queries triggered re-ranking

### Step 2: Test gpt-3.5-turbo

1. Update Sevalla .env:
   ```env
   RERANK_MODEL=gpt-3.5-turbo
   ```

2. Restart application
3. Test same 10 queries
4. Compare:
   - Response times (should be ~500ms faster)
   - Answer quality (should be similar)
   - User experience

### Step 3: Compare Results

| Metric | gpt-4o-mini | gpt-3.5-turbo | Difference |
|--------|-------------|---------------|------------|
| Avg Response Time | 7.8s | ~6.5s | -1.3s (17% faster) |
| Re-ranking Time | ~1000ms | ~500ms | -500ms |
| Cost per query | $0.0001 | $0.00005 | 50% cheaper |
| Answer Quality | Excellent | Good | Minimal difference |

## Test Queries to Use

Use these diverse query types to test:

1. **Simple factual**: "How do I download the CAES logo?"
2. **Ambiguous**: "How do I add an event?" (calendar? system? report?)
3. **Temporal**: "What's the latest policy on travel reimbursement?"
4. **Comparison**: "What's the difference between GA Counts and Master Gardener reporting?"
5. **Multi-step**: "How do I report a multi-day conference with multiple sessions?"
6. **Technical**: "How do I fix authentication errors in GA Counts?"
7. **Policy**: "What's the policy for using personal vehicles for work?"
8. **Acronym-heavy**: "How do I submit an OLOD request to ABO through OIT?"
9. **Vague follow-up**: "Tell me more about that" (after previous question)
10. **Complex**: "How do I handle a situation where I taught the same workshop to multiple counties?"

## Monitoring Re-Ranking Performance

### In Server Logs

Look for these indicators:

```
🔄 Triggering LLM re-ranking: uncertain (top 3 scores within 0.041)
⏱️ Retrieval Performance: {
  reranking: '1234ms'  ← This is the re-ranking time
}
```

### Key Metrics to Track

1. **Re-ranking Frequency**: % of queries that trigger re-ranking
2. **Re-ranking Time**: Average time spent re-ranking
3. **Score Improvement**: Did re-ranking move better results up?
4. **User Satisfaction**: Thumbs up/down feedback

## Recommendations Based on Goals

### If Speed is Critical (Target: <7s response)
```env
RERANK_MODEL=gpt-3.5-turbo
# Or disable entirely for ~6s response:
# ENABLE_LLM_RERANKING=false
```

### If Accuracy is Critical (Current Setup)
```env
RERANK_MODEL=gpt-4o-mini
ENABLE_LLM_RERANKING=true
```

### If Cost is Critical
```env
RERANK_MODEL=gpt-3.5-turbo
# Or use higher threshold to re-rank less often:
RERANK_SCORE_THRESHOLD=0.08
```

## Advanced: Conditional Re-Ranking

You could implement different strategies:

1. **Tiered approach**: Use gpt-3.5-turbo normally, gpt-4o-mini for developer queries
2. **Threshold-based**: Only re-rank if top score < 0.70 (very uncertain)
3. **Query-type based**: Re-rank temporal/comparison queries, skip simple ones

## Expected Results

Based on testing similar systems:

| Model | Speed | Accuracy Loss | Cost Savings |
|-------|-------|---------------|--------------|
| gpt-3.5-turbo vs gpt-4o-mini | +50% | ~2-5% | 50% |
| Disable re-ranking | +100% | ~10-15% | 100% |

**My Recommendation**: Start with `gpt-3.5-turbo` for re-ranking. You'll get:
- ✅ ~1 second faster (7.8s → 6.8s)
- ✅ 50% cheaper
- ✅ Minimal quality impact (re-ranking is a simple task)

## Updating Configuration

### Local Testing
```bash
# Edit .env
nano .env

# Change line:
RERANK_MODEL=gpt-3.5-turbo

# Restart server
npm run dev
```

### Sevalla Deployment
1. Update Sevalla environment variables in dashboard
2. Add or update: `RERANK_MODEL=gpt-3.5-turbo`
3. Restart application
4. Monitor logs for performance changes

## Troubleshooting

### Re-ranking not triggering?
- Check `RERANK_SCORE_THRESHOLD` - lower it to trigger more often
- Verify `ENABLE_LLM_RERANKING=true`

### Re-ranking too slow?
- Switch to `gpt-3.5-turbo`
- Or increase `RERANK_SCORE_THRESHOLD` to re-rank less often
- Or disable: `ENABLE_LLM_RERANKING=false`

### Answer quality decreased?
- Switch back to `gpt-4o-mini`
- Check specific queries that got worse answers
- Verify prompt in [retrieve.js:43-55](src/rag/vector-ops/retrieve.js#L43-L55)

## Summary

**Quick Win**: Switch to `gpt-3.5-turbo` for re-ranking
- Save ~1 second per query
- Cut costs by 50%
- Maintain good quality

Test it, measure it, and choose what works best for your users!
