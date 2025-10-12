# Clarification Mode Usage Guide

## Overview

Clarification Mode is an opt-in feature that makes the chatbot ask for clarification when users ask ambiguous follow-up questions, instead of automatically assuming what they mean.

## Configuration

Add to your `.env` file:

```bash
# Enable clarification mode for ambiguous follow-up questions
# Options: true (ask for clarification) | false (auto-reframe, default)
ENABLE_CLARIFICATION_MODE=false
```

### Default Behavior (ENABLE_CLARIFICATION_MODE=false or not set)

The chatbot **automatically reframes** ambiguous questions based on conversation context:

```
User: "What is Georgia Counts?"
Bot: [Explains Georgia Counts]

User: "How do I log in?"
→ System auto-reframes to: "How do I log in to Georgia Counts?"
→ Searches and responds with Georgia Counts login info
```

### Clarification Mode (ENABLE_CLARIFICATION_MODE=true)

The chatbot **asks for clarification** on ambiguous questions:

```
User: "What is Georgia Counts?"
Bot: [Explains Georgia Counts]

User: "How do I log in?"
→ Bot: "I want to make sure I understand your question correctly.
       Are you asking about **Georgia Counts** (which we just discussed),
       or something else?

       You can reply with:
       - 'Georgia Counts' to continue that topic
       - The name of another system/topic you're asking about"

User: "Georgia Counts"
→ Bot: [Provides Georgia Counts login instructions]
```

## When Clarification is Triggered

Clarification mode only activates for **medium-confidence** ambiguous questions:

### ✅ **Triggers Clarification** (Medium Confidence)
- Generic action questions without specific entities:
  - "How do I log in?"
  - "Where do I find it?"
  - "What's the deadline?"
  - "Can I do that?"

### ❌ **No Clarification** (High Confidence - obvious follow-ups)
- Questions with pronouns clearly referring to previous context:
  - "Tell me more about it"
  - "What about that?"
  - "Explain this"

### ❌ **No Clarification** (High Confidence - specific entities mentioned)
- Questions that include specific system names:
  - "How do I log in to Georgia Counts?"
  - "Where is the WordPress dashboard?"
  - "Can I use Canvas for this?"

### ❌ **No Clarification** (Low Confidence - likely new topic)
- Long, detailed questions about new topics
- Questions with no apparent connection to conversation history

## Confidence Scoring System

The system uses a three-tier confidence approach:

| Confidence | Behavior | Example |
|------------|----------|---------|
| **High** | Auto-reframe (no clarification) | "Tell me more", "What about it?" |
| **Medium** | Ask for clarification (if mode enabled) | "How do I log in?" |
| **Low** | Treat as new question | Completely different topic |

## Testing

### Test Auto-Reframe Mode (Default)

1. Ensure `.env` has `ENABLE_CLARIFICATION_MODE=false` (or omit the variable)
2. Restart server: `npm start`
3. Check console: `Clarification mode: DISABLED`
4. Test conversation:
   ```
   User: "What is Georgia Counts?"
   User: "How do I log in?"  # Should auto-reframe
   ```

### Test Clarification Mode

1. Set in `.env`: `ENABLE_CLARIFICATION_MODE=true`
2. Restart server: `npm start`
3. Check console: `Clarification mode: ENABLED`
4. Test conversation:
   ```
   User: "What is Georgia Counts?"
   User: "How do I log in?"  # Should ask for clarification
   User: "Georgia Counts"     # Should provide Georgia Counts login info
   ```

## Console Logging

When clarification is triggered, you'll see:

```
Ambiguous question detected: {
  question: 'how do I log in?',
  confidence: 'medium',
  reason: 'generic_action_without_entity',
  mainTopic: 'Georgia Counts'
}
```

When auto-reframing occurs:

```
Question reframed: {
  original: 'how do I log in?',
  reframed: 'How do I log in to Georgia Counts?'
}
```

## API Response Differences

### Auto-Reframe Mode Response

```json
{
  "answer": "To log in to Georgia Counts, visit...",
  "sources": [...],
  "responseTime": 1234,
  "sessionId": "abc123"
}
```

### Clarification Mode Response

```json
{
  "answer": "I want to make sure I understand...",
  "requiresClarification": true,
  "mainTopic": "Georgia Counts",
  "originalQuestion": "how do I log in?",
  "responseTime": 123,
  "sessionId": "abc123"
}
```

The frontend can detect `requiresClarification: true` and optionally provide clickable options.

## Pros and Cons

### Auto-Reframe Mode (Default)

**Pros:**
- ✅ Faster - one less interaction
- ✅ More conversational flow
- ✅ Works well when assumption is correct

**Cons:**
- ❌ Can guess wrong
- ❌ User has to ask again if wrong
- ❌ Less transparent

### Clarification Mode

**Pros:**
- ✅ More accurate - no guessing
- ✅ Better UX when topics could be ambiguous
- ✅ Professional, careful approach
- ✅ User feels heard

**Cons:**
- ❌ Extra interaction adds friction
- ❌ Might feel tedious for obvious questions
- ❌ Increases conversation length

## Recommendation

**Start with Auto-Reframe Mode** (default) for most use cases.

**Enable Clarification Mode** if:
- Your chatbot covers many different systems/topics
- Ambiguity is common in your use case
- User feedback indicates bot is guessing wrong too often
- Professional/formal tone is preferred over conversational

## Customization

You can customize the detection logic in `src/questionReframer.js`:

### Add More Known Topics

```javascript
const knownTopics = [
  { pattern: /georgia counts?|gacounts?/i, name: 'Georgia Counts' },
  { pattern: /\b4-h\b/i, name: '4-H' },
  // Add your own:
  { pattern: /my custom system/i, name: 'Custom System' },
];
```

### Adjust Generic Action Patterns

```javascript
const genericActionPatterns = [
  /^how (do|can|should) (i|we|you)/,
  /^(what|where|when) (is|are|do|does)/,
  // Add your own:
  /^tell me about/,
];
```

### Customize Clarification Message

Edit `generateClarificationResponse()` in `src/questionReframer.js` to change the clarification message format.

## Monitoring

Check stats:

```bash
curl http://localhost:3000/chat/stats -H "x-api-key: your-key"
```

If running in DEBUG_RAG mode, responses include:

```json
{
  "debug": {
    "totalMatches": 8,
    "topScore": 0.82,
    "conversationTurns": 3,
    "questionReframed": true
  }
}
```

## Troubleshooting

**Clarification not triggering:**
- Check `.env` has `ENABLE_CLARIFICATION_MODE=true`
- Restart server after changing `.env`
- Check console shows "Clarification mode: ENABLED"
- Question must be ambiguous (generic without specific entity)

**Too many clarifications:**
- Lower the ambiguity threshold by adding more specific entity patterns
- Adjust generic action patterns to be less broad

**Not enough clarifications:**
- Expand generic action patterns
- Remove some specific entity patterns

## Future Enhancements

Potential improvements:
- Interactive buttons on frontend for clarification options
- Learn from user corrections to improve detection
- Context-aware entity recognition
- Multi-topic conversation handling
