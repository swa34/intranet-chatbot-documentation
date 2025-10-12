# Database Schema
**PostgreSQL database structure and design**

This document describes all tables, indexes, views, and functions in the PostgreSQL database.

## Table of Contents

- [Overview](#overview)
- [Tables](#tables)
- [Indexes](#indexes)
- [Views](#views)
- [Functions](#functions)
- [Migrations](#migrations)

---

## Overview

**Database**: PostgreSQL 12+
**Connection**: Via `pg` npm package
**Pool Size**: 20 connections
**Schema**: Public (default)

---

## Tables

### 1. `conversations`
**Purpose**: Store all chat interactions and feedback

```sql
CREATE TABLE conversations (
  -- Primary key
  id SERIAL PRIMARY KEY,

  -- Session and user tracking
  session_id TEXT NOT NULL,
  username TEXT,

  -- Question and answer
  question TEXT NOT NULL,
  answer TEXT NOT NULL,

  -- Source documents (JSONB array)
  sources JSONB DEFAULT '[]'::JSONB,

  -- Feedback
  feedback_rating INTEGER CHECK (feedback_rating IN (-1, 1)),
  feedback_comment TEXT,

  -- Comment analysis
  comment_sentiment TEXT CHECK (comment_sentiment IN ('positive', 'negative', 'neutral', 'mixed')),
  comment_issues JSONB DEFAULT '[]'::JSONB,
  comment_analyzed_at TIMESTAMP,

  -- Performance metrics
  response_time_ms INTEGER,
  top_similarity_score NUMERIC(5,4),
  below_threshold BOOLEAN DEFAULT FALSE,
  was_cached BOOLEAN DEFAULT FALSE,
  cache_id TEXT,

  -- Metadata
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE
);
```

**Sample Row**:
```json
{
  "id": 123,
  "session_id": "sess_abc123",
  "username": "john.doe",
  "question": "How do I report in GaCounts?",
  "answer": "To report in GaCounts...",
  "sources": [
    {"source": "gacounts.md", "url": "https://...", "score": 0.92}
  ],
  "feedback_rating": 1,
  "feedback_comment": "Very helpful!",
  "comment_sentiment": "positive",
  "comment_issues": [],
  "response_time_ms": 1456,
  "timestamp": "2025-10-12T10:30:00Z"
}
```

---

### 2. `cached_responses`
**Purpose**: Store cached Q&A responses

```sql
CREATE TABLE cached_responses (
  -- Primary key
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,

  -- Question (original and normalized)
  question TEXT NOT NULL,
  question_normalized TEXT NOT NULL,

  -- Response and sources
  response TEXT NOT NULL,
  sources JSONB DEFAULT '[]'::JSONB,

  -- Quality metrics
  confidence NUMERIC(5,4) NOT NULL,
  hit_count INTEGER DEFAULT 0,
  positive_feedback INTEGER DEFAULT 0,
  negative_feedback INTEGER DEFAULT 0,

  -- Question variations
  variations JSONB DEFAULT '[]'::JSONB,

  -- Cache control
  is_active BOOLEAN DEFAULT TRUE,
  created_by TEXT DEFAULT 'auto',
  ttl_days INTEGER DEFAULT 30,

  -- Timestamps
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_hit_at TIMESTAMP,
  expires_at TIMESTAMP
);
```

**Normalization**: `question_normalized` is lowercase, no punctuation, single spaces

**Sample Row**:
```json
{
  "id": "cache_abc123",
  "question": "How do I report a presentation in GaCounts?",
  "question_normalized": "how do i report a presentation in gacounts",
  "response": "To report a presentation...",
  "sources": [...],
  "confidence": 0.92,
  "hit_count": 45,
  "positive_feedback": 38,
  "negative_feedback": 2,
  "variations": ["how to report presentation gacounts", "reporting presentations"],
  "is_active": true,
  "created_at": "2025-10-01T12:00:00Z",
  "last_hit_at": "2025-10-12T10:30:00Z"
}
```

---

### 3. `users`
**Purpose**: User authentication and profiles

```sql
CREATE TABLE users (
  -- Primary key
  id SERIAL PRIMARY KEY,

  -- Credentials
  username TEXT UNIQUE NOT NULL,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,

  -- Profile
  first_name TEXT,
  last_name TEXT,
  department TEXT,

  -- Role and permissions
  role TEXT DEFAULT 'user' CHECK (role IN ('user', 'admin', 'developer')),
  is_developer BOOLEAN DEFAULT FALSE,
  is_active BOOLEAN DEFAULT TRUE,

  -- Metadata
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_login TIMESTAMP,
  login_count INTEGER DEFAULT 0
);
```

**Password Hash**: Bcrypt with 10 rounds

---

### 4. `sessions`
**Purpose**: Session storage for authentication

```sql
CREATE TABLE sessions (
  -- Primary key
  sid VARCHAR PRIMARY KEY,

  -- Session data
  sess JSON NOT NULL,

  -- Expiration
  expire TIMESTAMP(6) NOT NULL
);
```

**Used by**: `express-session` with `connect-pg-simple`

---

## Indexes

### Performance Indexes

```sql
-- conversations table
CREATE INDEX idx_conversations_session_id ON conversations(session_id);
CREATE INDEX idx_conversations_timestamp ON conversations(timestamp DESC);
CREATE INDEX idx_conversations_feedback ON conversations(feedback_rating) WHERE feedback_rating IS NOT NULL;
CREATE INDEX idx_conversations_cache_id ON conversations(cache_id) WHERE cache_id IS NOT NULL;

-- cached_responses table
CREATE INDEX idx_cache_question_normalized ON cached_responses(question_normalized);
CREATE INDEX idx_cache_active ON cached_responses(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_cache_confidence ON cached_responses(confidence DESC);
CREATE INDEX idx_cache_last_hit ON cached_responses(last_hit_at DESC NULLS LAST);

-- users table
CREATE UNIQUE INDEX idx_users_username ON users(username);
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- sessions table
CREATE INDEX idx_sessions_expire ON sessions(expire);
```

---

## Views

### 1. `daily_feedback_summary`
**Purpose**: Daily aggregation of feedback

```sql
CREATE OR REPLACE VIEW daily_feedback_summary AS
SELECT
  DATE(timestamp) as date,
  COUNT(*) as total_conversations,
  COUNT(feedback_rating) as feedback_count,
  COUNT(CASE WHEN feedback_rating = 1 THEN 1 END) as helpful_count,
  COUNT(CASE WHEN feedback_rating = -1 THEN 1 END) as not_helpful_count,
  ROUND(
    COUNT(CASE WHEN feedback_rating = 1 THEN 1 END)::NUMERIC /
    NULLIF(COUNT(feedback_rating), 0),
    4
  ) as helpful_rate,
  ROUND(AVG(response_time_ms)) as avg_response_time_ms,
  ROUND(AVG(top_similarity_score), 4) as avg_similarity_score
FROM conversations
WHERE timestamp > NOW() - INTERVAL '30 days'
  AND is_active = TRUE
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

---

### 2. `top_sources`
**Purpose**: Most frequently used sources

```sql
CREATE OR REPLACE VIEW top_sources AS
SELECT
  jsonb_array_elements(sources)->>'source' as source_file,
  COUNT(*) as usage_count,
  ROUND(AVG((jsonb_array_elements(sources)->>'score')::NUMERIC), 4) as avg_score,
  COUNT(CASE WHEN feedback_rating = 1 THEN 1 END) as positive_feedback,
  COUNT(CASE WHEN feedback_rating = -1 THEN 1 END) as negative_feedback,
  ROUND(
    COUNT(CASE WHEN feedback_rating = 1 THEN 1 END)::NUMERIC /
    NULLIF(COUNT(feedback_rating), 0),
    4
  ) as helpful_rate
FROM conversations
WHERE jsonb_array_length(sources) > 0
  AND timestamp > NOW() - INTERVAL '30 days'
  AND is_active = TRUE
GROUP BY source_file
ORDER BY usage_count DESC
LIMIT 20;
```

---

### 3. `problematic_sources`
**Purpose**: Sources with high negative feedback

```sql
CREATE OR REPLACE VIEW problematic_sources AS
SELECT
  jsonb_array_elements(sources)->>'source' as source_file,
  COUNT(*) as usage_count,
  COUNT(CASE WHEN feedback_rating = -1 THEN 1 END) as negative_feedback,
  ROUND(
    COUNT(CASE WHEN feedback_rating = 1 THEN 1 END)::NUMERIC /
    NULLIF(COUNT(feedback_rating), 0),
    4
  ) as helpful_rate,
  ROUND(AVG((jsonb_array_elements(sources)->>'score')::NUMERIC), 4) as avg_score
FROM conversations
WHERE jsonb_array_length(sources) > 0
  AND feedback_rating IS NOT NULL
  AND timestamp > NOW() - INTERVAL '30 days'
  AND is_active = TRUE
GROUP BY source_file
HAVING COUNT(CASE WHEN feedback_rating = -1 THEN 1 END) > 2
ORDER BY negative_feedback DESC, helpful_rate ASC
LIMIT 10;
```

---

## Functions

### 1. `detect_comment_issues(comment TEXT)`
**Purpose**: Pattern-based issue detection in feedback comments

```sql
CREATE OR REPLACE FUNCTION detect_comment_issues(comment TEXT)
RETURNS TEXT[] AS $$
DECLARE
  issues TEXT[] := '{}';
BEGIN
  IF comment ~* 'out of date|outdated|old information' THEN
    issues := array_append(issues, 'outdated_information');
  END IF;

  IF comment ~* 'wrong|incorrect|inaccurate' THEN
    issues := array_append(issues, 'inaccurate_information');
  END IF;

  IF comment ~* 'not enough|need more|too vague|missing' THEN
    issues := array_append(issues, 'insufficient_detail');
  END IF;

  IF comment ~* 'confusing|unclear|hard to understand' THEN
    issues := array_append(issues, 'unclear_explanation');
  END IF;

  IF comment ~* 'different question|not what I asked' THEN
    issues := array_append(issues, 'answer_not_relevant');
  END IF;

  RETURN issues;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

**Usage**:
```sql
SELECT detect_comment_issues('This information is out of date');
-- Returns: {outdated_information}
```

---

### 2. `normalize_question(question TEXT)`
**Purpose**: Normalize question for cache matching

```sql
CREATE OR REPLACE FUNCTION normalize_question(question TEXT)
RETURNS TEXT AS $$
BEGIN
  RETURN LOWER(
    REGEXP_REPLACE(
      REGEXP_REPLACE(question, '[^\w\s]', '', 'g'),
      '\s+', ' ', 'g'
    )
  );
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

**Usage**:
```sql
SELECT normalize_question('How do I report in GaCounts?');
-- Returns: 'how do i report in gacounts'
```

---

## Migrations

### Running Migrations

```bash
# Run authentication migration
npm run migrate:auth
```

### Migration Files

**Location**: `src/migrations/`

#### `001_create_users_table.js`
Creates users and sessions tables for authentication.

---

## Query Examples

### Get recent conversations
```sql
SELECT
  id,
  question,
  answer,
  feedback_rating,
  response_time_ms,
  timestamp
FROM conversations
WHERE timestamp > NOW() - INTERVAL '7 days'
ORDER BY timestamp DESC
LIMIT 100;
```

### Get cache statistics
```sql
SELECT
  COUNT(*) as total_entries,
  COUNT(CASE WHEN is_active THEN 1 END) as active_entries,
  SUM(hit_count) as total_hits,
  ROUND(AVG(confidence), 4) as avg_confidence,
  ROUND(AVG(positive_feedback::NUMERIC / NULLIF(hit_count, 0)), 4) as avg_satisfaction
FROM cached_responses;
```

### Get user feedback summary
```sql
SELECT
  COUNT(*) as total_feedback,
  COUNT(CASE WHEN feedback_rating = 1 THEN 1 END) as thumbs_up,
  COUNT(CASE WHEN feedback_rating = -1 THEN 1 END) as thumbs_down,
  ROUND(
    COUNT(CASE WHEN feedback_rating = 1 THEN 1 END)::NUMERIC /
    COUNT(*)::NUMERIC,
    4
  ) as satisfaction_rate
FROM conversations
WHERE feedback_rating IS NOT NULL
  AND timestamp > NOW() - INTERVAL '30 days';
```

---

## Database Maintenance

### Cleanup old sessions
```sql
DELETE FROM sessions WHERE expire < NOW();
```

### Archive old conversations
```sql
UPDATE conversations
SET is_active = FALSE
WHERE timestamp < NOW() - INTERVAL '1 year';
```

### Vacuum and analyze
```sql
VACUUM ANALYZE conversations;
VACUUM ANALYZE cached_responses;
```

---

## Related Documentation

- [System Overview](SYSTEM_OVERVIEW.md)
- [Caching Architecture](CACHING_ARCHITECTURE.md)
- [Feedback Learning](FEEDBACK_LEARNING_POSTGRES.md)

---

**Last Updated**: October 2025
