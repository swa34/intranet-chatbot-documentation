# Development Workflow
**Daily development practices and workflows**

This guide covers common development tasks and workflows for the CAES Intranet Help Bot.

## Table of Contents

- [Daily Development](#daily-development)
- [Git Workflow](#git-workflow)
- [Testing Workflow](#testing-workflow)
- [Document Updates](#document-updates)
- [Deployment Workflow](#deployment-workflow)
- [Common Tasks](#common-tasks)

---

## Daily Development

### Morning Routine

```bash
# 1. Pull latest changes
git pull origin main

# 2. Install any new dependencies
npm install

# 3. Start development server
npm run dev

# 4. Check health
curl http://localhost:3000/health
```

### Development Cycle

```
1. Pick a task from backlog
   ↓
2. Create feature branch
   ↓
3. Make changes
   ↓
4. Test locally
   ↓
5. Commit changes
   ↓
6. Push to GitHub
   ↓
7. Create Pull Request
   ↓
8. Code Review
   ↓
9. Merge to main
```

---

## Git Workflow

### Branch Strategy

```
main (production)
  ├── feature/add-streaming
  ├── fix/cache-bug
  └── docs/update-readme
```

**Branch Types**:
- `main`: Production-ready code
- `feature/*`: New features
- `fix/*`: Bug fixes
- `docs/*`: Documentation
- `refactor/*`: Code improvements

### Creating a Feature

```bash
# 1. Ensure you're on main
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feature/add-new-feature

# 3. Make changes
# ... edit files ...

# 4. Check status
git status

# 5. Stage changes
git add src/newFeature.js
git add tests/newFeature.test.js

# 6. Commit with descriptive message
git commit -m "feat: add new feature for X

- Implement core functionality
- Add tests
- Update documentation"

# 7. Push to GitHub
git push origin feature/add-new-feature

# 8. Create Pull Request on GitHub
```

### Keeping Branch Updated

```bash
# Pull latest main
git checkout main
git pull origin main

# Merge into your branch
git checkout feature/add-new-feature
git merge main

# Or rebase (preferred for clean history)
git rebase main

# If conflicts, resolve them
git add .
git rebase --continue
```

---

## Testing Workflow

### Manual Testing Checklist

Before each commit:

```bash
# 1. Start server
npm run dev

# 2. Test main endpoint
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "test message"}'

# 3. Check response format
# - Has answer field
# - Has sources array
# - Has responseTime
# - Has sessionId

# 4. Test error cases
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "test"}' # Missing API key

# 5. Test caching
# Send same question twice, second should be faster

# 6. Check logs for errors
# Review console output
```

### Performance Testing

```bash
# Test response time
time curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "How do I report in GaCounts?"}'

# Should complete in <2s
```

### Load Testing (Optional)

```bash
# Install artillery
npm install -g artillery

# Create test config
cat > load-test.yml <<EOF
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - flow:
      - post:
          url: "/chat"
          headers:
            x-api-key: "your-key"
          json:
            message: "test"
EOF

# Run load test
artillery run load-test.yml
```

---

## Document Updates

### Adding New Documents

```bash
# 1. Add documents to appropriate folder
cp new-document.pdf docs/dropbox/

# 2. Process if needed (PDF/Office files)
cd python
python document_processor.py "../docs/dropbox/new-document.pdf"

# 3. Ingest into Pinecone
cd ..
npm run ingest:dropbox

# 4. Test search
curl "http://localhost:3000/debug-query?q=new%20document" \
  -H "x-api-key: your-key"
```

### Updating Existing Documents

```bash
# 1. Replace file in docs folder
cp updated-document.md docs/dropbox/document.md

# 2. Re-ingest (will overwrite existing vectors)
npm run ingest:dropbox

# 3. Clear cache (important!)
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: your-key" \
  -H "Content-Type: application/json" \
  -d '{"clearAll": true}'
```

### Removing Documents

```bash
# 1. Delete file
rm docs/dropbox/old-document.md

# 2. Remove from Pinecone
node src/rag/vector-ops/deleteDocument.js old-document.md

# 3. Clear cache
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: your-key" \
  -d '{"clearAll": true}'
```

---

## Deployment Workflow

### Pre-Deployment Checklist

- [ ] All tests pass
- [ ] No console.log statements
- [ ] Environment variables documented
- [ ] Database migrations tested
- [ ] Performance tested
- [ ] Documentation updated

### Deploying to Sevalla

```bash
# 1. Ensure code is on main branch
git checkout main
git pull origin main

# 2. Build (if needed)
# No build step for this project

# 3. Push to Sevalla
git push sevalla main

# 4. Run migrations (if needed)
# SSH into Sevalla server
npm run migrate:auth

# 5. Restart server
# Sevalla auto-restarts on git push

# 6. Verify deployment
curl https://your-sevalla-app.sevalla.app/health

# 7. Test main functionality
curl -X POST https://your-sevalla-app.sevalla.app/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "test"}'
```

### Rollback Procedure

```bash
# 1. Identify last good commit
git log --oneline

# 2. Revert to that commit
git checkout <commit-hash>

# 3. Force push (be careful!)
git push sevalla HEAD:main --force

# 4. Verify rollback worked
curl https://your-sevalla-app.sevalla.app/health
```

---

## Common Tasks

### Task: Add a New API Endpoint

```javascript
// 1. Add route in src/server.js
app.get('/api/new-endpoint', requireApiKey, async (req, res) => {
  try {
    const result = await yourLogic();
    res.json({ success: true, data: result });
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 2. Test it
curl http://localhost:3000/api/new-endpoint \
  -H "x-api-key: your-key"

// 3. Document in API_REFERENCE.md

// 4. Commit
git add src/server.js
git add documentation/guides/API_REFERENCE.md
git commit -m "feat(api): add new endpoint"
```

### Task: Improve Cache Hit Rate

```javascript
// 1. Check current stats
curl http://localhost:3000/admin/cache/stats \
  -H "x-api-key: your-key"

// 2. Identify popular questions
curl http://localhost:3000/api/popular-questions \
  -H "x-api-key: your-key"

// 3. Pre-cache popular questions
node src/rag/cache/generateFAQCache.js

// 4. Verify improvement
# Wait 1 hour, check stats again
```

### Task: Debug Slow Queries

```bash
# 1. Enable debug mode
DEBUG_RAG=true npm run dev

# 2. Send problematic query
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "slow query"}'

# 3. Check logs for timing breakdown
# Look for:
# - ⏱️ Retrieval Performance
# - 📊 Performance Breakdown

# 4. Identify bottleneck
# - High embedding time? Check OpenAI status
# - High Pinecone time? Reduce topK
# - High generation time? Use faster model
```

### Task: Update System Prompt

```bash
# 1. Edit prompt
nano src/prompts/systemPrompt.txt

# 2. Restart server (prompt loaded at startup)
# Ctrl+C, then npm run dev

# 3. Test new behavior
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "test"}'

# 4. Clear cache to apply to all responses
curl -X POST http://localhost:3000/admin/cache/clear \
  -H "x-api-key: your-key" \
  -d '{"clearAll": true}'
```

---

## Environment Management

### Development

```env
NODE_ENV=development
DEBUG_RAG=true
PORT=3000
ENABLE_CLARIFICATION_MODE=true
```

### Production

```env
NODE_ENV=production
DEBUG_RAG=false
PORT=3000
ENABLE_CLARIFICATION_MODE=true
```

### Switching Environments

```bash
# Local → Staging
export NODE_ENV=staging
npm run dev

# Staging → Production
export NODE_ENV=production
npm start
```

---

## Monitoring in Development

### Check Server Health

```bash
# Health check
curl http://localhost:3000/health

# Performance metrics (developer only)
curl http://localhost:3000/api/developer/performance \
  -H "x-api-key: your-key"

# Cache stats
curl http://localhost:3000/admin/cache/stats \
  -H "x-api-key: your-key"
```

### Monitor Logs

```bash
# Watch logs in real-time
tail -f logs/app.log

# Search logs
grep ERROR logs/app.log
```

---

## Tips and Best Practices

1. **Commit often**: Small, focused commits
2. **Test before committing**: Always test your changes
3. **Write clear commit messages**: Future you will thank you
4. **Keep branches short-lived**: Merge within 2-3 days
5. **Update documentation**: Document as you code
6. **Ask for help**: Don't struggle alone

---

## Related Documentation

- [Quick Start](QUICK_START.md)
- [Contributing Guide](CONTRIBUTING.md)
- [API Reference](API_REFERENCE.md)
- [Troubleshooting](TROUBLESHOOTING.md)

---

**Last Updated**: October 2025
