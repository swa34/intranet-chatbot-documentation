# ColdFusion Session Context for Crawler

## Issue

Some ColdFusion apps require session context (like `session.collegeid`) to display content. The crawler authentication works (bypasses SSO), but the apps error because they expect user session data.

## Apps Affected

**File Sharing** - Requires `session.collegeid` to be set
- Error: `The PARAM_COLLEGEID argument passed to the CAES_FILE_SHARING_GetMembershipForUser function is not of type numeric`
- Location: `F:\inetpub\wwwroot\Applications\FileSharing\Application.cfm:132`

## Solution

Update the crawler authentication in `Application.cfm` to set session context when authenticating the crawler:

### Enhanced Crawler Authentication Pattern

```coldfusion
<cfscript>
    // Check for crawler token
    crawlerToken = cgi.HTTP_X_CRAWL_TOKEN;
    if (NOT len(trim(crawlerToken))) {
        crawlerToken = url.crawl_token ?: "";
    }

    // Check for crawler college ID
    crawlerCollegeId = url.collegeid ?: "23325"; // Default to admin college ID

    // Validate token
    validCrawlerToken = "dev_crawler_secret_123";

    if (len(trim(crawlerToken)) AND crawlerToken EQ validCrawlerToken) {
        // Bypass SSO and set session context
        session.authenticated = true;
        session.isCrawler = true;

        // SET SESSION CONTEXT FOR CRAWLER
        session.collegeid = val(crawlerCollegeId); // Convert to numeric
        session.username = "sa69508";
        session.fullname = "CAES Crawler Bot";
        session.role = "admin";
        session.email = "caesweb@uga.edu";

        // App-specific session variables
        session.user = {
            collegeid: val(crawlerCollegeId),
            username: "sa69508",
            fullname: "CAES Crawler Bot",
            role: "admin"
        };

        // Skip normal SSO authentication
        // ... continue to app ...
    } else {
        // Normal SSO authentication flow
        // ... existing SSO code ...
    }
</cfscript>
```

## Apps That Work Without Session Context

These apps work fine with just the basic crawler authentication:

✅ Extension Training System (ETS)
✅ Extension Calendar
✅ CAES Calendar
✅ Georgia Counts
✅ Extension Supply List
✅ Farm Gate Value Survey
✅ Impact Statements
✅ Personnel Database
✅ Research Farm Project Database

## Apps Needing Enhanced Session Context

❌ **File Sharing** - Needs `session.collegeid` set

## Workaround for Now

The crawler will successfully crawl:
1. WordPress intranet pages
2. ColdFusion apps that don't require session context (9 out of 10 apps)
3. Help pages and public-facing pages in all apps

For **File Sharing**, the enhanced session context can be added later, or we can manually crawl specific pages that don't require membership checks.

## Testing

After adding enhanced session context to File Sharing:

```bash
curl -H "X-Crawl-Token: dev_crawler_secret_123" \
  "https://devssl.caes.uga.edu/filesharing/?function=help&collegeid=23325&crawl_token=dev_crawler_secret_123"
```

Should return help content instead of error.