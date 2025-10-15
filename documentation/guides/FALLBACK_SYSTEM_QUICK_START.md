# Modular Fallback System - Quick Start Guide

**Quick reference for working with fallback responses**

This is a condensed guide for developers who need to quickly work with the fallback system. For comprehensive documentation, see [SYSTEM_PROMPT_ARCHITECTURE.md](../architecture/SYSTEM_PROMPT_ARCHITECTURE.md).

---

## Quick Reference

### File Locations

```
src/
├── config/
│   └── fallbackResponses.js     ← URL constants & fallback templates
├── prompts/
│   ├── index.js                 ← Prompt builder & validator
│   └── systemPrompt.txt         ← Base prompt rules
└── server.js                    ← Loads system prompt

tests/
└── test-fallback-responses.js   ← Automated validation tests
```

---

## Common Tasks

### View All Fallback Templates

```javascript
import { getAllFallbackTemplates } from './src/config/fallbackResponses.js';

const templates = getAllFallbackTemplates();
console.log(templates);
```

### Get Current System Prompt

```javascript
import { getSystemPrompt } from './src/prompts/index.js';

const prompt = getSystemPrompt();
console.log(prompt);
```

### Add a New Fallback (5 steps)

#### 1. Add URL constant

```javascript
// src/config/fallbackResponses.js
export const CANONICAL_URLS = {
  // ...existing
  MY_RESOURCE: 'https://example.uga.edu/',
};
```

#### 2. Add contact info (optional)

```javascript
export const CONTACTS = {
  // ...existing
  MY_RESOURCE: {
    email: 'help@uga.edu',
    phone: '706-542-XXXX',
  },
};
```

#### 3. Create fallback template

```javascript
export const FALLBACK_RESPONSES = {
  // ...existing
  myResource: () =>
    buildFallbackTemplate({
      primaryUrl: CANONICAL_URLS.MY_RESOURCE,
      primaryLinkText: 'My Resource',
      description: 'For help with this resource, visit',
      contactInfo: `Contact: ${CONTACTS.MY_RESOURCE.email}`,
    }),
};
```

#### 4. Add to system prompt

```javascript
// src/prompts/index.js - in buildFallbackSection()
For my resource queries:
"${FALLBACK_RESPONSES.myResource()}"
```

#### 5. Run tests

```bash
node tests/test-fallback-responses.js
```

---

## Update Existing URL

### Change URL Everywhere

```javascript
// src/config/fallbackResponses.js
export const CANONICAL_URLS = {
  CAES_OIT: 'https://oit.caes.uga.edu/', // Change here
};

// That's it! Used everywhere automatically ✅
```

### Update Contact Info

```javascript
// src/config/fallbackResponses.js
export const CONTACTS = {
  CAES_OIT: {
    athens: {
      email: 'newemail@uga.edu', // Change here
      phone: '706-542-XXXX',
    },
  },
};

// Updates all templates automatically ✅
```

---

## Testing

### Run All Tests

```bash
node tests/test-fallback-responses.js
```

### Validate System Prompt

```bash
node -e "import('./src/prompts/index.js').then(m => m.printValidationResults())"
```

### Test Specific Fallback

```javascript
import { FALLBACK_RESPONSES } from './src/config/fallbackResponses.js';

const output = FALLBACK_RESPONSES.technical();
console.log(output);

// Verify URL appears first
const urlIndex = output.indexOf('[');
const emailIndex = output.indexOf('@');
console.log('URL first:', urlIndex < emailIndex); // Should be true
```

---

## Debugging

### Check Current Prompt

```javascript
// In Node REPL or a script
import { getSystemPrompt } from './src/prompts/index.js';

const prompt = getSystemPrompt();

// Search for specific section
if (prompt.includes('For technical issues:')) {
  console.log('✓ Technical fallback present');
}

// Check prompt size
console.log(`Prompt size: ${(prompt.length / 1024).toFixed(2)} KB`);
```

### Force Reload Prompt

```javascript
import { getSystemPrompt } from './src/prompts/index.js';

// Force fresh reload (bypasses cache)
const freshPrompt = getSystemPrompt(true);
```

### Validate URLs

```javascript
import { validateCanonicalUrls } from './src/config/fallbackResponses.js';

const validation = validateCanonicalUrls();
console.log('Valid URLs:', validation.valid.length);
console.log('Invalid URLs:', validation.invalid);
console.log('Warnings:', validation.warnings);
```

---

## Hot Reload (Development)

### Enable

```bash
# Set environment variable
set NODE_ENV=development  # Windows
export NODE_ENV=development  # Mac/Linux

# Start server
npm start
```

### Test

1. Edit `src/config/fallbackResponses.js`
2. Save file
3. Watch console:
   ```
   📝 Fallback config changed: fallbackResponses.js, clearing cache...
   🔄 System prompt reloaded (development mode)
   ```
4. Next request uses updated prompt

---

## URL-First Rule

### Rule

**ALL fallbacks MUST show clickable URL before any contact info**

### Good Example ✅

```javascript
buildFallbackTemplate({
  primaryUrl: CANONICAL_URLS.CAES_OIT,
  primaryLinkText: 'CAES OIT',
  description: 'For technical support, visit',
  contactInfo: 'Contact: oithelp@uga.edu', // Comes AFTER URL
});
```

**Output:**

```
For technical support, visit [CAES OIT](https://oit.caes.uga.edu/)

Contact: oithelp@uga.edu
```

### Bad Example ❌

```javascript
// DON'T DO THIS - contact info in description (before URL)
buildFallbackTemplate({
  description: 'Contact oithelp@uga.edu or visit', // ❌ Email first
  primaryUrl: CANONICAL_URLS.CAES_OIT,
  primaryLinkText: 'CAES OIT',
});
```

---

## Cheat Sheet

### All Available Fallbacks

| Function                          | Use Case                      |
| --------------------------------- | ----------------------------- |
| `FALLBACK_RESPONSES.general()`    | Unknown/broad topics          |
| `FALLBACK_RESPONSES.department()` | Department questions          |
| `FALLBACK_RESPONSES.policy()`     | Policy/procedure questions    |
| `FALLBACK_RESPONSES.technical()`  | Technical support             |
| `FALLBACK_RESPONSES.it()`         | IT help (alias for technical) |
| `FALLBACK_RESPONSES.leave()`      | Leave/PTO questions           |

### All Canonical URLs

```javascript
CANONICAL_URLS.CAES_INTRANET; // https://intranet.caes.uga.edu/
CANONICAL_URLS.CAES_DEPARTMENTS; // https://www.caes.uga.edu/departments.html
CANONICAL_URLS.CAES_OIT; // https://oit.caes.uga.edu/
CANONICAL_URLS.CAES_ABO; // https://abo.caes.uga.edu/
CANONICAL_URLS.UGA_POLICIES; // https://legal.uga.edu/policies/
CANONICAL_URLS.UGA_EITS; // https://eits.uga.edu/
CANONICAL_URLS.TEAMDYNAMIX_LEAVE; // https://uga.teamdynamix.com/...
```

### Contact Info

```javascript
CONTACTS.CAES_OIT.athens.email; // oithelp@uga.edu
CONTACTS.CAES_OIT.athens.phone; // 706-542-2139
CONTACTS.CAES_OIT.griffin.email; // grifoit@uga.edu
CONTACTS.CAES_OIT.griffin.phone; // 770-229-3020
CONTACTS.UGA_EITS.email; // helpdesk@uga.edu
```

---

## Common Patterns

### Adding Multiple Links

```javascript
buildFallbackTemplate({
  primaryUrl: CANONICAL_URLS.CAES_ABO,
  primaryLinkText: 'ABO',
  description: 'For policy information, visit',
  additionalLinks: [
    { text: 'UGA Policy Directory', url: CANONICAL_URLS.UGA_POLICIES },
    { text: 'CAES Intranet', url: CANONICAL_URLS.CAES_INTRANET },
  ],
});
```

### Custom Contact Block

```javascript
buildFallbackTemplate({
  primaryUrl: CANONICAL_URLS.CAES_OIT,
  primaryLinkText: 'CAES OIT',
  description: 'For technical support, visit',
  contactInfo: `You can also contact:
- Athens Campus: Email ${CONTACTS.CAES_OIT.athens.email} or call ${CONTACTS.CAES_OIT.athens.phone}
- Griffin Campus: Email ${CONTACTS.CAES_OIT.griffin.email} or call ${CONTACTS.CAES_OIT.griffin.phone}`,
});
```

### Adding Guidance Text

```javascript
buildFallbackTemplate({
  primaryUrl: CANONICAL_URLS.CAES_INTRANET,
  primaryLinkText: 'CAES Intranet',
  description: 'For general information, visit',
  additionalGuidance: `This might help if:
- You're new to CAES
- You need department contacts
- You want to find forms or policies`,
});
```

---

## Troubleshooting

| Problem                | Solution                                                   |
| ---------------------- | ---------------------------------------------------------- |
| Tests fail             | Run `node tests/test-fallback-responses.js` to see details |
| URL not first          | Check `description` doesn't contain contact info           |
| Changes not applied    | Run `getSystemPrompt(true)` to force reload                |
| Server won't start     | Check syntax in `fallbackResponses.js`                     |
| Hot reload not working | Verify `NODE_ENV=development`                              |

---

## Need More Help?

- **Full documentation:** [SYSTEM_PROMPT_ARCHITECTURE.md](../architecture/SYSTEM_PROMPT_ARCHITECTURE.md)
- **Testing guide:** [TESTING_GUIDE.md](./TESTING_GUIDE.md)
- **Code recipes:** [CODE_RECIPES.md](./CODE_RECIPES.md)
- **Ask the team:** See [DEVELOPER_GUIDE.md](../DEVELOPER_GUIDE.md)

---

**Last Updated:** October 15, 2025
