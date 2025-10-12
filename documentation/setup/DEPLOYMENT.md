# Deployment Guide
**Deploying the CAES Intranet Help Bot to production**

This guide covers deploying the chatbot to Sevalla and other production environments.

## Table of Contents

- [Pre-Deployment Checklist](#pre-deployment-checklist)
- [Sevalla Deployment](#sevalla-deployment)
- [Environment Variables](#environment-variables)
- [Database Setup](#database-setup)
- [Post-Deployment](#post-deployment)
- [Monitoring](#monitoring)
- [Rollback Procedure](#rollback-procedure)

---

## Pre-Deployment Checklist

Before deploying to production:

- [ ] All tests pass locally
- [ ] Code reviewed and approved
- [ ] No console.log statements in production code
- [ ] Environment variables documented
- [ ] Database migrations tested
- [ ] Performance tested (load testing)
- [ ] Security review completed
- [ ] Documentation updated
- [ ] Backup plan in place

---

## Sevalla Deployment

Sevalla is the recommended hosting platform for Node.js applications.

### Initial Setup

1. **Create Sevalla Account**
   - Go to https://sevalla.com/
   - Create account
   - Create new Node.js application

2. **Connect Git Repository**
   - Link your GitHub repository
   - Select branch to deploy (usually `main`)
   - Sevalla will auto-deploy on push

3. **Configure Build Settings**
   ```yaml
   # No build step required for this project
   # Sevalla will run: npm install
   ```

4. **Set Start Command**
   ```bash
   npm start
   # Or: node src/server.js
   ```

### Environment Variables

Configure in Sevalla dashboard:

```env
# Required
NODE_ENV=production
OPENAI_API_KEY=sk-...
PINECONE_API_KEY=...
PINECONE_INDEX_NAME=uga-intranet-index
CHATBOT_API_KEY=...
DATABASE_URL=postgresql://...
PORT=3000

# Optional but recommended
REDIS_URL=redis://...
ENABLE_RESPONSE_CACHE=true
ENABLE_LLM_RERANKING=true
MIN_SIMILARITY=0.75
GEN_MODEL=gpt-4o
EMBED_MODEL=text-embedding-3-large

# Sevalla specific
SEVALLA_DOMAIN=your-app.sevalla.app
```

### Deploy

```bash
# Commit your changes
git add .
git commit -m "feat: ready for production"

# Push to main branch
git push origin main

# Sevalla auto-deploys on push
# Monitor deployment in Sevalla dashboard
```

---

## Database Setup

### PostgreSQL on Sevalla

1. **Provision PostgreSQL**
   - In Sevalla dashboard, add PostgreSQL addon
   - Copy connection URL

2. **Set DATABASE_URL**
   ```env
   DATABASE_URL=postgresql://user:pass@host:5432/dbname
   ```

3. **Run Migrations**
   ```bash
   # SSH into Sevalla
   sevalla run bash

   # Run migrations
   npm run migrate:auth
   ```

### External PostgreSQL (e.g., Supabase)

1. **Create Database**
   - Sign up at https://supabase.com/
   - Create new project
   - Copy connection string

2. **Configure**
   ```env
   DATABASE_URL=postgresql://postgres:[password]@db.[project].supabase.co:5432/postgres
   ```

3. **Test Connection**
   ```bash
   psql $DATABASE_URL -c "SELECT NOW()"
   ```

---

## Redis Setup (Optional)

### Redis on Sevalla

1. **Provision Redis**
   - Add Redis addon in Sevalla dashboard
   - Copy connection URL

2. **Configure**
   ```env
   REDIS_URL=redis://:[password]@host:6379
   ```

### External Redis (e.g., Upstash)

1. **Create Redis Instance**
   - Sign up at https://upstash.com/
   - Create database
   - Copy connection string

2. **Configure**
   ```env
   REDIS_URL=rediss://:[password]@host:6379
   ```

---

## Post-Deployment

### Verify Deployment

1. **Health Check**
   ```bash
   curl https://your-app.sevalla.app/health
   ```

2. **Test Chat Endpoint**
   ```bash
   curl -X POST https://your-app.sevalla.app/chat \
     -H "Content-Type: application/json" \
     -H "x-api-key: your-key" \
     -d '{"message": "test"}'
   ```

3. **Check Logs**
   ```bash
   # In Sevalla dashboard
   # Navigate to Logs section
   # Look for startup messages
   ```

### Initial Data Load

```bash
# SSH into server
sevalla run bash

# Ingest documents
npm run ingest:dropbox
npm run ingest:olod
npm run ingest:oit
```

### Create Admin User

```bash
# Connect to PostgreSQL
psql $DATABASE_URL

# Create user
INSERT INTO users (username, email, password_hash, role, is_developer)
VALUES (
  'admin',
  'admin@example.com',
  '$2b$10$your_bcrypt_hash_here',
  'admin',
  true
);
```

Generate password hash:
```javascript
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash('your-password', 10);
console.log(hash);
```

---

## Monitoring

### Application Monitoring

1. **Health Checks**
   - Set up uptime monitoring (e.g., UptimeRobot)
   - Check `/health` endpoint every 5 minutes

2. **Performance Monitoring**
   - Monitor response times
   - Track error rates
   - Check cache hit rates

### Database Monitoring

```sql
-- Check connection count
SELECT COUNT(*) FROM pg_stat_activity;

-- Check table sizes
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check slow queries
SELECT
  query,
  calls,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Alerts

Set up alerts for:
- High error rate (>5%)
- Slow response times (>3s avg)
- Low cache hit rate (<30%)
- Database connection issues
- High memory usage
- API rate limits

---

## Rollback Procedure

If deployment fails:

```bash
# 1. Identify last working commit
git log --oneline

# 2. Revert to that commit
git reset --hard <commit-hash>

# 3. Force push (be careful!)
git push origin main --force

# 4. Monitor Sevalla deployment

# 5. Verify rollback
curl https://your-app.sevalla.app/health
```

---

## Production Best Practices

1. **Never deploy directly to production**
   - Use staging environment first
   - Test thoroughly before prod

2. **Database Backups**
   ```bash
   # Daily backups
   pg_dump $DATABASE_URL > backup-$(date +%Y%m%d).sql

   # Upload to S3 or similar
   ```

3. **Log Retention**
   - Keep logs for at least 30 days
   - Set up log aggregation (e.g., Papertrail)

4. **Security**
   - Use HTTPS (Sevalla provides this)
   - Rotate API keys regularly
   - Keep dependencies updated
   - Monitor for vulnerabilities

5. **Performance**
   - Enable Redis caching
   - Monitor response times
   - Optimize database queries
   - Use CDN for static assets

---

## Troubleshooting Production Issues

### Application Won't Start

Check Sevalla logs for:
```
Missing environment variable: OPENAI_API_KEY
```

**Solution**: Set missing environment variables in Sevalla dashboard

### Database Connection Fails

```
Error: connect ECONNREFUSED
```

**Solution**:
- Check DATABASE_URL is correct
- Verify database is running
- Check IP whitelist (if using external DB)

### High Memory Usage

**Solution**:
- Increase Sevalla plan resources
- Check for memory leaks
- Optimize chunking/caching

---

## Related Documentation

- [Authentication Setup](AUTH_IMPLEMENTATION.md)
- [Intranet Setup](INTRANET_SETUP.md)
- [Google Sheets Setup](GOOGLE_SHEETS_SETUP.md)
- [Troubleshooting](../guides/TROUBLESHOOTING.md)

---

**Last Updated**: October 2025
