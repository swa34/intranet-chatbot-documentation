# Contributing Guide
**How to contribute to the CAES Intranet Help Bot project**

Thank you for considering contributing to this project! This guide will help you get started.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Process](#development-process)
- [Code Style Guidelines](#code-style-guidelines)
- [Testing Requirements](#testing-requirements)
- [Pull Request Process](#pull-request-process)
- [Commit Message Guidelines](#commit-message-guidelines)

---

## Code of Conduct

- Be respectful and professional
- Focus on constructive feedback
- Help newcomers feel welcome
- Collaborate openly

---

## Getting Started

### 1. Fork and Clone

```bash
# Fork the repository on GitHub
# Then clone your fork
git clone https://github.com/YOUR_USERNAME/CAES-INTRANET-HELP-BOT.git
cd CAES-INTRANET-HELP-BOT
```

### 2. Set Up Development Environment

```bash
# Install dependencies
npm install

# Copy environment variables
cp .env.example .env

# Edit .env with your API keys
```

See [Quick Start Guide](QUICK_START.md) for detailed setup.

### 3. Create a Branch

```bash
# Create a feature branch
git checkout -b feature/your-feature-name

# Or a bugfix branch
git checkout -b fix/bug-description
```

**Branch naming conventions**:
- `feature/` - New features
- `fix/` - Bug fixes
- `docs/` - Documentation changes
- `refactor/` - Code refactoring
- `test/` - Adding tests
- `chore/` - Maintenance tasks

---

## Development Process

### 1. Make Your Changes

- Write clean, readable code
- Follow existing code style
- Add comments for complex logic
- Update documentation if needed

### 2. Test Your Changes

```bash
# Start the server
npm run dev

# Test manually
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "test"}'

# Run tests (if available)
npm test
```

### 3. Commit Your Changes

```bash
git add .
git commit -m "feat: add new feature"
```

See [Commit Message Guidelines](#commit-message-guidelines) below.

### 4. Push and Create Pull Request

```bash
git push origin feature/your-feature-name
```

Then create a pull request on GitHub.

---

## Code Style Guidelines

### JavaScript Style

**Formatting**:
- **Indentation**: 2 spaces
- **Quotes**: Single quotes `'hello'`
- **Semicolons**: Always use
- **Line length**: ~100 characters (soft limit)

**Naming**:
```javascript
// camelCase for variables and functions
const userInput = 'hello';
function getUserData() {}

// PascalCase for classes
class UserManager {}

// SCREAMING_SNAKE_CASE for constants
const MAX_RETRIES = 3;
const API_ENDPOINT = 'https://api.example.com';
```

**Example**:
```javascript
// ✅ Good
async function retrieveRelevantChunks(query, topK = 8) {
  const embedRes = await openai.embeddings.create({
    model: process.env.EMBED_MODEL,
    input: query
  });

  return embedRes.data[0].embedding;
}

// ❌ Bad
async function retrieve_relevant_chunks(query,topK=8){
  const embedRes=await openai.embeddings.create({model:process.env.EMBED_MODEL,input:query});
  return embedRes.data[0].embedding
}
```

### File Structure

**Keep files focused**:
- One main responsibility per file
- Split large files (>500 lines) into modules
- Group related functions

**Export patterns**:
```javascript
// ✅ Named exports (preferred)
export function myFunction() {}
export const MY_CONSTANT = 42;

// ✅ Default export for main class
export default class MyClass {}
```

### Error Handling

**Always handle errors**:
```javascript
// ✅ Good
try {
  const result = await riskyOperation();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  throw error; // Re-throw if caller needs to handle
}

// ❌ Bad - No error handling
const result = await riskyOperation();
```

### Comments

**Use JSDoc for functions**:
```javascript
/**
 * Retrieve relevant document chunks for a query
 * @param {string} query - User's question
 * @param {number} topK - Number of results to return
 * @returns {Promise<Object>} Retrieved chunks with scores
 */
export async function retrieveRelevantChunks(query, topK = 8) {
  // Implementation
}
```

**Add comments for complex logic**:
```javascript
// Check if scores are too similar (within 0.05) - indicates uncertainty
if (Math.abs(topScore - thirdScore) < 0.05) {
  // Trigger LLM re-ranking for better accuracy
  await llmRerank(matches, query);
}
```

---

## Testing Requirements

### Manual Testing

Before submitting a pull request:

1. **Test the main flow**:
```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-key" \
  -d '{"message": "How do I report in GaCounts?"}'
```

2. **Test error cases**:
   - Missing API key
   - Invalid input
   - Empty message

3. **Test edge cases**:
   - Very long questions
   - Special characters
   - Multi-turn conversations

### Automated Testing (Future)

When test suite is implemented:
```bash
npm test
npm run test:coverage
```

---

## Pull Request Process

### 1. Before Submitting

- [ ] Code follows style guidelines
- [ ] All tests pass
- [ ] Documentation updated
- [ ] No console.log statements (use proper logging)
- [ ] No commented-out code
- [ ] No merge conflicts

### 2. Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Code refactoring
- [ ] Performance improvement

## Testing
How did you test this?

## Screenshots (if applicable)
Add screenshots if UI changed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-reviewed the code
- [ ] Commented complex code
- [ ] Updated documentation
- [ ] No breaking changes (or documented)
```

### 3. Review Process

- Maintainer will review within 2-3 days
- Address feedback promptly
- Make requested changes
- Request re-review when ready

### 4. After Approval

- Maintainer will merge
- Delete your branch after merge
- Pull latest changes:
```bash
git checkout main
git pull upstream main
```

---

## Commit Message Guidelines

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type

- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style (formatting, no logic change)
- **refactor**: Code refactoring
- **perf**: Performance improvement
- **test**: Adding tests
- **chore**: Maintenance (dependencies, config)

### Scope (Optional)

- `rag`: RAG pipeline
- `cache`: Caching system
- `auth`: Authentication
- `api`: API endpoints
- `docs`: Documentation

### Examples

```bash
# Feature
git commit -m "feat(rag): add LLM re-ranking for uncertain results"

# Bug fix
git commit -m "fix(cache): correct Redis connection pool leak"

# Documentation
git commit -m "docs: add troubleshooting guide for Pinecone errors"

# Refactor
git commit -m "refactor(api): extract feedback logic to separate module"
```

### Multi-line Commits

```bash
git commit -m "feat(rag): add feedback learning system

- Implement score adjustment based on user ratings
- Add PostgreSQL storage for feedback data
- Create analytics views for feedback insights

Closes #42"
```

---

## Common Contribution Areas

### 1. Adding New Document Sources

See [Adding New Features](../architecture/ADDING_NEW_FEATURES.md#adding-a-new-document-source)

### 2. Improving RAG Pipeline

- Experiment with different reranking strategies
- Add new metadata filters
- Improve chunking algorithm

### 3. Enhancing UI

- Improve chat interface design
- Add new visualizations
- Improve mobile responsiveness

### 4. Documentation

- Fix typos and errors
- Add more examples
- Create tutorials
- Translate documentation

### 5. Performance Optimization

- Profile and optimize slow queries
- Reduce memory usage
- Improve caching strategies

---

## Questions?

- **Check documentation** first
- **Search existing issues** on GitHub
- **Ask in discussions** for general questions
- **Open an issue** for bugs or feature requests

---

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.

---

## Related Documentation

- [Development Workflow](DEVELOPMENT_WORKFLOW.md)
- [Code Recipes](CODE_RECIPES.md)
- [Adding New Features](../architecture/ADDING_NEW_FEATURES.md)

---

**Last Updated**: October 2025
