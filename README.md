# 🎓 DP-800: Developing AI-Enabled Database Solutions

[![Exam](https://img.shields.io/badge/Exam-DP--800-blue?style=for-the-badge&logo=microsoft)](https://learn.microsoft.com/credentials/certifications/exams/dp-800/)
[![Level](https://img.shields.io/badge/Level-Associate-green?style=for-the-badge)](https://learn.microsoft.com/credentials/certifications/developing-ai-enabled-database-solutions/)
[![Platform](https://img.shields.io/badge/Platform-Azure%20SQL%20%7C%20SQL%20Server%20%7C%20Fabric-0078D4?style=for-the-badge&logo=microsoft-azure)](https://learn.microsoft.com/azure/azure-sql/)

> **Skills measured as of March 12, 2026**

---

## 📋 Table of Contents

- [👤 Audience Profile](#-audience-profile)
- [📊 Skills at a Glance](#-skills-at-a-glance)
- [📁 Study Resources in this Repo](#-study-resources-in-this-repo)
- [🔧 Exam Domains](#-exam-domains)
- [📚 Quick Links](#-quick-links)
- [💡 Study Tips](#-study-tips)

---

## 👤 Audience Profile

As a candidate for DP-800, you should have **subject matter expertise** in:

- 🗄️ **Designing and developing AI-enabled database solutions** across Microsoft SQL platforms
- 💻 **Writing T-SQL code** and developing databases in Microsoft SQL Server, Azure SQL, and Fabric SQL
- 🤖 **AI concepts** — embeddings, vectors, models, RAG patterns
- 🔁 **CI/CD practices** in GitHub with AI-assisted development tools

### Your Responsibilities

✅ Design and develop database solutions (structured + semi-structured data)  
✅ Integrate AI features into modern enterprise applications  
✅ Secure, optimize, and deploy database solutions  
✅ Implement AI capabilities (embeddings, vector search, RAG)  

---

## 📊 Skills at a Glance

| Domain | Weight | Focus Areas |
|--------|--------|-------------|
| **🏗️ Design and Develop Database Solutions** | 35–40% | Tables, programmability, T-SQL, AI-assisted tools |
| **🔒 Secure, Optimize, and Deploy** | 35–40% | Security, performance, CI/CD, Azure services |
| **🤖 Implement AI Capabilities** | 25–30% | Embeddings, vector search, intelligent search, RAG |

---

## 📁 Study Resources in this Repo

| File | Topic | Domain |
|------|-------|--------|
| [`write_advanced_tsql.md`](./write_advanced_tsql.md) | Write Advanced T-SQL Code | Design & Develop (35–40%) |
| [`intelligent_search.md`](./intelligent_search.md) | Design and Implement Intelligent Search | AI Capabilities (25–30%) |
| [`design_implement_rag.md`](./design_implement_rag.md) | Design and Implement RAG | AI Capabilities (25–30%) |

---

## 🔧 Exam Domains

### 1. Design and Develop Database Solutions (35–40%)

#### 🏗️ Design and Implement Database Objects
- Tables: data types, indexes, columnstore indexes
- Specialized tables: in-memory, temporal, external, ledger, graph
- JSON columns and indexes
- Constraints: PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, DEFAULT
- SEQUENCES, partitioning for tables and indexes

#### 💻 Implement Programmability Objects
- Views, scalar functions, table-valued functions
- Stored procedures, triggers

#### 📝 [Write Advanced T-SQL Code](./write_advanced_tsql.md)
- CTEs (recursive and non-recursive)
- Window functions (ROW_NUMBER, RANK, LAG, LEAD, SUM OVER, etc.)
- JSON functions (JSON_OBJECT, JSON_ARRAY, JSON_ARRAYAGG, OPENJSON, FOR JSON)
- Regular expressions (REGEXP_LIKE, REGEXP_REPLACE, REGEXP_MATCHES, etc.)
- Fuzzy string matching (EDIT_DISTANCE, EDIT_DISTANCE_SIMILARITY, JARO_WINKLER_DISTANCE)
- Graph queries with MATCH operator
- Correlated queries (EXISTS, NOT EXISTS)
- Error handling (TRY...CATCH, THROW, XACT_STATE)

#### 🤖 Design and Implement SQL Solutions Using AI-Assisted Tools
- GitHub Copilot and Microsoft Copilot in Fabric
- Model Context Protocol (MCP) tool options
- GitHub Copilot instruction files
- MCP server endpoints (SQL Server, Fabric lakehouse)

---

### 2. Secure, Optimize, and Deploy Database Solutions (35–40%)

#### 🔒 Implement Data Security and Compliance
- Always Encrypted, column-level encryption
- Dynamic Data Masking
- Row-Level Security (RLS)
- Object-level permissions
- Secure database access (passwordless / Managed Identity)
- Auditing, secure model endpoints, GraphQL/REST/MCP endpoint security

#### ⚡ Optimize Database Performance
- Database configurations
- Transaction isolation levels and concurrency controls
- Query execution plans, DMVs, Query Store
- Blocking and deadlock identification/resolution

#### 🔁 Implement CI/CD Using SQL Database Projects
- Testing strategy (unit tests, integration tests)
- SQL Database Projects (SDK-style models)
- Source control, branching, pull requests, conflict resolution
- Secrets management, schema drift detection
- Deployment pipeline controls (branching policies, triggers, code owners)

#### ☁️ Integrate SQL Solutions with Azure Services
- Data API Builder (DAB) configuration
- REST and GraphQL endpoints
- Change event streaming (CES), CDC, Change Tracking, Azure Functions
- Azure Monitor, Application Insights, Log Analytics

---

### 3. Implement AI Capabilities in Database Solutions (25–30%)

#### 🧠 Design and Implement Models and Embeddings
- Evaluate external models (multimodal, multilanguage, sizes)
- Create and manage external models
- Embedding maintenance methods (triggers, CDC, Azure Functions, Microsoft Foundry)
- Choose columns to include in embeddings
- Design chunks for embeddings
- Generate embeddings

#### 🔍 [Design and Implement Intelligent Search](./intelligent_search.md)
- Choose from full-text, semantic vector, and hybrid search
- Implement full-text search (CONTAINS, FREETEXT, CONTAINSTABLE, FREETEXTTABLE)
- Design for vector data (VECTOR data type, vector indexes, size)
- Vector functions: VECTOR_NORMALIZE, VECTOR_DISTANCE, VECTORPROPERTY, VECTOR_SEARCH
- ANN vs ENN for vector search
- Vector index types (DiskANN, IVFFlat) and metrics (cosine, euclidean, dot)
- Implement vector search and hybrid search
- Reciprocal Rank Fusion (RRF)
- Evaluate vector and hybrid search performance

#### 🤖 [Design and Implement RAG](./design_implement_rag.md)
- Identify use cases for RAG
- Create prompts using `sp_invoke_external_rest_endpoint`
- Convert structured data to JSON for LLM processing
- Send results to language model (Azure OpenAI)
- Extract language model responses (JSON_VALUE)

---

## 📚 Quick Links

| Resource | Link |
|----------|------|
| Official exam page | [learn.microsoft.com/credentials/certifications/exams/dp-800](https://learn.microsoft.com/credentials/certifications/exams/dp-800/) |
| Study guide (official) | [learn.microsoft.com/.../study-guides/dp-800](https://learn.microsoft.com/credentials/certifications/resources/study-guides/dp-800) |
| Certification page | [Developing AI-Enabled Database Solutions](https://learn.microsoft.com/credentials/certifications/developing-ai-enabled-database-solutions/) |
| Azure SQL docs | [learn.microsoft.com/azure/azure-sql](https://learn.microsoft.com/azure/azure-sql/) |
| SQL Server 2022 docs | [learn.microsoft.com/sql/sql-server](https://learn.microsoft.com/sql/sql-server/) |
| Fabric SQL database | [learn.microsoft.com/fabric/database/sql](https://learn.microsoft.com/fabric/database/sql/) |
| Azure OpenAI docs | [learn.microsoft.com/azure/ai-services/openai](https://learn.microsoft.com/azure/ai-services/openai/) |
| Practice assessment | [Microsoft Learn Practice Assessment](https://learn.microsoft.com/credentials/certifications/exams/dp-800/practice/assessment?assessment-type=practice&assessmentId=90&practice-assessment-type=certification) |

---

## 💡 Study Tips

1. **Set up a free Azure SQL Database** — many features (REGEXP_, fuzzy matching, vector search) are Azure SQL / Fabric SQL only, not SQL Server on-prem
2. **Practice the complete RAG pipeline** in T-SQL: embed → vector search → FOR JSON → `sp_invoke_external_rest_endpoint` → JSON_VALUE
3. **Learn DiskANN index creation** — syntax and metric options are exam-tested
4. **Understand VECTOR_SEARCH vs VECTOR_DISTANCE** — VECTOR_SEARCH uses ANN index; VECTOR_DISTANCE is brute-force ENN
5. **Know the RRF formula** — k=60 default, why it works (no score normalization needed)
6. **Practice CTEs, window functions, and JSON functions** — high-frequency T-SQL topics

---

*Last updated: September 2026 | DP-800 Skills measured as of March 12, 2026*
