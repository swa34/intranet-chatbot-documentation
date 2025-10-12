# Plan: Add Clarification Mode for Ambiguous Follow-up Questions

## Problem
When users ask ambiguous follow-up questions like "how do I log in?" after discussing Georgia Counts, the system automatically assumes they mean Georgia Counts. Better UX would be to ask for clarification.

**Example:**
```
User: "what is ga counts?"
Bot: [Explains Georgia Counts]

User: "how do I log in?"
Current behavior: Auto-reframes to "How do I log in to Georgia Counts?"
Better UX: "Are you asking about logging in to Georgia Counts, or another system?"
```

## Proposed Solution: Two-Tier Strategy

### **Tier 1: High Confidence → Auto-reframe** (keep current behavior)
When context is very clear, automatically reframe:
- Very obvious follow-ups: "what about that?", "tell me more", "how about it?"
- User explicitly refers to previous topic with pronouns
- Question would make no sense without context

**Example:**
```
User: "What is Georgia Counts?"
User: "Tell me more about it"
→ Auto-reframe to: "Tell me more about Georgia Counts"
```

### **Tier 2: Medium Confidence → Ask for Clarification** (NEW)
When question could go either way, ask user to clarify:
- Generic action questions: "how do I log in?", "where do I find it?", "what's the deadline?"
- Could apply to multiple systems/topics
- Question is generic enough to be about something else

**Example:**
```
User: "What is Georgia Counts?"
User: "How do I log in?"
→ Bot asks: "Are you asking about logging in to Georgia Counts (which we just discussed), or another system?"
```

## Implementation Steps

### 1. **Create ambiguity detection in `src/questionReframer.js`**
Add function `detectQuestionAmbiguity()` that returns:
```javascript
{
  isAmbiguous: true/false,
  confidence: 'high' | 'medium' | 'low',
  possibleTopics: ['Georgia Counts', 'WordPress', 'general'],
  primaryTopic: 'Georgia Counts' // from conversation history
}
```

### 2. **Create clarification response generator**
New function `generateClarificationResponse()`:
```javascript
// Extract main topic from history
const mainTopic = extractMainTopicFromHistory(conversationHistory);

// Generate clarification prompt
return `Are you asking about **${mainTopic}** (which we just discussed), or another system/topic?`;
```

### 3. **Update `src/server.js` chat endpoint**
```javascript
// After detecting context-dependent question
if (shouldReframeQuestion(message, conversationHistory)) {
  const ambiguityCheck = detectQuestionAmbiguity(message, conversationHistory);

  if (ambiguityCheck.isAmbiguous && process.env.ENABLE_CLARIFICATION_MODE === 'true') {
    // Return clarification request instead of auto-reframing
    return res.json({
      answer: generateClarificationResponse(ambiguityCheck),
      requiresClarification: true,
      possibleTopics: ambiguityCheck.possibleTopics,
      sessionId: activeSessionId
    });
  }

  // Otherwise, auto-reframe as usual
  questionForRetrieval = await reframeQuestion(message, conversationHistory);
}
```

### 4. **Update frontend `public/chat.js`** (optional enhancement)
Could add interactive buttons for clarification:
```javascript
if (data.requiresClarification) {
  // Show clarification message with clickable options
  appendClarificationOptions(data.possibleTopics);
}
```

Or just let bot ask in natural language (simpler, works immediately).

### 5. **Make it configurable**
Add to `.env`:
```bash
# Enable clarification mode for ambiguous questions
# Options: true (ask for clarification) | false (auto-reframe)
ENABLE_CLARIFICATION_MODE=false
```

Default to `false` for backward compatibility.

## Example Flows

### Flow 1: Auto-reframe (High Confidence)
```
User: "What is Georgia Counts?"
Bot: [Explains Georgia Counts system]

User: "Tell me more about it"
System: High confidence - "it" clearly refers to Georgia Counts
→ Auto-reframe to: "Tell me more about Georgia Counts"
→ Search and respond
```

### Flow 2: Clarification (Medium Confidence)
```
User: "What is Georgia Counts?"
Bot: [Explains Georgia Counts system]

User: "How do I log in?"
System: Medium confidence - could be Georgia Counts or another system
→ Bot: "Are you asking about logging in to **Georgia Counts** (which we just discussed), or another system like WordPress, Canvas, or Zoom?"

User: "Georgia Counts"
→ Reframe to: "How do I log in to Georgia Counts?"
→ Search and respond with Georgia Counts login info
```

### Flow 3: New Topic (User Clarifies)
```
User: "What is Georgia Counts?"
Bot: [Explains Georgia Counts system]

User: "How do I log in?"
→ Bot: "Are you asking about logging in to **Georgia Counts**, or another system?"

User: "No, I mean WordPress"
→ Search for WordPress login instructions
→ New conversation context established
```

## Confidence Level Criteria

### High Confidence (Auto-reframe)
- Question starts with: "what about", "how about", "tell me more", "elaborate"
- Contains pronouns: "it", "that", "this", "those"
- Very short: < 10 characters
- Question makes no sense without context

### Medium Confidence (Ask for clarification)
- Generic action questions: "how do I...", "where is...", "when is..."
- No specific entity mentioned
- Could reasonably apply to multiple systems/topics
- Conversation has clear main topic

### Low Confidence (Treat as new question)
- Question contains specific entities
- Very different topic from conversation history
- Long, detailed question
- No apparent connection to previous context

## Pros and Cons

### Pros
✅ Better UX - no assumptions made
✅ More accurate responses
✅ User feels heard and in control
✅ Handles edge cases better
✅ Reduces frustration when bot guesses wrong
✅ Professional, conversational tone

### Cons
❌ Extra interaction adds friction
❌ Increases conversation length
❌ Might feel tedious for obvious follow-ups
❌ Adds implementation complexity
❌ Requires careful tuning of confidence thresholds

## Recommendation

**Implement with opt-in flag** so you can:
1. Test both approaches with real users
2. A/B test to see which performs better
3. Roll out gradually
4. Revert easily if needed

**Start with**: `ENABLE_CLARIFICATION_MODE=false` (current auto-reframe behavior)
**Test with**: `ENABLE_CLARIFICATION_MODE=true` (new clarification mode)

## Metrics to Track

If implementing, track:
- How often clarification is triggered
- User response patterns (do they clarify or abandon?)
- Accuracy improvement (fewer wrong-topic responses)
- User satisfaction (feedback ratings)
- Average conversation length change

## Alternative: Hybrid Approach

Could also implement a **confidence threshold**:
- **High confidence (>80%)**: Auto-reframe
- **Medium confidence (40-80%)**: Ask for clarification
- **Low confidence (<40%)**: Treat as new question

This gives you the best of both worlds.

## Next Steps

1. **Decision**: Do you want clarification mode?
2. **If yes**: Implement confidence scoring system
3. **Test**: Try both modes with real queries
4. **Measure**: Track metrics above
5. **Decide**: Keep, modify, or remove based on data

---

**Status**: Planning - awaiting decision on implementation
**Created**: 2025-10-01
**Related Files**:
- `src/questionReframer.js` (would need updates)
- `src/server.js` (would need updates)
- `public/chat.js` (optional updates)
