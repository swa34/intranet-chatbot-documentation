# Sevalla Redis Configuration (CONFIDENTIAL)

## Redis Database Details

**Database Name:** pincone-cache
**Status:** ✅ Ready
**Version:** Redis 8.x
**Location:** South Carolina, USA (us-east1)
**Resources:** 0.25 CPU, 0.25 GB RAM, 1 GB storage
**Created:** October 10, 2025

## Connection Information

### Internal Connection (Production - Use This)
- **Host:** `pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local`
- **Port:** `6379`
- **Password:** `[STORED IN .env]`
- **URL Format:** `redis://:[PASSWORD]@[INTERNAL_HOST]:6379/0`

### External Connection (Testing/Development Only)
- **Host:** `us-east1-001.proxy.kinsta.app`
- **Port:** `30088`
- **Password:** `[STORED IN .env]`
- **URL Format:** `redis://:[PASSWORD]@us-east1-001.proxy.kinsta.app:30088/0`

## Connected Applications
✅ **CAES Intranet Help Bot** - Connected via internal connection

## Environment Variable Setup

Add to your `.env` file:

```env
# Redis Configuration for Pinecone Caching
# Use INTERNAL connection for production (faster, secure)
# Format: redis://:[password]@[host]:[port]/[database]
REDIS_URL=redis://:[YOUR_PASSWORD]@pincone-cache-cep24-redis-master.pincone-cache-cep24.svc.cluster.local:6379/0

# For local testing, use external connection:
# REDIS_URL=redis://:[YOUR_PASSWORD]@us-east1-001.proxy.kinsta.app:30088/0
```

## Important Notes

1. **Use Internal Connection in Production** - It's faster and more secure
2. **External Connection is for Testing** - Only use when developing locally
3. **Password Security** - Never commit the password to git
4. **TTL Setting** - Cache expires after 1 hour (3600 seconds)
5. **Automatic Fallback** - If Redis fails, app falls back to local caching

## Monitoring

Check Redis connection in your logs:
- `✅ Redis connected for Pinecone caching` - Connection successful
- `♻️ Using Redis-cached Pinecone validation` - Cache hit
- `💾 Cached Pinecone validation in Redis` - Cache write

## Cost
- $5/month for Redis add-on
- Shared with your application resources (DB1 plan)

## Security
- Internal connection is only accessible within Sevalla cluster
- External connection requires password authentication
- Public access can be disabled after testing