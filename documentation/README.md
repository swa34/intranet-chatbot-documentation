# Intranet Chatbot Documentation

Welcome to the comprehensive documentation for the Intranet Chatbot project. This documentation is organized by category to help you quickly find the information you need.

## 🚀 **NEW: Comprehensive Developer's Guide**

**⭐ START HERE** for complete developer onboarding: **[DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)**

This guide includes:
- 5-minute quick start
- Complete architecture overview with diagrams
- API reference with examples
- Development workflows
- Code recipes and examples
- Troubleshooting guide

---

## 📚 Quick Navigation

### New to the Project?
**Developers**: Start with [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)

**Users**: Start here:
1. [Command Reference](guides/COMMAND_REFERENCE.md) - All commands you need to know
2. [Caching Guide](guides/CACHING_GUIDE.md) - How to manage the cache system
3. [Testing Guide](guides/TESTING_GUIDE.md) - For testers and department leads

### For Developers
**🎯 Essential Developer Docs:**
1. **[DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)** - Complete developer documentation
2. [Quick Start](guides/QUICK_START.md) - Get running in 5 minutes
3. [Code Organization](architecture/CODE_ORGANIZATION.md) - Understand the codebase
4. [API Reference](guides/API_REFERENCE.md) - Complete API documentation
5. [System Overview](architecture/SYSTEM_OVERVIEW.md) - Architecture with diagrams
6. [Development Workflow](guides/DEVELOPMENT_WORKFLOW.md) - Daily development practices
7. [Contributing](guides/CONTRIBUTING.md) - How to contribute

**🏗️ Architecture & Technical:**
1. [RAG Pipeline](architecture/RAG_PIPELINE.md) - How retrieval and generation works
2. [Database Schema](architecture/DATABASE_SCHEMA.md) - Complete database documentation
3. [Document Ingestion](architecture/DOCUMENT_INGESTION_PIPELINE.md) - Document processing pipeline
4. [Caching Architecture](architecture/CACHING_ARCHITECTURE.md) - Technical deep dive into caching
5. [Adding New Features](architecture/ADDING_NEW_FEATURES.md) - Extend the system

**🛠️ Practical Guides:**
1. [Code Recipes](guides/CODE_RECIPES.md) - Common code patterns
2. [Troubleshooting](guides/TROUBLESHOOTING.md) - Fix common issues
3. [Testing Strategy](testing/TESTING_STRATEGY.md) - Testing phases and procedures
4. [Performance Optimizations](optimization/PERFORMANCE_OPTIMIZATIONS_SUMMARY.md) - Current optimizations
5. [Deployment](setup/DEPLOYMENT.md) - Production deployment guide

---

## 📂 Documentation Structure

### 🎯 `/guides` - User-Facing Documentation
Practical guides for daily operations and usage.

| Document | Description | Audience |
|----------|-------------|----------|
| **[QUICK_START.md](guides/QUICK_START.md)** | **5-minute setup guide for new developers** | **Developers (NEW)** |
| **[API_REFERENCE.md](guides/API_REFERENCE.md)** | **Complete API documentation with curl examples** | **Developers (NEW)** |
| **[DEVELOPMENT_WORKFLOW.md](guides/DEVELOPMENT_WORKFLOW.md)** | **Daily development practices and workflows** | **Developers (NEW)** |
| **[CONTRIBUTING.md](guides/CONTRIBUTING.md)** | **How to contribute code and documentation** | **Developers (NEW)** |
| **[TROUBLESHOOTING.md](guides/TROUBLESHOOTING.md)** | **Common issues and solutions** | **Developers (NEW)** |
| **[CODE_RECIPES.md](guides/CODE_RECIPES.md)** | **Common code patterns and examples** | **Developers (NEW)** |
| [CACHING_GUIDE.md](guides/CACHING_GUIDE.md) | Complete cache management guide | Administrators, Developers |
| [COMMAND_REFERENCE.md](guides/COMMAND_REFERENCE.md) | Comprehensive reference of all npm scripts | All users |
| [TESTING_GUIDE.md](guides/TESTING_GUIDE.md) | Instructions for testers and feedback submission | Testers, Department Leads |
| [CLARIFICATION_MODE_USAGE.md](guides/CLARIFICATION_MODE_USAGE.md) | How to use clarification mode | End users |

### 🏗️ `/architecture` - Technical Documentation
In-depth technical details about system design and implementation.

| Document | Description | Key Topics |
|----------|-------------|------------|
| **[CODE_ORGANIZATION.md](architecture/CODE_ORGANIZATION.md)** | **Complete codebase structure and organization** | **File structure, Modules (NEW)** |
| **[SYSTEM_OVERVIEW.md](architecture/SYSTEM_OVERVIEW.md)** | **High-level architecture with Mermaid diagrams** | **Components, Data flow (NEW)** |
| **[RAG_PIPELINE.md](architecture/RAG_PIPELINE.md)** | **Detailed RAG implementation** | **Retrieval, Generation (NEW)** |
| **[DATABASE_SCHEMA.md](architecture/DATABASE_SCHEMA.md)** | **Complete PostgreSQL schema documentation** | **Tables, Indexes, Views (NEW)** |
| **[DOCUMENT_INGESTION_PIPELINE.md](architecture/DOCUMENT_INGESTION_PIPELINE.md)** | **How documents are processed and indexed** | **Crawling, Chunking, Embedding (NEW)** |
| **[ADDING_NEW_FEATURES.md](architecture/ADDING_NEW_FEATURES.md)** | **Guide to extending the system** | **Adding sources, endpoints (NEW)** |
| [CACHING_ARCHITECTURE.md](architecture/CACHING_ARCHITECTURE.md) | Complete technical documentation of the caching system | Redis, PostgreSQL, Performance metrics |
| [CONVERSATION_MEMORY.md](architecture/CONVERSATION_MEMORY.md) | Chat memory and context management implementation | Session handling, Memory persistence |
| [FEEDBACK_LEARNING_POSTGRES.md](architecture/FEEDBACK_LEARNING_POSTGRES.md) | PostgreSQL-based feedback and learning system | Database schema, Analytics |
| [FORM_HELP_IMPLEMENTATION.md](architecture/FORM_HELP_IMPLEMENTATION.md) | Form assistance feature implementation | Auto-fill, Help system |
| [LLM_RERANKING.md](architecture/LLM_RERANKING.md) | Advanced reranking using language models | Scoring algorithms, Optimization |
| [METADATA_FILTERING.md](architecture/METADATA_FILTERING.md) | Metadata-based search filtering system | Filter strategies, Performance |
| [ACRONYM_EXPANSION.md](architecture/ACRONYM_EXPANSION.md) | Automatic acronym detection and expansion | NLP processing, Dictionary management |
| [WORDPRESS_DOCUMENT_MAPPING.md](architecture/WORDPRESS_DOCUMENT_MAPPING.md) | WordPress content integration and mapping | API integration, Content sync |

### 🧪 `/testing` - Testing Documentation
Testing strategies, procedures, and guides.

| Document | Description | Focus Area |
|----------|-------------|------------|
| [TESTING_STRATEGY.md](testing/TESTING_STRATEGY.md) | Comprehensive testing phases, timeline, and optimization strategies | Project managers, Developers |
| [RERANKING_TESTING_GUIDE.md](testing/RERANKING_TESTING_GUIDE.md) | Specific testing procedures for the reranking system | QA engineers |

### ⚡ `/optimization` - Performance & Optimization
Performance improvements, migration plans, and optimization strategies.

| Document | Description | Status |
|----------|-------------|--------|
| [GPT5_OPTIMIZATION.md](optimization/GPT5_OPTIMIZATION.md) | GPT-5 model integration and optimization | Active |
| [PERFORMANCE_OPTIMIZATIONS_SUMMARY.md](optimization/PERFORMANCE_OPTIMIZATIONS_SUMMARY.md) | Summary of all performance improvements | Current |
| [PRODUCTION_MIGRATION_PLAN.md](optimization/PRODUCTION_MIGRATION_PLAN.md) | Detailed production deployment strategy | In Progress |
| [POSTGRES_FEEDBACK_ANALYSIS.md](optimization/POSTGRES_FEEDBACK_ANALYSIS.md) | Feedback system performance analysis | Monitoring |
| [SEVALLA_REDIS_CONFIG.md](optimization/SEVALLA_REDIS_CONFIG.md) | Sevalla platform Redis configuration | Production |
| [SEVALLA_REDIS_INTEGRATION.md](optimization/SEVALLA_REDIS_INTEGRATION.md) | Full Sevalla Redis integration guide | Production |

### 💼 `/justifications` - Business Documentation
Business cases, justifications, and priority rankings.

| Document | Description | Purpose |
|----------|-------------|---------|
| [OpenAI_API_Justification.md](justifications/OpenAI_API_Justification.md) | Business case for OpenAI API usage | Budget approval |
| [Pinecone_Subscription_Justification.md](justifications/Pinecone_Subscription_Justification.md) | Business case for Pinecone vector database | Subscription justification |
| [PRIORITY_RANKINGS.md](justifications/PRIORITY_RANKINGS.md) | Feature and task prioritization | Project planning |

### ⚙️ `/setup` - Setup and Configuration
Initial setup guides and configuration documentation.

| Document | Description | When to Use |
|----------|-------------|-------------|
| **[DEPLOYMENT.md](setup/DEPLOYMENT.md)** | **Production deployment to Sevalla and other platforms** | **Going to production (NEW)** |
| [AUTH_IMPLEMENTATION.md](setup/AUTH_IMPLEMENTATION.md) | Authentication system setup | Initial deployment |
| [GOOGLE_SHEETS_SETUP.md](setup/GOOGLE_SHEETS_SETUP.md) | Google Sheets integration setup | Analytics setup |
| [GOOGLE_SHEETS_USER_TRACKING.md](setup/GOOGLE_SHEETS_USER_TRACKING.md) | User tracking configuration | Analytics setup |
| [INTRANET_SETUP.md](setup/INTRANET_SETUP.md) | Complete intranet integration guide | Initial deployment |
| [SEVALLA_GOOGLE_SHEETS_TROUBLESHOOTING.md](setup/SEVALLA_GOOGLE_SHEETS_TROUBLESHOOTING.md) | Troubleshooting Google Sheets on Sevalla | Problem solving |

### 📦 `/archive` - Historical Documentation
Outdated, superseded, or completed documentation kept for reference.

Contains 19 files including:
- Original cache documentation (before consolidation)
- Original command references (before consolidation)
- Original testing documentation (before consolidation)
- Completed cleanup plans
- Superseded implementations
- Historical ColdFusion integration docs

---

## 🚀 Most Common Tasks

### For Daily Operations
1. **Run a crawl and ingest**: See [Command Reference](guides/COMMAND_REFERENCE.md#quick-start-commands)
2. **Clear cache**: See [Caching Guide](guides/CACHING_GUIDE.md#cache-management)
3. **Check performance**: See [Caching Guide](guides/CACHING_GUIDE.md#daily-commands)

### For Development
1. **Understand caching**: Start with [Caching Architecture](architecture/CACHING_ARCHITECTURE.md)
2. **Run tests**: See [Testing Strategy](testing/TESTING_STRATEGY.md)
3. **Optimize performance**: Review [Performance Optimizations](optimization/PERFORMANCE_OPTIMIZATIONS_SUMMARY.md)

### For Testing
1. **Join testing program**: Read [Testing Guide](guides/TESTING_GUIDE.md)
2. **Submit feedback**: Follow instructions in [Testing Guide](guides/TESTING_GUIDE.md#providing-feedback)
3. **Report issues**: Use the feedback form at the testing URL

---

## 📊 Documentation Statistics

| Category | Files | Last Updated |
|----------|-------|--------------|
| **Developer Guide** | **1 (main hub)** | **October 2025 (NEW)** |
| Guides | **10 (+6 new)** | October 2025 |
| Architecture | **14 (+6 new)** | October 2025 |
| Testing | 2 | October 2025 |
| Optimization | 6 | October 2025 |
| Justifications | 3 | October 2025 |
| Setup | **6 (+1 new)** | October 2025 |
| Archive | 19 | Various |

**Total Active Documentation**: **42 files** (+14 new developer docs)
**Developer-focused documentation**: **14 comprehensive guides**
**Lines of documentation**: **~15,000+**

---

## 🔄 Recent Updates

- **October 12, 2025**: **✨ MAJOR UPDATE - Comprehensive Developer Documentation**
  - **Added DEVELOPER_GUIDE.md** - Central hub for all developer documentation
  - **Created 14 new developer guides** covering:
    - Quick Start (5-minute setup)
    - Complete API Reference with curl examples
    - System Architecture with Mermaid diagrams
    - RAG Pipeline deep dive
    - Database Schema documentation
    - Development Workflow guide
    - Contributing guidelines
    - Code Recipes with examples
    - Troubleshooting guide
    - Document Ingestion Pipeline
    - Adding New Features guide
    - Deployment guide
    - Code Organization
  - **Total new content**: ~15,000 lines of comprehensive documentation

- **October 2025**: Major documentation reorganization
  - Consolidated caching documentation (6 files → 2 files)
  - Consolidated command references (2 files → 1 file)
  - Consolidated testing documentation (5 files → 2 files)
  - Created categorical folder structure
  - Archived outdated documentation

---

## 📝 Contributing

When adding new documentation:
1. Place it in the appropriate category folder
2. Update this README with a description
3. Follow the existing naming conventions
4. Include a clear description and table of contents in your document
5. Archive old versions rather than deleting them

---

## 🤝 Support

For questions about the documentation:
- Technical questions: Contact the development team
- Testing questions: See [Testing Guide](guides/TESTING_GUIDE.md)
- General questions: Refer to [Command Reference](guides/COMMAND_REFERENCE.md)

---

*Documentation last reorganized: October 2025*