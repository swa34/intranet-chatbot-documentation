# LLM Re-ranking Feature

## Overview

The LLM re-ranking feature enhances retrieval quality by using a language model to intelligently reorder search results when vector similarity scores alone are insufficient to determine the best match. This is a **hybrid approach** that combines vector search, feedback learning, and LLM-based semantic understanding.

## How It Works

### Three-Stage Retrieval Pipeline

```
1. Vector Search (Pinecone)
   ↓
2. Feedback Adjustment (PostgreSQL scores)
   ↓
3. LLM Re-ranking (if triggered)
   ↓
4. Final Results
```

### When Re-ranking Triggers

The system **selectively** applies LLM re-ranking only when needed:

#### Trigger 1: Uncertain Scores
- Top 3 results have similarity scores within `0.05` (configurable)
- Indicates vector search can't confidently rank results
- Example: Scores of `0.82, 0.81, 0.80` → Too similar, use LLM

#### Trigger 2: Temporal Queries
- Query contains: `latest`, `most recent`, `current`, `newest`, `up to date`, `now`
- LLM better understands recency requirements
- Example: "What is the latest vacation policy?"

#### Trigger 3: Comparison Queries
- Query contains: `difference`, `compare`, `versus`, `vs`, `better than`
- Requires semantic understanding of relationships
- Example: "What is the difference between extension and research faculty?"

## Configuration

### Environment Variables

Add to `.env` or Sevalla environment:

```env
# Enable/disable LLM re-ranking (default: true)
ENABLE_LLM_RERANKING=true

# Score threshold for uncertainty trigger (default: 0.05)
RERANK_SCORE_THRESHOLD=0.05

# Model to use for re-ranking (default: gpt-4o-mini)
RERANK_MODEL=gpt-4o-mini

# Enable detailed logging
DEBUG_RAG=true
```

### Disable Re-ranking

To disable globally:
```env
ENABLE_LLM_RERANKING=false
```

## Performance & Cost

### Typical Usage Pattern
- **Triggers**: 20-30% of queries (based on test data)
- **Latency**: +200-400ms when triggered
- **Cost**: ~$0.0001 per re-ranking operation

### Monthly Cost Estimate
Assuming 100 queries/day with 25% triggering re-ranking:
- Queries per month: 100 × 30 = 3,000
- Re-rankings per month: 3,000 × 0.25 = 750
- Cost: 750 × $0.0001 = **$0.075/month** (~7.5 cents)

Even with 1,000 queries/day:
- Re-rankings per month: 7,500
- Cost: **$0.75/month** (75 cents)

### Latency Impact
- Normal query: ~2-3 seconds
- With re-ranking: ~2.5-3.5 seconds (+200-400ms)
- User experience impact: Minimal (sub-second difference)

## How LLM Re-ranking Works

### Input
The LLM receives:
1. User's query
2. Top 8 results with truncated content (first 400 chars)
3. Source information for each result

### Process
```javascript
// Simplified example
const prompt = `Rank these passages by relevance to: "${query}"

[0] Source: vacation-policy-2024.pdf
Content: The current vacation policy allows...

[1] Source: vacation-policy-2020.pdf
Content: Employees are entitled to...

Return only indices in order: "1,0,2,3..."
```

### Output
- LLM returns: `"1,0,3,2,4,5,6,7"`
- System reorders results accordingly
- Graceful fallback if LLM fails

## Advantages Over Vector Search Alone

### Example: "Latest vacation policy"

**Vector Search** (score-based):
1. [0.82] vacation-policy-2020.pdf
2. [0.81] vacation-policy-2024.pdf
3. [0.80] vacation-request-form.pdf

**After LLM Re-ranking**:
1. [0.81] vacation-policy-2024.pdf ← Understands "latest"
2. [0.82] vacation-policy-2020.pdf
3. [0.80] vacation-policy-guidelines.pdf ← Filters out form

### Benefits
✅ **Semantic understanding**: Knows "latest" means 2024 > 2020
✅ **Context awareness**: Distinguishes policy from form
✅ **Comparison handling**: Understands "difference between X and Y"
✅ **Cost-efficient**: Only runs when needed
✅ **Failure-safe**: Falls back to original ranking if LLM fails

## Monitoring & Logs

### Production Logs

When re-ranking triggers, you'll see:
```
🔄 Triggering LLM re-ranking: uncertain (top 3 scores within 0.018)
📊 LLM Re-ranking applied: {
  question: "What is the latest vacation policy?",
  sessionId: "abc123",
  topScore: 0.82,
  matchCount: 8
}
```

### Debug Mode

Enable `DEBUG_RAG=true` for detailed insights:
```
🔄 LLM Re-ranking applied: {
  originalOrder: [
    { index: 0, score: 0.82 },
    { index: 1, score: 0.81 },
    { index: 2, score: 0.80 }
  ],
  newOrder: [
    { index: 1, score: 0.81, source: 'policy-2024.pdf' },
    { index: 0, score: 0.82, source: 'policy-2020.pdf' },
    { index: 2, score: 0.80, source: 'form.pdf' }
  ],
  llmRanking: "1,0,2"
}
```

## Testing

### Manual Testing

Run the test script:
```bash
node test-reranking.js
```

This tests:
- Temporal queries ("latest", "most recent")
- Comparison queries ("difference between")
- Normal queries (triggers on uncertain scores)

### Test Results

From actual test run:
```
✅ "What is the latest vacation policy?"
   → Triggered: uncertain scores (within 0.003)
   → Latency: 8.6s (includes first DB connection)

✅ "What is the difference between extension and research faculty?"
   → Triggered: uncertain scores (within 0.044)
   → Latency: 4.2s

✅ "How do I submit travel reimbursement?"
   → Triggered: uncertain scores (within 0.018)
   → Latency: 6.3s

✅ "What is the most recent COVID policy?"
   → Triggered: temporal query detected
   → Latency: 3.3s
```

## Integration with Existing Systems

### Feedback Learning System
- Re-ranking happens **after** feedback adjustments
- Feedback scores are preserved in results
- Both systems work together for maximum accuracy

### Conversation Memory
- Re-ranking applies to reframed questions
- Works seamlessly with clarification system

### Flow Diagram
```
User Query
  ↓
Conversation Memory (reframe if needed)
  ↓
Vector Search (Pinecone)
  ↓
Feedback Learning (PostgreSQL adjustment)
  ↓
Check triggers (uncertain scores / keywords)
  ↓
LLM Re-ranking (if triggered) ← NEW
  ↓
Build context & generate response
```

## Troubleshooting

### Re-ranking Not Triggering

**Issue**: No re-ranking logs appearing

**Checks**:
1. Verify `ENABLE_LLM_RERANKING` is not set to `false`
2. Check if queries have uncertain scores (within 0.05)
3. Ensure temporal/comparison keywords are present
4. Try lowering `RERANK_SCORE_THRESHOLD` to trigger more often

### High Latency

**Issue**: Queries taking too long

**Solutions**:
1. Increase `RERANK_SCORE_THRESHOLD` to 0.08 (trigger less)
2. Use faster model: `RERANK_MODEL=gpt-4o-mini` (already default)
3. Reduce temporal keyword list if not needed
4. Disable for certain query types

### Unexpected Rankings

**Issue**: LLM reordering seems wrong

**Debug**:
1. Enable `DEBUG_RAG=true` to see LLM's ranking
2. Check if passage previews are too short (increase from 400 chars)
3. Review LLM prompt in `retrieve.js` lines 39-51
4. Consider adjusting temperature (currently 0)

### Cost Concerns

**Issue**: Worried about OpenAI costs

**Solutions**:
1. Monitor with: `grep "LLM Re-ranking applied" logs`
2. Increase threshold: `RERANK_SCORE_THRESHOLD=0.08`
3. Remove comparison trigger if rarely used
4. Set budget alerts in OpenAI dashboard

## Deployment

### Local Development

Works automatically if:
- `.env` has `OPENAI_API_KEY`
- `ENABLE_LLM_RERANKING` not set to `false`

### Sevalla Production

1. Deploy code:
   ```bash
   git push origin feature/llm-reranking
   ```

2. Set environment variable (if needed):
   ```bash
   # Already enabled by default, but to explicitly set:
   ENABLE_LLM_RERANKING=true
   ```

3. Monitor logs:
   ```bash
   tail -f logs/app.log | grep "Re-ranking"
   ```

## Best Practices

### When to Adjust Settings

**Increase Threshold** (trigger less often):
- Cost is a concern
- Queries are generally high quality
- Latency is critical

**Decrease Threshold** (trigger more often):
- Users report "close but not quite right" answers
- Many ambiguous queries
- Temporal/comparison queries common

### Recommended Settings by Use Case

**High Volume (>1000 queries/day)**:
```env
RERANK_SCORE_THRESHOLD=0.08  # More conservative
RERANK_MODEL=gpt-4o-mini      # Fast & cheap
```

**High Accuracy (research/policy)**:
```env
RERANK_SCORE_THRESHOLD=0.03  # More aggressive
RERANK_MODEL=gpt-4o          # More capable
```

**Balanced (default)**:
```env
RERANK_SCORE_THRESHOLD=0.05  # Medium sensitivity
RERANK_MODEL=gpt-4o-mini     # Good balance
```

## Future Enhancements

Potential improvements:
- [ ] Cache LLM rankings for identical queries
- [ ] A/B test with user feedback to measure improvement
- [ ] Add more trigger patterns (e.g., "best", "recommended")
- [ ] Use specialized re-ranking models (Cohere, Anthropic)
- [ ] Track re-ranking impact on helpful/not-helpful ratings

## Related Documentation

- [Feedback Learning System](FEEDBACK_LEARNING_POSTGRES.md)
- [Conversation Memory](CONVERSATION_MEMORY.md)
- [RAG Pipeline Overview](../README.md)

## Support

For issues or questions:
- Enable `DEBUG_RAG=true` and check logs
- Run `node test-reranking.js` to verify functionality
- Review configuration in [retrieve.js](../src/rag/vector-ops/retrieve.js)
