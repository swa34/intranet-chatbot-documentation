# Authentication System Setup Guide

## Overview

Your UGA CAES Intranet Chatbot now has a complete user authentication system with:

- ✅ User signup with @uga.edu email validation
- ✅ Secure login with bcrypt password hashing
- ✅ Session management using Redis (fast & scalable)
- ✅ Role-based access control (admin/tester roles)
- ✅ User profile with intranet knowledge level (1-10)
- ✅ PostgreSQL database for user data
- ✅ Rate limiting on login attempts (security)
- ✅ Beautiful, responsive signup & login pages

## Database Schema

### Users Table

```sql
- id (primary key)
- email (unique, must be @uga.edu)
- first_name
- last_name
- password_hash (bcrypt encrypted)
- role (admin or tester)
- intranet_knowledge_level (1-10)
- is_active (boolean)
- created_at
- last_login
```

### Sessions

- Stored in **Redis** for blazing fast lookups
- 24-hour expiration (auto-cleanup)
- Token-based authentication

## File Structure

```
src/
├── auth/
│   ├── authRoutes.js          # API endpoints for signup/login/logout
│   ├── authUtils.js           # Validation & password hashing utilities
│   └── sessionManager.js      # Redis session management
├── middleware/
│   └── sessionAuthMiddleware.js  # Authentication middleware
└── migrations/
    └── 001_create_users_table.js # Database setup

public/
└── auth/
    ├── signup.html            # User registration page
    └── login.html             # User login page
```

## API Endpoints

### Authentication

- `POST /api/auth/signup` - Register new user
- `POST /api/auth/login` - Login user
- `POST /api/auth/logout` - Logout user
- `GET /api/auth/me` - Get current user info
- `GET /api/auth/sessions` - Get active sessions

### Request/Response Examples

**Signup:**

```javascript
POST /api/auth/signup
{
  "email": "john.doe@uga.edu",
  "password": "SecurePass123",
  "firstName": "John",
  "lastName": "Doe",
  "knowledgeLevel": 7
}

Response: 201 Created
{
  "success": true,
  "user": { id, email, first_name, last_name, role },
  "sessionToken": "..."
}
```

**Login:**

```javascript
POST /api/auth/login
{
  "email": "john.doe@uga.edu",
  "password": "SecurePass123"
}

Response: 200 OK
{
  "success": true,
  "user": { ... },
  "sessionToken": "..."
}
```

## How to Use

### 1. Start the Server

```bash
npm start
```

### 2. Access the Application

- **Signup:** http://localhost:3000/auth/signup.html
- **Login:** http://localhost:3000/auth/login.html
- **Chatbot:** http://localhost:3000 (requires login)

### 3. Create Your First Admin User

**Option A: Via Signup Page**

1. Go to `/auth/signup.html`
2. Register with your @uga.edu email
3. Manually update role in database:

```sql
UPDATE users SET role = 'admin' WHERE email = 'your.email@uga.edu';
```

**Option B: Direct Database Insert**

```javascript
// Run this in psql or a database tool
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash('YourPassword123', 10);

INSERT INTO users (email, first_name, last_name, password_hash, role, intranet_knowledge_level)
VALUES ('admin@uga.edu', 'Admin', 'User', '<hash>', 'admin', 10);
```

## User Roles

### Tester (Default)

- Access chatbot
- Provide feedback
- View own conversation history

### Admin

- All tester permissions
- Access admin panel (`/admin/*`)
- View analytics & metrics
- Manage cache
- Developer tools access

## Security Features

### Password Requirements

- Minimum 8 characters
- At least one uppercase letter
- At least one lowercase letter
- At least one number

### Email Validation

- Must be @uga.edu domain
- Validated on both frontend and backend

### Rate Limiting

- Max 5 failed login attempts per 15 minutes
- Prevents brute force attacks

### Session Security

- HttpOnly cookies (prevents XSS)
- Secure flag in production (HTTPS only)
- SameSite protection (CSRF prevention)
- 24-hour automatic expiration

## Environment Variables

Ensure these are set in your `.env`:

```bash
# Database (already configured)
DB_HOST=your-host
DB_PORT=5432
DB_DATABASE=your-database
DB_USERNAME=your-username
DB_PASSWORD=your-password

# Redis (already configured)
REDIS_URL=redis://your-redis-url

# API Keys (already configured)
CHATBOT_API_KEY=your-key
OPENAI_API_KEY=your-key
PINECONE_API_KEY=your-key
```

## Migration Commands

Run database migrations:

```bash
npm run migrate:auth
```

## Switching Between Auth Systems

The old basic HTTP auth is still available. To switch:

**In `server.js` line 90:**

```javascript
// New session-based auth (current)
app.use(sessionAuthMiddleware);

// Or revert to old basic auth
app.use(authMiddleware);
```

## Testing Checklist

- [ ] Run migration: `npm run migrate:auth`
- [ ] Start server: `npm start`
- [ ] Visit `/auth/signup.html`
- [ ] Create test account with @uga.edu email
- [ ] Verify email domain validation works
- [ ] Test password requirements
- [ ] Test knowledge level slider
- [ ] Complete signup
- [ ] Verify redirect to home page
- [ ] Test logout
- [ ] Try login with same credentials
- [ ] Verify session persists
- [ ] Test invalid login (wrong password)
- [ ] Verify rate limiting after 5 failed attempts

## Next Steps

### 1. Create Your First Admin

Update the first user to admin role (see above)

### 2. Customize Roles

You can add more roles in the future:

- Modify `authUtils.js` `validateRole()` function
- Update database constraint
- Add middleware checks

### 3. Add Password Reset

Future enhancement to add "Forgot Password" functionality

### 4. Email Verification

Optional: Add email verification on signup

### 5. Two-Factor Authentication

Optional: Add 2FA for enhanced security

## Troubleshooting

### Redis Connection Issues

```bash
# Check if Redis is running
redis-cli ping
# Should return: PONG

# If not running, start Redis or check REDIS_URL in .env
```

### Database Connection Issues

```bash
# Test PostgreSQL connection
psql -h $DB_HOST -U $DB_USERNAME -d $DB_DATABASE
```

### Session Not Persisting

- Check browser cookies (should see `session_token`)
- Verify Redis is running
- Check console for errors

### Migration Errors

```bash
# If migration fails, check:
# 1. Database connection
# 2. Database user has CREATE TABLE permissions
# 3. Tables don't already exist (migration is idempotent)
```

## Architecture Diagram

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │
       │ HTTP/HTTPS
       ▼
┌─────────────────────┐
│   Express Server    │
│  (sessionAuthMW)    │
└──────┬──────────────┘
       │
       ├──────────────────────┐
       │                      │
       ▼                      ▼
┌─────────────┐     ┌──────────────┐
│   Redis     │     │  PostgreSQL  │
│  (sessions) │     │   (users)    │
└─────────────┘     └──────────────┘
```

## Support

For issues or questions:

1. Check this documentation
2. Review code comments in auth files
3. Check server logs for error messages
4. Verify all environment variables are set

---

**Built with ❤️ for UGA CAES**

Branch: `feature/user-authentication`
Migration: `001_create_users_table.js`
