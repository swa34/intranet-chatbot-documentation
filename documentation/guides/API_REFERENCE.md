# API Reference
**Complete REST API documentation for the CAES Intranet Help Bot**

This document provides detailed information about all API endpoints, request/response formats, authentication, and example usage.

## Table of Contents

- [Base URL](#base-url)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [Response Format](#response-format)
- [Error Handling](#error-handling)
- [Endpoints](#endpoints)
  - [Chat Endpoints](#chat-endpoints)
  - [Feedback Endpoints](#feedback-endpoints)
  - [Analytics Endpoints](#analytics-endpoints)
  - [Admin Endpoints](#admin-endpoints)
  - [Session Management](#session-management)
  - [Health & Diagnostics](#health--diagnostics)

---

## Base URL

**Development:**
```
http://localhost:3000
```

**Production:**
```
https://hospitalitychatbot-r7f2j.sevalla.app
```

---

## Authentication

The API supports two authentication methods:

### 1. API Key Authentication (Programmatic Access)

Include the API key in the `x-api-key` header:

```bash
curl -H "x-api-key: your-api-key-here" \
  http://localhost:3000/chat
```

**Header:**
```
x-api-key: your-api-key-here
```

**Configuration:**
Set `CHATBOT_API_KEY` in your `.env` file.

### 2. Session-Based Authentication (Web UI)

The web interface uses cookie-based sessions. After logging in:

```bash
POST /api/auth/login
{
  "username": "user@example.com",
  "password": "password123"
}
```

A session cookie is set automatically and sent with subsequent requests.

---

## Rate Limiting

Currently, there is no rate limiting enforced. However, be mindful of:

- **OpenAI API limits**: Based on your tier
- **Pinecone quota**: Based on your plan
- **Database connections**: Limited connection pool

**Best Practices:**
- Don't make more than 10 requests/second
- Implement exponential backoff for retries
- Cache responses when possible

---

## Response Format

All responses are JSON with this general structure:

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "timestamp": "2025-10-12T10:30:00Z"
}
```

**Success Response:**
```json
{
  "answer": "Response text",
  "sources": [...],
  "responseTime": 1234,
  "cached": false
}
```

**Error Response:**
```json
{
  "error": "Error description",
  "message": "Detailed error message",
  "details": { ... }
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 200 | Success | Request completed successfully |
| 400 | Bad Request | Missing required fields, invalid data |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Endpoint doesn't exist |
| 500 | Internal Server Error | Server-side error |

### Error Response Format

```json
{
  "error": "Error type",
  "message": "Human-readable error message",
  "details": {
    "field": "Specific field that caused error",
    "reason": "Why it failed"
  }
}
```

---

## Endpoints

## Chat Endpoints

### 1. Main Chat Endpoint

**Ask a question and get an AI-generated response**

```http
POST /chat
```

**Headers:**
```
Content-Type: application/json
x-api-key: your-api-key
```

**Request Body:**
```json
{
  "message": "How do I report a presentation in GaCounts?",
  "sessionId": "optional-session-id"
}
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | ✅ Yes | User's question |
| `sessionId` | string | ❌ No | Session ID for conversation context. Auto-generated if not provided. |

**Response (200 OK):**
```json
{
  "answer": "To report a presentation in GaCounts, follow these steps:\n\n1. Log into the Georgia Counts system\n2. Navigate to 'Reports' → 'Educational Activities'\n3. Select 'Presentation' as the report type...",
  "sources": [
    {
      "url": "https://intranet.caes.uga.edu/...",
      "title": "GaCounts Reporting Guide",
      "score": 0.92,
      "sourceFile": "GaCounts_Help_Document.md"
    },
    {
      "url": "https://dropbox.com/...",
      "title": "Presentation Reporting Instructions",
      "score": 0.87,
      "sourceFile": "presentations.md"
    }
  ],
  "responseTime": 2341,
  "sessionId": "sess_abc123xyz",
  "cached": false
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `answer` | string | AI-generated response in Markdown format |
| `sources` | array | Relevant source documents (max 3) |
| `sources[].url` | string | Link to source document or page |
| `sources[].title` | string | Human-readable title |
| `sources[].score` | number | Relevance score (0-1) |
| `responseTime` | number | Total response time in milliseconds |
| `sessionId` | string | Session ID for follow-up questions |
| `cached` | boolean | Whether response came from cache |

**Example cURL:**
```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: super-secret" \
  -d '{
    "message": "How do I report a presentation in GaCounts?",
    "sessionId": "test123"
  }'
```

**Example JavaScript:**
```javascript
const response = await fetch('http://localhost:3000/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'super-secret'
  },
  body: JSON.stringify({
    message: 'How do I report a presentation in GaCounts?',
    sessionId: 'test123' // Optional
  })
});

const data = await response.json();
console.log(data.answer);
console.log(data.sources);
```

**Example Python:**
```python
import requests

response = requests.post(
    'http://localhost:3000/chat',
    headers={
        'Content-Type': 'application/json',
        'x-api-key': 'super-secret'
    },
    json={
        'message': 'How do I report a presentation in GaCounts?',
        'sessionId': 'test123'  # Optional
    }
)

data = response.json()
print(data['answer'])
```

---

### 2. Streaming Chat Endpoint

**Get real-time streamed responses using Server-Sent Events (SSE)**

```http
POST /chat/stream
```

**Headers:**
```
Content-Type: application/json
x-api-key: your-api-key
Accept: text/event-stream
```

**Request Body:**
```json
{
  "message": "Explain the GaCounts reporting process",
  "sessionId": "optional-session-id"
}
```

**Response (200 OK - Server-Sent Events):**

The server streams events in SSE format:

```
data: {"type":"start","sessionId":"sess_123"}

data: {"type":"token","content":"To report"}

data: {"type":"token","content":" in GaCounts"}

data: {"type":"sources","sources":[{"url":"...","title":"..."}]}

data: {"type":"done","responseTime":1234}
```

**Event Types:**

| Type | Description | Data |
|------|-------------|------|
| `start` | Stream started | `{type, sessionId}` |
| `token` | Text chunk | `{type, content}` |
| `sources` | Source documents | `{type, sources: [...]}` |
| `done` | Stream completed | `{type, responseTime}` |
| `error` | Error occurred | `{type, error}` |

**Example JavaScript (EventSource):**
```javascript
const eventSource = new EventSource(
  'http://localhost:3000/chat/stream?' + new URLSearchParams({
    message: 'How do I report in GaCounts?',
    apiKey: 'super-secret'
  })
);

eventSource.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'token') {
    console.log(data.content); // Print each token
  } else if (data.type === 'sources') {
    console.log('Sources:', data.sources);
  } else if (data.type === 'done') {
    eventSource.close();
  }
});
```

**Example JavaScript (fetch):**
```javascript
const response = await fetch('http://localhost:3000/chat/stream', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'super-secret',
    'Accept': 'text/event-stream'
  },
  body: JSON.stringify({
    message: 'How do I report in GaCounts?'
  })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const chunk = decoder.decode(value);
  console.log(chunk);
}
```

---

## Feedback Endpoints

### 3. Submit Feedback

**Submit user feedback on a response**

```http
POST /feedback
```

**Headers:**
```
Content-Type: application/json
x-api-key: your-api-key
```

**Request Body:**
```json
{
  "session_id": "sess_abc123",
  "messageId": "msg_12345",
  "rating": "helpful",
  "comment": "Great answer, very helpful!",
  "question": "How do I report in GaCounts?",
  "answer": "To report in GaCounts...",
  "cacheId": "cache_789"
}
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | ✅ Yes | Session ID from chat response |
| `messageId` | string | ❌ No | Message identifier |
| `rating` | string | ✅ Yes | `"helpful"` or `"not-helpful"` |
| `comment` | string | ❌ No | Optional feedback comment |
| `question` | string | ❌ No | Original question |
| `answer` | string | ❌ No | Response that was rated |
| `cacheId` | string | ❌ No | Cache ID if response was cached |

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Feedback recorded",
  "location": "PostgreSQL + Google Sheets"
}
```

**Example cURL:**
```bash
curl -X POST http://localhost:3000/feedback \
  -H "Content-Type: application/json" \
  -H "x-api-key: super-secret" \
  -d '{
    "session_id": "test123",
    "rating": "helpful",
    "comment": "Very useful information",
    "question": "How do I report in GaCounts?"
  }'
```

---

## Analytics Endpoints

### 4. Get Analytics Dashboard

**Retrieve comprehensive analytics data**

```http
GET /api/analytics
```

**Headers:**
```
x-api-key: your-api-key
```

**Response (200 OK):**
```json
{
  "success": true,
  "generated_at": "2025-10-12T10:30:00Z",
  "period": "Last 30 days",
  "overall_stats": {
    "total_sessions": 1250,
    "total_conversations": 3420,
    "total_feedback": 890,
    "helpful_count": 756,
    "not_helpful_count": 134,
    "avg_response_time_ms": 1456
  },
  "daily_summary": [
    {
      "date": "2025-10-12",
      "total_conversations": 125,
      "feedback_count": 34,
      "helpful_rate": 0.82
    }
  ],
  "top_sources": [
    {
      "source_file": "GaCounts_Help.md",
      "usage_count": 456,
      "avg_score": 0.89,
      "helpful_rate": 0.91
    }
  ],
  "problematic_sources": [
    {
      "source_file": "old_policy.md",
      "negative_feedback": 12,
      "helpful_rate": 0.23
    }
  ]
}
```

**Example cURL:**
```bash
curl http://localhost:3000/api/analytics \
  -H "x-api-key: super-secret"
```

---

### 5. Get Comment Insights

**Retrieve detailed feedback comment analysis**

```http
GET /api/comment-insights
```

**Headers:**
```
x-api-key: your-api-key
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 50 | Maximum number of results |
| `filter` | string | `all` | `all`, `comments_only`, `ratings_only` |

**Response (200 OK):**
```json
{
  "success": true,
  "generated_at": "2025-10-12T10:30:00Z",
  "period": "Last 30 days",
  "filter": "all",
  "summary": {
    "total_comments": 234,
    "analyzed_comments": 210,
    "positive_count": 156,
    "negative_count": 42,
    "neutral_count": 12,
    "thumbs_up_count": 180,
    "thumbs_down_count": 54
  },
  "top_issues": [
    {
      "issue_type": "outdated_information",
      "frequency": 23,
      "avg_rating": -0.8
    },
    {
      "issue_type": "missing_context",
      "frequency": 15,
      "avg_rating": -0.6
    }
  ],
  "recent_comments": [
    {
      "session_id": "sess_123",
      "question": "How do I...",
      "comment": "This helped a lot!",
      "sentiment": "positive",
      "issues": [],
      "created_at": "2025-10-12T09:15:00Z"
    }
  ]
}
```

**Example cURL:**
```bash
curl "http://localhost:3000/api/comment-insights?limit=100&filter=comments_only" \
  -H "x-api-key: super-secret"
```

---

### 6. Get Popular Questions

**Retrieve frequently asked questions**

```http
GET /api/popular-questions
```

**Headers:**
```
x-api-key: your-api-key
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `timeWindow` | string | `week` | `day`, `week`, `month` |
| `limit` | number | 5 | Number of questions to return |

**Response (200 OK):**
```json
{
  "topQuestions": [
    {
      "question": "How do I report a presentation in GaCounts?",
      "count": 45,
      "rank": 1
    },
    {
      "question": "What are the reporting deadlines?",
      "count": 38,
      "rank": 2
    }
  ],
  "trending": [
    {
      "question": "How do I submit a logo request?",
      "growthRate": 2.5,
      "count": 12
    }
  ],
  "analytics": {
    "totalQueries": 1234,
    "uniqueQueries": 567,
    "cacheHitRate": 0.42
  },
  "timeWindow": "week",
  "timestamp": "2025-10-12T10:30:00Z"
}
```

**Example cURL:**
```bash
curl "http://localhost:3000/api/popular-questions?timeWindow=month&limit=10" \
  -H "x-api-key: super-secret"
```

---

## Admin Endpoints

**Note**: Admin endpoints require either:
- Developer role (set via `isDeveloper` flag in session)
- Valid API key

### 7. Cache Statistics

**Get cache performance metrics**

```http
GET /admin/cache/stats
```

**Response (200 OK):**
```json
{
  "postgresql": {
    "totalEntries": 234,
    "activeEntries": 198,
    "hitRate": 0.42,
    "avgConfidence": 0.89,
    "totalHits": 1234,
    "totalMisses": 1789
  },
  "redis": {
    "entries": 156,
    "hitRate": 0.67,
    "memory": "2.4 MB"
  },
  "combined": {
    "overallHitRate": 0.58,
    "avgResponseTime": 145
  }
}
```

---

### 8. Clear Cache

**Clear cached responses**

```http
POST /admin/cache/clear
```

**Request Body:**
```json
{
  "clearAll": true
}
```

**Or clear specific cache:**
```json
{
  "cacheId": "cache_123"
}
```

**Or clear by question:**
```json
{
  "question": "How do I report in GaCounts?"
}
```

**Or clear Redis only:**
```json
{
  "clearRedis": true
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "All cache entries cleared",
  "entriesCleared": 234,
  "stats": {
    "totalEntries": 0,
    "activeEntries": 0
  }
}
```

---

### 9. List Cache Entries

**View cached responses**

```http
GET /admin/cache/list
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 20 | Results per page |
| `minConfidence` | number | 0 | Minimum confidence score |

**Response (200 OK):**
```json
{
  "entries": [
    {
      "id": "cache_123",
      "question": "How do I report in GaCounts?",
      "response": "To report in GaCounts...",
      "confidence": 0.92,
      "hitCount": 45,
      "createdAt": "2025-10-01T12:00:00Z",
      "lastHitAt": "2025-10-12T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 234,
    "pages": 12
  }
}
```

---

### 10. View Conversations

**Browse conversation history**

```http
GET /admin/conversations
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 20 | Results per page |
| `rating` | string | - | Filter by rating: `positive`, `negative` |

**Response (200 OK):**
```json
{
  "conversations": [
    {
      "id": 123,
      "session_id": "sess_abc",
      "question": "How do I report?",
      "answer": "To report...",
      "feedback_rating": 1,
      "feedback_comment": "Helpful!",
      "response_time_ms": 1456,
      "timestamp": "2025-10-12T10:30:00Z",
      "sources": [...]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 3420
  }
}
```

---

## Session Management

### 11. Clear Session

**Clear conversation history for a session**

```http
POST /chat/clear-session
```

**Request Body:**
```json
{
  "sessionId": "sess_abc123"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Session cleared"
}
```

---

### 12. Get Session Stats

**Get statistics about active sessions**

```http
GET /chat/stats
```

**Response (200 OK):**
```json
{
  "activeSessions": 45,
  "totalConversations": 3420,
  "avgTurnsPerSession": 2.7,
  "avgSessionDuration": "5m 23s"
}
```

---

### 13. Export Session

**Export conversation history for a session**

```http
GET /chat/export/:sessionId
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "sessionId": "sess_abc123",
    "startedAt": "2025-10-12T10:00:00Z",
    "turns": [
      {
        "timestamp": "2025-10-12T10:01:00Z",
        "question": "How do I report?",
        "answer": "To report...",
        "sources": [...],
        "responseTime": 1456
      }
    ]
  }
}
```

---

## Health & Diagnostics

### 14. Health Check

**Check if the server is running**

```http
GET /health
```

**No authentication required**

**Response (200 OK):**
```json
{
  "status": "ok",
  "timestamp": "2025-10-12T10:30:00Z",
  "environment": "production"
}
```

---

### 15. Feedback Status

**Check Google Sheets connection**

```http
GET /feedback/status
```

**Response (200 OK):**
```json
{
  "connected": true,
  "spreadsheetId": "abc123...",
  "lastWrite": "2025-10-12T10:25:00Z"
}
```

---

### 16. Diagnostics

**Get detailed system diagnostics**

```http
GET /feedback/diagnostics
```

**Response (200 OK):**
```json
{
  "timestamp": "2025-10-12T10:30:00Z",
  "environment": "production",
  "googleSheetsConfig": {
    "hasSpreadsheetId": true,
    "hasCredentials": true,
    "connected": true
  },
  "storageStatus": {
    "initialized": true,
    "usingFallback": false
  }
}
```

---

### 17. Developer Info

**Get current user's developer status**

```http
GET /api/developer-info
```

**Requires session authentication**

**Response (200 OK):**
```json
{
  "isDeveloper": true,
  "username": "john.doe",
  "role": "admin",
  "firstName": "John",
  "lastName": "Doe"
}
```

---

### 18. Performance Metrics

**Get performance metrics (developer only)**

```http
GET /api/developer/performance
```

**Requires developer role**

**Response (200 OK):**
```json
{
  "queries": {
    "total": 3420,
    "cached": 1434,
    "cacheHitRate": 0.42
  },
  "timing": {
    "avgRetrievalTime": 234,
    "avgGenerationTime": 1123,
    "avgTotalTime": 1456
  },
  "models": {
    "generationModel": "gpt-4o",
    "embeddingModel": "text-embedding-3-large",
    "rerankingModel": "gpt-4o-mini"
  }
}
```

---

## Common Workflows

### Multi-Turn Conversation

```javascript
// First question
const response1 = await fetch('http://localhost:3000/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'your-key'
  },
  body: JSON.stringify({
    message: 'How do I report in GaCounts?'
  })
});

const data1 = await response1.json();
const sessionId = data1.sessionId; // Save this!

// Follow-up question
const response2 = await fetch('http://localhost:3000/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'your-key'
  },
  body: JSON.stringify({
    message: 'What about collaborators?',
    sessionId: sessionId // Include session ID
  })
});

const data2 = await response2.json();
console.log(data2.answer); // Context-aware response
```

### Submit Feedback

```javascript
// After getting a response
const feedbackResponse = await fetch('http://localhost:3000/feedback', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'your-key'
  },
  body: JSON.stringify({
    session_id: sessionId,
    rating: 'helpful',
    comment: 'Very clear explanation'
  })
});
```

---

## Error Codes

| Code | Error | Cause | Solution |
|------|-------|-------|----------|
| 400 | No message provided | Missing `message` field | Include `message` in request body |
| 401 | Unauthorized | Missing/invalid API key | Check `x-api-key` header |
| 403 | Developer access required | Not a developer | Request developer access |
| 500 | Chat request failed | Server error | Check server logs |

---

## Best Practices

1. **Always save sessionId** for multi-turn conversations
2. **Include error handling** for all requests
3. **Implement retry logic** with exponential backoff
4. **Cache responses** on the client side when appropriate
5. **Submit feedback** to improve the system
6. **Use streaming** for better UX on slow connections
7. **Monitor response times** and report issues

---

## Related Documentation

- [Quick Start](QUICK_START.md) - Get started quickly
- [Development Workflow](DEVELOPMENT_WORKFLOW.md) - Development guide
- [System Overview](../architecture/SYSTEM_OVERVIEW.md) - Architecture
- [Troubleshooting](TROUBLESHOOTING.md) - Common issues

---

**Last Updated**: October 2025
