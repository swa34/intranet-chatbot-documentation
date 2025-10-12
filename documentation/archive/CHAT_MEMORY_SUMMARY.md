# Chat Memory Feature - Quick Reference

## What Was Added

✅ **Conversation Memory System** - The chatbot now remembers previous questions and answers within a session

✅ **Question Reframing** - Follow-up questions are automatically rewritten to be standalone before searching

✅ **Context-Aware Responses** - The LLM uses conversation history to provide more relevant answers

✅ **Session Management** - Sessions are tracked via unique IDs and expire after 30 minutes

## New Files

1. **`src/conversationMemory.js`** - In-memory conversation storage and session management
2. **`src/questionReframer.js`** - Intelligent question reframing using OpenAI
3. **`docs/CONVERSATION_MEMORY.md`** - Comprehensive documentation

## Modified Files

1. **`src/server.js`**
   - Added conversation memory integration
   - Modified `/chat` endpoint to handle sessions
   - Added `/chat/clear-session` and `/chat/stats` endpoints
   - Updated response generation to include history

2. **`public/chat.js`**
   - Added session ID tracking
   - Modified requests to include session ID
   - Updated clear conversation to reset sessions

## How to Use

### For Users
No changes needed! The feature works automatically:
1. Start chatting normally
2. Ask follow-up questions like "What about X?" or "Tell me more"
3. The bot will understand the context from previous messages
4. Click "Clear Conversation" to start a fresh session

### For Developers

**Test the feature:**
```bash
# Start the server
npm start

# Ask a question
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "What is Georgia Counts?"}'

# Note the sessionId in the response, use it for follow-up:
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "How do I use it?", "sessionId": "abc123..."}'
```

**Check session stats:**
```bash
curl http://localhost:3000/chat/stats \
  -H "x-api-key: your-key"
```

## Example Conversation

```
User: What is Georgia Counts?
Bot: Georgia Counts is a reporting system used by UGA Extension...

User: How do I log in?
Bot: To log in to Georgia Counts, visit https://secure.caes.uga.edu/gacounts3...

User: What if I forgot my password?
Bot: [Understands this is about Georgia Counts password]
     If you forgot your Georgia Counts password, contact...

User: What about reporting 4-H activities?
Bot: [Reframes to "How do I report 4-H activities in Georgia Counts?"]
     To report 4-H activities in Georgia Counts...
```

## Key Features

### Smart Reframing
The system detects context-dependent questions using heuristics:
- Short questions (< 15 characters)
- Questions starting with "what about", "how about", "tell me more", etc.
- Questions with pronouns like "it", "that", "this" in the first few words

### Memory Management
- **History Limit**: Keeps last 10 conversation turns
- **Session Expiry**: 30 minutes of inactivity
- **Auto Cleanup**: Expired sessions removed every 5 minutes
- **In-Memory**: Fast access, cleared on server restart

### Cost Optimization
- Only reframes when necessary (smart heuristics)
- Uses last 3-5 turns for reframing (not full history)
- Full history (10 turns) used for final response generation
- Graceful fallback if reframing fails

## Configuration

All configuration is optional with sensible defaults:

```javascript
// src/conversationMemory.js
const conversationMemory = new ConversationMemory(
  maxHistory = 10,      // Max conversation turns
  ttlMinutes = 30       // Session expiration
);
```

## Testing Checklist

- [ ] Basic question works
- [ ] Follow-up question understands context
- [ ] "Clear Conversation" resets memory
- [ ] Session expires after 30 minutes
- [ ] Multiple users have separate sessions
- [ ] Server restart clears sessions
- [ ] Reframing activates for context-dependent questions
- [ ] Debug mode shows reframing info

## Performance Impact

- **Minimal latency**: 200-500ms overhead when reframing is needed
- **Token usage**: Increases with conversation length (mitigated by turn limit)
- **Memory usage**: Negligible for typical usage (~1-5KB per session)

## Deployment Notes

✅ **No database changes required**
✅ **No new environment variables needed**
✅ **Backward compatible** - works with existing frontend
✅ **Zero downtime** - can be deployed without migration

## Troubleshooting

**Sessions not persisting?**
- Check that sessionId is being returned in response
- Verify frontend is sending sessionId in follow-up requests

**Follow-ups not understanding context?**
- Enable DEBUG_RAG=true to see if reframing is occurring
- Check logs for "Question reframed" messages

**Memory leak concerns?**
- Sessions auto-expire after 30 minutes
- Cleanup runs every 5 minutes
- Server restart clears all sessions

## Next Steps

For production deployment with persistence:
1. Replace in-memory storage with Redis
2. Add conversation export functionality
3. Implement conversation summarization for long chats
4. Add analytics for common follow-up patterns

See `docs/CONVERSATION_MEMORY.md` for detailed implementation guide.
