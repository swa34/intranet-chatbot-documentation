# Feedback-Driven Learning System

## Overview
The chatbot uses user feedback (thumbs up/down) and comment analysis to continuously improve its responses by intelligently adjusting the ranking of document sources based on historical performance and issue detection.

## Architecture

### Data Flow
1. **User Feedback** → Google Sheets (persistent storage)
2. **Google Sheets** → Feedback Analyzer (periodic processing)
3. **Learning Cache** → Search Reranking (real-time application)

### Storage Locations
- **Production (Sevalla)**: Google Sheets API for feedback storage
- **Local Development**: Can use both Google Sheets and local files
- **Learning Cache**: `feedback/learning_cache.json` (committed to repo)

## Core Components

### 1. Feedback Collection & Storage
- User clicks thumbs up (helpful) or thumbs down (not helpful)
- Optional comments provide context about issues
- Data saved to Google Sheets via API
- Includes question, answer, sources used, rating, and comments
- Works on read-only filesystems (Sevalla compatible)

### 2. Intelligent Comment Analysis

#### Smart Scoring System
The system now analyzes comments to detect "helpful but has issues" scenarios:

**Regex-Based Detection (Fast & Free)**
- "broken link" → Weight: 0.7 (minor issue)
- "wrong information" → Weight: 0.3 (major issue)
- "thanks!" → Weight: 1.0 (truly helpful)

**LLM Analysis (For Ambiguous Cases)**
- Comments like "helpful but..." are analyzed with GPT-4o-mini
- Only ~20% of comments need LLM analysis
- Results are cached to avoid re-analysis

#### Issue Tracking
```javascript
{
  "sourceScores": {
    "docs/example.md": {
      "helpful": 8.5,           // Weighted score
      "notHelpful": 2,
      "helpfulWithIssues": 3,   // Tracked separately
      "total": 14,
      "score": 0.46,
      "issueTypes": {
        "broken_link": 2,
        "partial_answer": 1
      }
    }
  }
}
```

### 3. Learning Process

#### Score Calculation
```javascript
// Base scoring with comment adjustment
if (rating === 'helpful') {
  if (hasIssues) {
    helpful += weight; // 0.3-0.7 based on severity
  } else {
    helpful += 1.0;
  }
}

score = (helpful - (notHelpful * 2)) / total
```

#### Real-Time Adjustment
- Documents with positive feedback boosted up to 20%
- Documents with issues reduced proportionally
- Query patterns only include truly helpful responses (weight > 0.5)

## Usage

### Updating the Learning Cache

#### From Google Sheets (Production)
```bash
# Analyzes all feedback from Google Sheets
node src/rag/feedbackLearningSheets.js
```

Output shows:
- Sources analyzed
- Patterns identified
- LLM calls made (for ambiguous comments)
- Top/low performing sources

#### Commit and Deploy
```bash
# After running analysis
git add feedback/learning_cache.json
git commit -m "Update feedback learning cache"
git push
```

### Generating Development Insights
```bash
# Extract actionable tasks from comments
node src/rag/commentAnalyzer.js
```

Creates reports in `feedback/insights/`:
- `actionable_tasks.md` - Development todo list
- `feedback_report.md` - Detailed analysis
- `insights.json` - Raw data for automation

### MCP Server for Claude Desktop

#### Configuration
Add to `%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "feedback-analyzer": {
      "command": "node",
      "args": ["C:\\path\\to\\project\\mcp-final.js"],
      "cwd": "C:\\path\\to\\project",
      "env": {
        "GOOGLE_SHEETS_ID": "your-sheet-id",
        "GOOGLE_SERVICE_ACCOUNT_EMAIL": "your-service-account",
        "GOOGLE_PRIVATE_KEY": "your-private-key"
      }
    }
  }
}
```

#### Available Tools
- `get_feedback_metrics` - Analyze feedback and show source performance

## Environment Configuration

### Required Environment Variables
```env
# Google Sheets API (for feedback storage)
GOOGLE_SHEETS_ID=your-spreadsheet-id
GOOGLE_SERVICE_ACCOUNT_EMAIL=service-account@project.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# OpenAI API (for comment analysis - optional)
OPENAI_API_KEY=your-api-key
```

### Google Sheets Setup
1. Create a Google Sheet with columns:
   - timestamp, messageId, question, answer, rating, comments, sources, userAgent, ip, authenticatedUser
2. Share the sheet with your service account email
3. Use the sheet ID in your environment variables

## Monitoring & Maintenance

### Check Learning Status
```bash
# View top performing sources
cat feedback/learning_cache.json | jq '.sourceScores | to_entries | sort_by(.value.score) | reverse | .[0:5]'

# Count feedback entries in Google Sheets
node -e "import('./src/rag/feedbackLearningSheets.js').then(async m => {
  const f = new m.default();
  const r = await f.analyzeFeedback();
  console.log('Total feedback:', r.totalFeedback);
})"
```

### Best Practices

1. **Regular Updates**: Run `feedbackLearningSheets.js` weekly
2. **Review Issues**: Check sources with high `helpfulWithIssues` counts
3. **Fix Broken Links**: Priority fix for `broken_link` issues
4. **Monitor API Usage**: Track LLM calls if using comment analysis
5. **Never Clear Google Sheets**: Historical data improves accuracy

## Technical Architecture

### File Structure
```
src/
  rag/
    feedbackLearning.js        # Core learning logic (uses cache)
    feedbackLearningSheets.js  # Google Sheets analyzer
    commentAnalyzer.js         # Development insights generator
    commentScorer.js           # Smart comment scoring
  googleSheetsStorage.js       # Google Sheets API wrapper
  server.js                    # Main server (saves feedback)
feedback/
  learning_cache.json          # Generated cache (commit this)
  insights/                    # Development reports (git ignored)
mcp-final.js                   # MCP server for Claude Desktop
```

### How Ranking Works

1. **Base Retrieval**: Pinecone returns initial matches
2. **Score Adjustment**: `feedbackLearning.adjustRetrievalScores()`
   - Loads learning cache
   - Adjusts scores based on source history
   - Boosts pattern-matched sources
3. **Reranking**: Results reordered by adjusted scores
4. **Response Generation**: Top results used for answer

## Benefits

### Intelligent Feedback Processing
- **Nuanced Understanding**: Distinguishes "helpful but broken" from "truly helpful"
- **Cost Optimization**: Hybrid regex/LLM approach minimizes API costs
- **Issue Detection**: Automatically identifies broken links, wrong info, partial answers

### Continuous Improvement
- **Self-Improving**: Gets smarter with each feedback
- **Pattern Learning**: Identifies successful query-source combinations
- **Quality Control**: Surfaces problematic content needing updates

### Production Ready
- **Sevalla Compatible**: Uses Google Sheets, no local file writes
- **Scalable**: Handles thousands of feedback entries
- **Resilient**: Fallbacks for API failures

## Future Enhancements

Potential improvements:
- [ ] Real-time cache updates via webhooks
- [ ] Sentiment analysis for longer comments
- [ ] User-specific personalization
- [ ] A/B testing different weight adjustments
- [ ] Automatic issue ticket creation
- [ ] Dashboard for feedback metrics