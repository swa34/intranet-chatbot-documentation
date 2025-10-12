# Google Sheets Integration Troubleshooting for Sevalla

## Issue
The Google Sheets integration works locally but fails when deployed to Sevalla.

## Implemented Solutions

### 1. Enhanced Logging
Added detailed logging to `googleSheetsStorage.js` to track:
- Environment variable detection
- Authentication attempts
- API connection status
- Error details with full context

### 2. Diagnostic Endpoint
Created `/feedback/diagnostics` endpoint to check:
- Environment variable presence and format
- Private key structure
- Sevalla-specific variables

### 3. Private Key Handling
Updated authentication to handle various private key formats:
- Removes wrapping quotes (single or double)
- Properly converts escaped newlines
- Validates key structure

## Troubleshooting Steps

### Step 1: Check Diagnostics
```bash
curl -H "x-api-key: YOUR_API_KEY" https://your-app.sevalla.app/feedback/diagnostics
```

### Step 2: Verify Environment Variables on Sevalla

1. **GOOGLE_PRIVATE_KEY Format**
   - On Sevalla, wrap the private key in single quotes:
   ```
   GOOGLE_PRIVATE_KEY='-----BEGIN PRIVATE KEY-----
   MIIEvQIBADANBgkqhkiG9w0BAQ...
   -----END PRIVATE KEY-----'
   ```
   - Do NOT use double quotes
   - Do NOT escape newlines (no \n)
   - Include actual line breaks

2. **Alternative: Use Base64 Encoding**
   ```bash
   # Encode your key locally
   base64 -i service-account-key.json -o encoded-key.txt

   # Set on Sevalla
   GOOGLE_APPLICATION_CREDENTIALS_BASE64='<contents of encoded-key.txt>'
   ```

3. **Required Variables**
   - `GOOGLE_SHEETS_ID`: Your spreadsheet ID
   - `GOOGLE_SHEETS_RANGE`: Sheet1!A:I (or your range)
   - Either:
     - `GOOGLE_SERVICE_ACCOUNT_EMAIL` + `GOOGLE_PRIVATE_KEY`
     - OR `GOOGLE_APPLICATION_CREDENTIALS` (file path)

### Step 3: Test Connection
```bash
curl -H "x-api-key: YOUR_API_KEY" https://your-app.sevalla.app/feedback/status
```

### Step 4: Check Logs
Monitor Sevalla logs for enhanced debug output:
```
🔧 Initializing Google Sheets Storage...
🔑 Attempting authentication with environment variables...
✅ Successfully authenticated with environment variables
📊 Spreadsheet found: [Your Sheet Name]
✅ Google Sheets connection established successfully
```

## Common Issues on Sevalla

### 1. Private Key Formatting
**Problem**: Newlines in private key not preserved
**Solution**: Use single quotes and ensure actual line breaks

### 2. Network Access
**Problem**: Outbound connections blocked
**Solution**: Sevalla allows outbound HTTPS by default - no action needed

### 3. Environment Variable Size
**Problem**: Large private key truncated
**Solution**: Use Base64 encoding to reduce special characters

### 4. Authentication Scope
**Problem**: Service account lacks permissions
**Solution**: Share the Google Sheet with the service account email

## Testing Locally with Sevalla Environment

```bash
# Export the same variables locally
export GOOGLE_SHEETS_ID='your-sheet-id'
export GOOGLE_SERVICE_ACCOUNT_EMAIL='your-service@project.iam.gserviceaccount.com'
export GOOGLE_PRIVATE_KEY='-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----'

# Run locally
npm start

# Test feedback endpoint
curl -X POST http://localhost:3000/feedback \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question":"test","answer":"test","rating":5}'
```

## Alternative: File-based Authentication

If environment variables continue to fail:

1. Create a secure endpoint to upload the service account JSON
2. Store it in Sevalla's persistent storage
3. Reference via `GOOGLE_APPLICATION_CREDENTIALS=/storage/service-account.json`

## Fallback Mechanism

The app automatically falls back to local file storage if Google Sheets fails:
- Files saved to: `/feedback/feedback_YYYY-MM-DD.jsonl`
- Check logs for: "📁 Feedback saved to local file"

## Next Steps if Issue Persists

1. Check the diagnostic endpoint response
2. Review Sevalla logs for specific error messages
3. Verify the service account has edit permissions on the sheet
4. Test with a minimal Node.js script on Sevalla to isolate the issue
5. Contact Sevalla support about environment variable handling for multiline values