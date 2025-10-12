# ColdFusion Authentication Passthrough Status

## Summary

Testing shows which ColdFusion apps on devssl.caes.uga.edu already have crawler passthrough vs which need it added.

**Strategy:** Crawl DEV sites (`devssl.caes.uga.edu`), but show PROD URLs (`secure.caes.uga.edu`) to users during ingestion.

## ✅ Apps Already Working (3)

These apps already have the `X-Crawl-Token` passthrough configured:

| App | Dev URL | Status |
|-----|---------|--------|
| **Extension Training System (ETS)** | https://devssl.caes.uga.edu/trainings2/ | ✅ Working |
| **Extension Calendar** | https://devssl.caes.uga.edu/extension/calendar/admin | ✅ Working |
| **Research Farm Project** | https://devssl.caes.uga.edu/caesresearchfarmproject/ | ✅ Working |
| **Georgia Counts** | https://devssl.caes.uga.edu/gacounts3/ | ✅ Working (empty homepage but auth works) |

### Georgia Counts Note
GaCounts returns 0 bytes on the homepage, but authentication IS working. The JavaScript crawler (`crawlGaCounts.js`) already successfully crawled it using both URL parameter and header authentication.

## ❌ Apps Needing Passthrough Added (6)

These apps redirect to SSO login and need the authentication passthrough configured:

| # | App | Dev URL | application.cfm/cfc Location |
|---|-----|---------|------------------------------|
| 1 | **CAES Calendar** | https://devssl.caes.uga.edu/caescalendar/ | `/inetpub/wwwroot/Applications/CAESCalendar/` |
| 2 | **Extension Supply List** | https://devssl.caes.uga.edu/supplylist/ | `/inetpub/wwwroot/Applications/SupplyList/` |
| 3 | **Farm Gate Value Survey** | https://devssl.caes.uga.edu/farmgate/ | `/inetpub/wwwroot/Applications/FarmGate/` |
| 4 | **File Sharing** | https://devssl.caes.uga.edu/filesharing/ | `/inetpub/wwwroot/Applications/FileSharing/` |
| 5 | **Impact Statements** | https://devssl.caes.uga.edu/impactstatements/ | `/inetpub/wwwroot/Applications/ImpactStatements/` |
| 6 | **Personnel Database** | https://devssl.caes.uga.edu/personnel/admin/ | `/inetpub/wwwroot/Applications/Personnel/` |

## Authentication Implementation

### Pattern from Working Apps

Looking at **GaCounts** and **ETS**, the authentication pattern is:

**In `application.cfm` or `application.cfc`:**

```coldfusion
<cfscript>
    // Check for crawler token in header
    crawlerToken = cgi.HTTP_X_CRAWL_TOKEN;

    // Or check in URL parameter (fallback)
    if (NOT len(trim(crawlerToken))) {
        crawlerToken = url.crawl_token ?: "";
    }

    // Validate token
    validCrawlerToken = "dev_crawler_secret_123"; // From config

    if (len(trim(crawlerToken)) AND crawlerToken EQ validCrawlerToken) {
        // Bypass SSO authentication
        session.authenticated = true;
        session.isCrawler = true;
        // Set minimal user info for the crawler
        session.user = {
            username: "crawler",
            fullname: "CAES Crawler Bot",
            role: "readonly"
        };
    } else {
        // Normal SSO authentication flow
        // ... existing SSO code ...
    }
</cfscript>
```

### Files to Modify

For each app, you need to:

1. **Edit `application.cfm` or `application.cfc`**
2. **Add crawler token check BEFORE SSO authentication**
3. **Test with:** `https://devssl.caes.uga.edu/[app]/?crawl_token=dev_crawler_secret_123`

### Token Configuration

**Dev Token:** `dev_crawler_secret_123`
**Prod Token:** `CAES_CRAWLER_SECRET_2024_PROD` (not needed if only crawling dev)

## Testing

After adding passthrough to each app, test with:

```bash
cd python
python test_coldfusion_dev_auth.py
```

Should show:
- ✅ Success for apps with passthrough
- ❌ Fail (redirects to SSO) for apps still needing it

## Next Steps

### 1. Add Passthrough to 6 Apps
Use the pattern from GaCounts/ETS to add crawler authentication to the 6 apps listed above.

### 2. Update WordPress Crawler
Once all apps pass authentication, enhance the WordPress crawler to:
- Follow links to ColdFusion apps
- Auto-detect domain and use appropriate token
- Maintain document relationship mapping

### 3. URL Mapping During Ingestion
Update `ingest.js` to replace dev URLs with prod URLs:
- `devssl.caes.uga.edu` → `secure.caes.uga.edu`
- Store both URLs in metadata for reference

## Benefits of This Approach

✅ **Security:** No auth passthrough on production servers
✅ **Safety:** Can test on dev without affecting prod users
✅ **Simplicity:** One crawler for all sites (WordPress + ColdFusion)
✅ **Context:** WordPress pages provide navigation context for ColdFusion apps
✅ **Relationships:** Document mapping works across all sites