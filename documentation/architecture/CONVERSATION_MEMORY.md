# Conversation Memory Feature

## Overview

The conversation memory system enables the chatbot to remember previous questions and answers within a session, allowing natural follow-up questions. Conversations are persisted in PostgreSQL, surviving server restarts and enabling analytics.

## Architecture

### Storage: PostgreSQL-Backed Persistence

**Benefits:**
- ✅ Conversations survive restarts
- ✅ Multi-instance ready (shared DB)
- ✅ Analytics via SQL queries
- ✅ Session export available
- ✅ Completely anonymous (no user data required)

**What We Store:**
- Anonymous session ID (random string)
- Question and answer text
- Source documents used
- Response times
- Optional feedback ratings

**What We DON'T Store:**
- No user names, emails, or MyID
- No IP addresses, cookies, or browser fingerprints
- No personally identifiable information (PII)

## Database Schema

### Tables

**conversations**
```sql
- id (serial primary key)
- session_id (varchar 32) -- anonymous session ID
- question (text)
- answer (text)
- sources (jsonb) -- array of source documents
- response_time_ms (integer)
- feedback_rating (smallint) -- -1, 0, 1
- feedback_comment (text)
- timestamp (timestamp)
```

**conversation_sessions**
```sql
- session_id (varchar 32 primary key)
- created_at (timestamp)
- last_accessed_at (timestamp)
- total_turns (integer)
- is_active (boolean)
```

**pending_clarifications**
```sql
- session_id (varchar 32 unique)
- original_question (text)
- main_topic (varchar 255)
- resolved (boolean)
- timestamp (timestamp)
```

### Views

**session_analytics**
```sql
SELECT
  DATE(timestamp) as date,
  COUNT(DISTINCT session_id) as unique_sessions,
  COUNT(*) as total_questions,
  AVG(response_time_ms) as avg_response_time,
  COUNT(CASE WHEN feedback_rating = 1...) as positive_feedback
FROM conversations
GROUP BY DATE(timestamp);
```

## How It Works

### 1. Session Management (`src/conversationMemoryPostgres.js`)

- Each user gets a unique session ID tracked in PostgreSQL
- Session history persisted in the database
- Sessions expire after 30 minutes of inactivity (configurable)
- Maximum of 10 conversation turns retained per session (configurable)
- Automatic cleanup of old sessions via database function

### 2. Question Reframing (`src/questionReframer.js`)

When a user asks a follow-up question that depends on context (e.g., "What about the drawbacks?"), the system:

1. Detects that the question is context-dependent using heuristics
2. Uses the LLM to rewrite the question as a standalone question
3. Example:
   - **Original:** "What about the drawbacks?"
   - **Reframed:** "What are the main drawbacks of using a vector database?"

### 3. RAG Integration

- The reframed question is used to query Pinecone for relevant documents
- This ensures accurate retrieval even for follow-up questions
- Original question is used for response generation to maintain natural conversation flow

### 4. Response Generation

- The LLM receives both:
  - The retrieved context from Pinecone
  - The conversation history (last 3-5 turns)
- This allows for context-aware, conversational responses

## API Endpoints

### `POST /chat`

**Request:**
```json
{
  "message": "What about the drawbacks?",
  "sessionId": "optional-existing-session-id"
}
```

**Response:**
```json
{
  "answer": "In addition to the benefits we discussed...",
  "sources": [...],
  "responseTime": 1234,
  "sessionId": "abc123def456"
}
```

### `POST /chat/clear-session`

Clear a conversation session (marks as inactive in database).

**Request:**
```json
{
  "sessionId": "abc123def456"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Session cleared"
}
```

### `GET /chat/stats`

Get statistics about active sessions (requires API key).

**Response:**
```json
{
  "activeSessions": 5,
  "totalTurns": 42
}
```

### `GET /chat/export/:sessionId`

Export complete conversation history for a session.

**Request:**
```http
GET /chat/export/:sessionId
Authorization: Bearer YOUR_API_KEY
```

**Response:**
```json
{
  "success": true,
  "data": {
    "sessionId": "a3f4b2c1...",
    "sessionInfo": {
      "created_at": "2025-10-04T...",
      "last_accessed_at": "2025-10-04T...",
      "total_turns": 5
    },
    "conversations": [
      {
        "question": "How do I log in to GaCounts?",
        "answer": "For login assistance...",
        "timestamp": "2025-10-04T...",
        "sources": [...],
        "feedback": {
          "rating": 1,
          "comment": "Very helpful!"
        }
      }
    ]
  }
}
```

## Frontend Integration

The frontend (`public/chat.js`) automatically:
1. Receives and stores the session ID from the first response
2. Includes the session ID in all subsequent requests
3. Clears the session when the user clicks "Clear Conversation"

## Configuration

### Environment Variables

Required for PostgreSQL connection (configured on Sevalla):

```env
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=your_database
DB_USERNAME=your_user
DB_PASSWORD=your_password
```

### Adjustable Parameters

In `src/conversationMemoryPostgres.js`:
- `maxHistory`: Maximum number of turns to store (default: 10)
- `ttlMinutes`: Session expiration time (default: 30 minutes)

In `src/questionReframer.js`:
- Adjust heuristics for detecting context-dependent questions
- Modify the reframing prompt for different behavior

## Use Cases

### Example 1: Follow-up Questions

```
User: What is Georgia Counts?
Bot: Georgia Counts (GaCounts) is the reporting system...

User: How do I log in?
Bot: To log in to Georgia Counts, visit...

User: What if I forgot my password?
Bot: [Understands "my password" refers to Georgia Counts password]
```

### Example 2: Clarification Questions

```
User: How do I report 4-H activities?
Bot: To report 4-H activities in Georgia Counts...

User: Can you give more details about that?
Bot: [Provides additional details about 4-H reporting]
```

### Example 3: Comparative Questions

```
User: Tell me about Master Gardener reporting
Bot: Master Gardener activities are reported...

User: How is that different from 4-H reporting?
Bot: [Compares both types of reporting]
```

## Analytics Queries

### Daily Active Sessions
```sql
SELECT
  DATE(timestamp) as date,
  COUNT(DISTINCT session_id) as daily_active_sessions
FROM conversations
WHERE timestamp >= NOW() - INTERVAL '30 days'
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

### Most Common Questions
```sql
SELECT
  question,
  COUNT(*) as frequency
FROM conversations
WHERE timestamp >= NOW() - INTERVAL '7 days'
GROUP BY question
ORDER BY frequency DESC
LIMIT 20;
```

### Average Response Quality
```sql
SELECT
  DATE(timestamp) as date,
  AVG(CASE WHEN feedback_rating = 1 THEN 1 ELSE 0 END) * 100 as positive_rate,
  AVG(CASE WHEN feedback_rating = -1 THEN 1 ELSE 0 END) * 100 as negative_rate
FROM conversations
WHERE feedback_rating IS NOT NULL
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

### Conversation Length Distribution
```sql
SELECT
  total_turns,
  COUNT(*) as session_count
FROM conversation_sessions
WHERE is_active = TRUE
GROUP BY total_turns
ORDER BY total_turns;
```

## Deployment

### 1. Run Database Schema

On Sevalla PostgreSQL:
```bash
psql -h DB_HOST -U DB_USERNAME -d DB_DATABASE -f database/schema/conversations.sql
```

### 2. Verify Deployment

```bash
# Check stats endpoint
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://your-app.sevalla.app/chat/stats

# Export a session
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://your-app.sevalla.app/chat/export/SESSION_ID
```

## Performance Considerations

### Token Usage
- Each conversation turn increases token usage since history is sent to the LLM
- History is limited to last 3-5 turns for reframing to manage costs
- Full history (up to 10 turns) is provided for response generation

### Database Performance
- Indexed on session_id and timestamp for fast queries
- Periodic cleanup via `cleanup_old_sessions()` function
- Monitor database size with pg_size_pretty queries

### Latency
- Question reframing adds one additional LLM call
- Smart heuristics minimize unnecessary reframing
- Typical overhead: 200-500ms for reframing when needed
- Database queries add <50ms overhead vs in-memory

## Debugging

### Enable Debug Mode

Set `DEBUG_RAG=true` in `.env` to see:
- Whether questions were reframed
- Conversation turn count
- Session activity
- Database query performance

### Check Session Stats

```bash
curl -X GET https://your-app.sevalla.app/chat/stats \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Monitor Database

```sql
SELECT
  pg_size_pretty(pg_total_relation_size('conversations')) as conversations_size,
  pg_size_pretty(pg_total_relation_size('conversation_sessions')) as sessions_size;
```

## Maintenance

### Cleanup Old Sessions

Run periodically:
```sql
SELECT cleanup_old_sessions();
-- Returns: number of sessions marked inactive
```

Or via cron:
```bash
# Daily cleanup at 2am
0 2 * * * psql -c "SELECT cleanup_old_sessions();"
```

## Future Enhancements

### 1. Conversation Summarization
```sql
-- Add summary column
ALTER TABLE conversation_sessions
ADD COLUMN conversation_summary TEXT;

-- Update with LLM-generated summary
UPDATE conversation_sessions
SET conversation_summary = summarize_conversation(session_id)
WHERE total_turns > 5;
```

### 2. User Authentication (Optional)
```sql
-- Add user tracking
ALTER TABLE conversation_sessions
ADD COLUMN user_id VARCHAR(50); -- MyID if authenticated
```

### 3. Full-text Search
```sql
-- Create search index
CREATE INDEX idx_conversations_search
ON conversations
USING gin(to_tsvector('english', question || ' ' || answer));

-- Search query
SELECT * FROM conversations
WHERE to_tsvector('english', question || ' ' || answer)
  @@ to_tsquery('english', 'gacounts & login');
```

## Technical Notes

### Session ID Generation
- Uses crypto.randomBytes(16) for secure, unique IDs
- IDs are 32-character hex strings

### Error Handling
- If reframing fails, falls back to original question
- Missing session IDs are handled gracefully (new session created)
- Database connection errors logged and handled

### Backward Compatibility
- Existing non-session requests still work
- Frontend gracefully handles missing session support

## Support

For issues:
- Check Sevalla logs for errors
- Verify DB connection with `GET /chat/stats`
- Check PostgreSQL logs for query errors
- Ensure schema was run successfully

---

**Implementation:** `src/conversationMemoryPostgres.js`
**Database:** PostgreSQL (Sevalla)
**Status:** Production
