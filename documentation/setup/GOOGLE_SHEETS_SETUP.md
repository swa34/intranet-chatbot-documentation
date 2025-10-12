# Google Sheets Feedback Setup Guide

This guide will help you set up Google Sheets to collect feedback from the chatbot.

## Step 1: Create a Google Sheet

1. Go to [Google Sheets](https://sheets.google.com)
2. Create a new spreadsheet
3. Name it "Chatbot Feedback" (or your preference)
4. In the first row, add these headers:
   - A1: `Timestamp`
   - B1: `Message ID`
   - C1: `Question`
   - D1: `Answer`
   - E1: `Rating`
   - F1: `Comments`
   - G1: `Sources`
   - H1: `User Agent`
   - I1: `IP Address`

5. Note the Spreadsheet ID from the URL:
   - URL format: `https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]/edit`
   - Copy the `SPREADSHEET_ID` part

## Step 2: Create a Service Account

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Enable Google Sheets API:
   - Go to "APIs & Services" > "Library"
   - Search for "Google Sheets API"
   - Click on it and press "Enable"

4. Create Service Account:
   - Go to "APIs & Services" > "Credentials"
   - Click "Create Credentials" > "Service Account"
   - Give it a name like "chatbot-feedback"
   - Click "Create and Continue"
   - Skip the optional steps
   - Click "Done"

5. Create Key:
   - Click on your new service account
   - Go to "Keys" tab
   - Click "Add Key" > "Create New Key"
   - Choose "JSON"
   - Save the downloaded JSON file securely

## Step 3: Share Sheet with Service Account

1. Open the JSON key file you downloaded
2. Find the `"client_email"` field (looks like: `something@project.iam.gserviceaccount.com`)
3. Go back to your Google Sheet
4. Click the "Share" button
5. Paste the service account email
6. Give it "Editor" permission
7. Click "Send"

## Step 4: Configure the Application

1. Add these to your `.env` file:
```bash
# Google Sheets Configuration
GOOGLE_SHEETS_ID=your_spreadsheet_id_here
GOOGLE_SHEETS_RANGE=Sheet1!A:I
GOOGLE_SERVICE_ACCOUNT_EMAIL=your-service-account@project.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nYour private key here\n-----END PRIVATE KEY-----\n"

# Or path to JSON key file (choose one method)
GOOGLE_APPLICATION_CREDENTIALS=./path/to/your-service-account-key.json
```

2. Choose ONE method:
   - **Method A**: Copy the `client_email` and `private_key` from the JSON file to `.env`
   - **Method B**: Save the entire JSON file and reference its path

## Step 5: Test the Integration

1. Restart the server
2. Ask the chatbot a question
3. Submit feedback
4. Check your Google Sheet - the feedback should appear!

## Security Notes

⚠️ **IMPORTANT**:
- Never commit the service account key or `.env` file to git
- Keep the JSON key file secure
- For production, use Kinsta's environment variables

## Viewing the Data

Your Google Sheet will automatically update with:
- Real-time feedback as users submit
- All data in a sortable, filterable format
- Easy sharing with stakeholders
- Built-in charts and analytics capabilities

## Troubleshooting

If feedback isn't appearing:
1. Check the service account has Editor permission on the sheet
2. Verify the Sheet ID is correct
3. Ensure the API is enabled in Google Cloud Console
4. Check server logs for error messages

## For Kinsta Deployment

In Kinsta's MyKinsta dashboard:
1. Go to your site > Settings > Environment variables
2. Add the same variables from your `.env`
3. Deploy your application

The feedback will continue flowing to the same Google Sheet!