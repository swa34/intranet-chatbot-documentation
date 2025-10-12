# Contextual Form Help Implementation Guide

## Overview
Add live Q&A assistance to forms while users fill them out. Users can ask questions about specific fields, form requirements, or procedures directly within the form interface.

---

## Backend Changes (Node.js/Express)

### 1. Update Chat Endpoint to Accept Form Context

**File: `src/server.js`**

Modify the `/chat` endpoint to accept optional form context:

```javascript
app.post('/chat', requireApiKey, async (req, res) => {
  const startTime = Date.now();

  try {
    const { message, sessionId, formContext } = req.body;  // Add formContext

    if (!message) {
      return res.status(400).json({ error: 'No message provided.' });
    }

    const activeSessionId = sessionId || conversationMemory.createSessionId();

    // Log form context if provided
    if (formContext) {
      console.log('Form context:', {
        formName: formContext.formName,
        currentField: formContext.currentField,
        section: formContext.section
      });
    }

    // ... rest of existing code
```

### 2. Enhance System Prompt with Form Context

**File: `src/server.js`** (in the chat endpoint, around line 300)

Add form context to the system prompt:

```javascript
// Build form context section
let formContextSection = '';
if (formContext) {
  formContextSection = `

FORM CONTEXT:
The user is currently filling out: ${formContext.formName}
${formContext.currentField ? `Current field: ${formContext.currentField}` : ''}
${formContext.section ? `Current section: ${formContext.section}` : ''}
${formContext.fieldLabel ? `Field label: ${formContext.fieldLabel}` : ''}

When answering:
1. Prioritize information specific to this form and field
2. Be concise and actionable
3. Provide examples if relevant to this field
4. Reference help documentation when available
`;
}

const enhancedSystemPrompt = `${systemPrompt}

CRITICAL: Use ONLY Markdown formatting. NEVER use HTML tags like <a href="">.
${formContextSection}
${historySection}`;
```

### 3. Optionally Boost Form-Related Content in Retrieval

**File: `src/rag/retrieve.js`** (optional enhancement)

Add metadata filtering or score boosting for form-specific content:

```javascript
export async function retrieveRelevantChunks(queryText, formContext = null) {
  // ... existing code ...

  // If form context provided, boost relevant results
  if (formContext && formContext.formName) {
    const formKeywords = formContext.formName.toLowerCase().split(' ');

    matches = matches.map(match => {
      const text = match.metadata.text.toLowerCase();
      const hasFormKeyword = formKeywords.some(keyword => text.includes(keyword));

      if (hasFormKeyword) {
        match.score = match.score * 1.2; // Boost score by 20%
      }

      return match;
    });

    // Re-sort after boosting
    matches.sort((a, b) => b.score - a.score);
  }

  // ... rest of existing code
}
```

### 4. Update the Chat Handler Call

**File: `src/server.js`** (around line 237)

Pass form context to retrieve function:

```javascript
// 1) Retrieve from Pinecone with aggregation support and form context
const { matches, belowThreshold, topScore, isAggregation } =
  await retrieveRelevantChunks(questionForRetrieval, formContext);
```

---

## Frontend Implementation

### Option A: React Implementation

#### 1. Create Form Help Component

**File: `components/FormHelp.jsx`**

```jsx
import { useState, useEffect } from 'react';
import './FormHelp.css';

export default function FormHelp({ formName, currentField, fieldLabel, section }) {
  const [isOpen, setIsOpen] = useState(false);
  const [messages, setMessages] = useState([]);
  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [sessionId, setSessionId] = useState(null);

  useEffect(() => {
    // Generate session ID on mount
    setSessionId(`form_help_${Date.now()}`);
  }, []);

  const sendMessage = async (message) => {
    if (!message.trim()) return;

    // Add user message to chat
    setMessages(prev => [...prev, { role: 'user', content: message }]);
    setInputValue('');
    setIsLoading(true);

    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/chat`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': process.env.REACT_APP_API_KEY
        },
        body: JSON.stringify({
          message,
          sessionId,
          formContext: {
            formName,
            currentField,
            fieldLabel,
            section
          }
        })
      });

      const data = await response.json();

      // Add assistant response
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: data.answer
      }]);
    } catch (error) {
      console.error('Chat error:', error);
      setMessages(prev => [...prev, {
        role: 'error',
        content: 'Sorry, I encountered an error. Please try again.'
      }]);
    } finally {
      setIsLoading(false);
    }
  };

  const handleQuickQuestion = (question) => {
    sendMessage(question);
  };

  return (
    <div className={`form-help ${isOpen ? 'open' : ''}`}>
      {/* Toggle Button */}
      <button
        className="form-help-toggle"
        onClick={() => setIsOpen(!isOpen)}
        title="Get help with this form"
      >
        ?
      </button>

      {/* Chat Panel */}
      {isOpen && (
        <div className="form-help-panel">
          <div className="form-help-header">
            <h3>Form Help</h3>
            <button onClick={() => setIsOpen(false)}>×</button>
          </div>

          {/* Quick Questions */}
          {messages.length === 0 && currentField && (
            <div className="quick-questions">
              <p>Quick questions about <strong>{fieldLabel || currentField}</strong>:</p>
              <button onClick={() => handleQuickQuestion(`What is ${fieldLabel || currentField}?`)}>
                What is this field?
              </button>
              <button onClick={() => handleQuickQuestion(`What should I enter for ${fieldLabel || currentField}?`)}>
                What should I enter?
              </button>
              <button onClick={() => handleQuickQuestion(`Is ${fieldLabel || currentField} required?`)}>
                Is this required?
              </button>
            </div>
          )}

          {/* Messages */}
          <div className="form-help-messages">
            {messages.map((msg, idx) => (
              <div key={idx} className={`message ${msg.role}`}>
                {msg.content}
              </div>
            ))}
            {isLoading && <div className="message loading">Thinking...</div>}
          </div>

          {/* Input */}
          <div className="form-help-input">
            <input
              type="text"
              value={inputValue}
              onChange={(e) => setInputValue(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && sendMessage(inputValue)}
              placeholder="Ask a question about this form..."
            />
            <button onClick={() => sendMessage(inputValue)}>Send</button>
          </div>
        </div>
      )}
    </div>
  );
}
```

#### 2. Add Styles

**File: `components/FormHelp.css`**

```css
.form-help {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
}

.form-help-toggle {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: #0066cc;
  color: white;
  border: none;
  font-size: 24px;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transition: transform 0.2s;
}

.form-help-toggle:hover {
  transform: scale(1.1);
}

.form-help-panel {
  position: absolute;
  bottom: 80px;
  right: 0;
  width: 400px;
  max-height: 600px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  display: flex;
  flex-direction: column;
}

.form-help-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
}

.form-help-header h3 {
  margin: 0;
  font-size: 18px;
}

.form-help-header button {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #666;
}

.quick-questions {
  padding: 16px;
  background: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;
}

.quick-questions p {
  margin: 0 0 12px 0;
  font-size: 14px;
  color: #666;
}

.quick-questions button {
  display: block;
  width: 100%;
  margin-bottom: 8px;
  padding: 10px;
  background: white;
  border: 1px solid #ddd;
  border-radius: 6px;
  text-align: left;
  cursor: pointer;
  font-size: 14px;
}

.quick-questions button:hover {
  background: #f0f0f0;
}

.form-help-messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  min-height: 200px;
  max-height: 400px;
}

.message {
  margin-bottom: 12px;
  padding: 10px 14px;
  border-radius: 8px;
  font-size: 14px;
  line-height: 1.5;
}

.message.user {
  background: #0066cc;
  color: white;
  margin-left: 40px;
  text-align: right;
}

.message.assistant {
  background: #f0f0f0;
  color: #333;
  margin-right: 40px;
}

.message.loading {
  background: #f0f0f0;
  color: #999;
  font-style: italic;
}

.form-help-input {
  display: flex;
  gap: 8px;
  padding: 16px;
  border-top: 1px solid #e0e0e0;
}

.form-help-input input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
}

.form-help-input button {
  padding: 10px 20px;
  background: #0066cc;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
}

.form-help-input button:hover {
  background: #0052a3;
}
```

#### 3. Use in Your Form Component

**File: `pages/YourForm.jsx`**

```jsx
import { useState } from 'react';
import FormHelp from '../components/FormHelp';

export default function TimesheetForm() {
  const [currentField, setCurrentField] = useState(null);
  const [formData, setFormData] = useState({
    employeeName: '',
    programArea: '',
    contactHours: '',
    // ... other fields
  });

  const fieldLabels = {
    employeeName: 'Employee Name',
    programArea: 'Program Area',
    contactHours: 'Contact Hours',
    // ... other labels
  };

  return (
    <div className="form-container">
      <form>
        <div className="form-group">
          <label htmlFor="employeeName">Employee Name</label>
          <input
            id="employeeName"
            type="text"
            value={formData.employeeName}
            onChange={(e) => setFormData({...formData, employeeName: e.target.value})}
            onFocus={() => setCurrentField('employeeName')}
          />
        </div>

        <div className="form-group">
          <label htmlFor="programArea">Program Area</label>
          <select
            id="programArea"
            value={formData.programArea}
            onChange={(e) => setFormData({...formData, programArea: e.target.value})}
            onFocus={() => setCurrentField('programArea')}
          >
            <option value="">Select...</option>
            <option value="4h">4-H Youth Development</option>
            <option value="ag">Agriculture & Natural Resources</option>
            {/* ... other options */}
          </select>
        </div>

        {/* ... more fields */}
      </form>

      {/* Form Help Widget */}
      <FormHelp
        formName="GaCounts Timesheet"
        currentField={currentField}
        fieldLabel={currentField ? fieldLabels[currentField] : null}
        section="Time Entry"
      />
    </div>
  );
}
```

---

### Option B: Vue Implementation

#### 1. Create Form Help Component

**File: `components/FormHelp.vue`**

```vue
<template>
  <div class="form-help" :class="{ open: isOpen }">
    <!-- Toggle Button -->
    <button
      class="form-help-toggle"
      @click="isOpen = !isOpen"
      title="Get help with this form"
    >
      ?
    </button>

    <!-- Chat Panel -->
    <div v-if="isOpen" class="form-help-panel">
      <div class="form-help-header">
        <h3>Form Help</h3>
        <button @click="isOpen = false">×</button>
      </div>

      <!-- Quick Questions -->
      <div v-if="messages.length === 0 && currentField" class="quick-questions">
        <p>Quick questions about <strong>{{ fieldLabel || currentField }}</strong>:</p>
        <button @click="handleQuickQuestion(`What is ${fieldLabel || currentField}?`)">
          What is this field?
        </button>
        <button @click="handleQuickQuestion(`What should I enter for ${fieldLabel || currentField}?`)">
          What should I enter?
        </button>
        <button @click="handleQuickQuestion(`Is ${fieldLabel || currentField} required?`)">
          Is this required?
        </button>
      </div>

      <!-- Messages -->
      <div class="form-help-messages">
        <div
          v-for="(msg, idx) in messages"
          :key="idx"
          :class="['message', msg.role]"
        >
          {{ msg.content }}
        </div>
        <div v-if="isLoading" class="message loading">Thinking...</div>
      </div>

      <!-- Input -->
      <div class="form-help-input">
        <input
          v-model="inputValue"
          type="text"
          @keypress.enter="sendMessage"
          placeholder="Ask a question about this form..."
        />
        <button @click="sendMessage">Send</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'FormHelp',
  props: {
    formName: String,
    currentField: String,
    fieldLabel: String,
    section: String
  },
  data() {
    return {
      isOpen: false,
      messages: [],
      inputValue: '',
      isLoading: false,
      sessionId: null
    };
  },
  mounted() {
    this.sessionId = `form_help_${Date.now()}`;
  },
  methods: {
    async sendMessage() {
      const message = this.inputValue.trim();
      if (!message) return;

      // Add user message
      this.messages.push({ role: 'user', content: message });
      this.inputValue = '';
      this.isLoading = true;

      try {
        const response = await fetch(`${process.env.VUE_APP_API_URL}/chat`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'x-api-key': process.env.VUE_APP_API_KEY
          },
          body: JSON.stringify({
            message,
            sessionId: this.sessionId,
            formContext: {
              formName: this.formName,
              currentField: this.currentField,
              fieldLabel: this.fieldLabel,
              section: this.section
            }
          })
        });

        const data = await response.json();

        // Add assistant response
        this.messages.push({
          role: 'assistant',
          content: data.answer
        });
      } catch (error) {
        console.error('Chat error:', error);
        this.messages.push({
          role: 'error',
          content: 'Sorry, I encountered an error. Please try again.'
        });
      } finally {
        this.isLoading = false;
      }
    },
    handleQuickQuestion(question) {
      this.inputValue = question;
      this.sendMessage();
    }
  }
};
</script>

<style scoped>
/* Same CSS as React version */
.form-help {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
}

/* ... rest of the CSS from React version ... */
</style>
```

#### 2. Use in Your Form Component

**File: `pages/TimesheetForm.vue`**

```vue
<template>
  <div class="form-container">
    <form>
      <div class="form-group">
        <label for="employeeName">Employee Name</label>
        <input
          id="employeeName"
          v-model="formData.employeeName"
          type="text"
          @focus="currentField = 'employeeName'"
        />
      </div>

      <div class="form-group">
        <label for="programArea">Program Area</label>
        <select
          id="programArea"
          v-model="formData.programArea"
          @focus="currentField = 'programArea'"
        >
          <option value="">Select...</option>
          <option value="4h">4-H Youth Development</option>
          <option value="ag">Agriculture & Natural Resources</option>
        </select>
      </div>

      <!-- ... more fields -->
    </form>

    <!-- Form Help Widget -->
    <FormHelp
      form-name="GaCounts Timesheet"
      :current-field="currentField"
      :field-label="fieldLabels[currentField]"
      section="Time Entry"
    />
  </div>
</template>

<script>
import FormHelp from '@/components/FormHelp.vue';

export default {
  name: 'TimesheetForm',
  components: {
    FormHelp
  },
  data() {
    return {
      currentField: null,
      formData: {
        employeeName: '',
        programArea: '',
        contactHours: ''
      },
      fieldLabels: {
        employeeName: 'Employee Name',
        programArea: 'Program Area',
        contactHours: 'Contact Hours'
      }
    };
  }
};
</script>
```

---

## Environment Variables

Add to `.env`:

```env
# React
REACT_APP_API_URL=http://localhost:3000
REACT_APP_API_KEY=your_api_key

# Vue
VUE_APP_API_URL=http://localhost:3000
VUE_APP_API_KEY=your_api_key
```

---

## Testing Checklist

- [ ] Test with form context - verify enhanced responses
- [ ] Test without form context - ensure backward compatibility
- [ ] Test quick questions functionality
- [ ] Test conversation memory across fields
- [ ] Test on mobile (responsive design)
- [ ] Test with different forms
- [ ] Verify session management

---

## Optional Enhancements

### 1. Field-Specific Help Icons
Add inline help icons next to each field that open the chat with a pre-filled question.

### 2. Smart Suggestions
Analyze common questions per field and show them as suggestions.

### 3. Form Validation Help
When validation fails, automatically open help for that field.

### 4. Progress Tracking
Show which fields the user has asked about and mark them as understood.

### 5. Multi-language Support
Detect user language and provide help in their preferred language.

---

## Deployment Notes

1. Ensure backend changes are deployed first
2. Update frontend environment variables
3. Test in staging environment
4. Monitor form completion rates
5. Collect feedback on help quality
6. Adjust system prompts based on common questions
