# Google Sheets User Tracking Setup

## Overview
The feedback system now tracks which authenticated user submitted each feedback entry.

## Google Sheets Setup

### 1. Add Column Header
Open your Google Sheet and add a header in column J:
- **Column J**: `User`

Your headers should now be:
- A: Timestamp
- B: Message ID
- C: Question
- D: Answer
- E: Rating
- F: Comments
- G: Sources
- H: User Agent
- I: IP Address
- J: **User** (NEW)

### 2. Update Environment Variables

#### For Local Development (.env)
```bash
GOOGLE_SHEETS_RANGE=Sheet1!A:J
AUTHORIZED_USERS=scott:sa69508,jesse:jdk48542,matt:mgn19478,ben:bwhet
```

#### For Sevalla Deployment
Add/Update these environment variables:
- `GOOGLE_SHEETS_RANGE`: `Sheet1!A:J`
- `AUTHORIZED_USERS`: `scott:sa69508,jesse:jdk48542,matt:mgn19478,ben:bwhet`

## How It Works

1. User logs in with their credentials (e.g., scott:sa69508)
2. Authentication middleware adds username to the request
3. When feedback is submitted, the username is captured
4. Username is saved in column J of the Google Sheet

## Example Data
```
Timestamp | ... | Rating | ... | User
2025-09-19T12:00:00Z | ... | helpful | ... | scott
2025-09-19T12:05:00Z | ... | not-helpful | ... | jesse
2025-09-19T12:10:00Z | ... | helpful | ... | matt
```

## Benefits
- Track which team member is testing the app
- Identify patterns in feedback by user
- Monitor usage during development
- Accountability for feedback quality

## Testing
1. Log in with your credentials
2. Submit feedback
3. Check Google Sheet - your username should appear in column J

## Notes
- Users not logged in will show as "anonymous"
- Old feedback entries won't have user data
- Works with both single-user and multi-user authentication modes