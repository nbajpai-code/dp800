# 🤖 Design and Implement Retrieval-Augmented Generation (RAG)

> **DP-800 Exam Domain:** Implement AI Capabilities in Database Solutions (25–30%)  
> **Subtopic:** Design and Implement Retrieval-Augmented Generation (RAG)  
> **Skills Measured (as of March 12, 2026)**

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [RAG Architecture Overview](#-rag-architecture-overview)
- [1. Identify Use Cases for RAG](#1-identify-use-cases-for-rag)
- [2. Create a Prompt Using sp_invoke_external_rest_endpoint](#2-create-a-prompt-using-sp_invoke_external_rest_endpoint)
- [3. Convert Structured Data to JSON for LLM Processing](#3-convert-structured-data-to-json-for-language-model-processing)
- [4. Send Results to Language Model](#4-send-results-to-language-model)
- [5. Extract Language Model Responses](#5-extract-language-model-responses)
- [End-to-End RAG Example](#-end-to-end-rag-in-sql--complete-example)
- [RAG Design Patterns](#-rag-design-patterns)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Identify use cases for RAG | ⭐⭐ |
| 2 | Create a prompt using `sp_invoke_external_rest_endpoint` | ⭐⭐⭐⭐ |
| 3 | Convert structured data to JSON for language model processing | ⭐⭐⭐ |
| 4 | Send results to language model | ⭐⭐⭐⭐ |
| 5 | Extract language model responses | ⭐⭐⭐ |

---

## 🏗️ RAG Architecture Overview

RAG enriches LLM responses with **factual, current, private data** retrieved from your database — solving the LLM's lack of domain knowledge and knowledge cutoff limitations.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      RAG Pipeline in SQL                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. USER QUERY                                                      │
│     "What is the return policy for electronics?"                    │
│                   ↓                                                 │
│  2. RETRIEVAL (SQL Vector / Hybrid Search)                          │
│     - Embed query → query vector                                    │
│     - VECTOR_SEARCH → top-K relevant documents                      │
│     - Optionally: FTS + RRF for hybrid retrieval                    │
│                   ↓                                                 │
│  3. CONTEXT CONSTRUCTION (FOR JSON / JSON_ARRAYAGG)                 │
│     - Format retrieved docs as JSON                                 │
│     - Build system + user prompt                                    │
│                   ↓                                                 │
│  4. LLM CALL (sp_invoke_external_rest_endpoint)                     │
│     - POST to Azure OpenAI / OpenAI API                             │
│     - Send prompt with context                                      │
│                   ↓                                                 │
│  5. RESPONSE EXTRACTION (JSON_VALUE)                                │
│     - Parse JSON response                                           │
│     - Extract generated text answer                                 │
│     - Return to application or log for audit                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Identify Use Cases for RAG

### 🎯 When to Use RAG

| Use Case | Why RAG? |
|----------|----------|
| **Customer support chatbot** | LLM answers based on your product docs/policies |
| **Internal knowledge base Q&A** | LLM searches and answers from company wiki |
| **Product recommendations** | Retrieve similar products, LLM explains why |
| **Document summarization** | Retrieve relevant sections, LLM summarizes |
| **SQL query explanation** | Retrieve query + schema, LLM explains results |
| **Compliance checking** | Retrieve rules, LLM checks record compliance |
| **Medical/legal research** | Retrieve domain documents, LLM synthesizes answer |

### ❌ When NOT to Use RAG

| Scenario | Better Approach |
|----------|----------------|
| General knowledge question | Pure LLM (no retrieval needed) |
| Exact structured query | SQL query directly |
| Simple CRUD operations | Direct database access |
| Real-time data aggregations | SQL + reporting tools |

### 🔑 RAG vs Fine-Tuning

| | RAG | Fine-Tuning |
|--|-----|-------------|
| **Data freshness** | ✅ Real-time (live DB) | ❌ Baked into model weights |
| **Cost** | ✅ Cheaper (no GPU training) | ❌ Expensive to train |
| **Control** | ✅ Retrieval visible and auditable | ❌ Black box |
| **Hallucination risk** | ✅ Lower (grounded in retrieved docs) | ❌ Higher |
| **Use for** | Dynamic, frequently changing data | Consistent style/behavior/format |

---

## 2. Create a Prompt Using `sp_invoke_external_rest_endpoint`

### 🔑 What is `sp_invoke_external_rest_endpoint`?

`sp_invoke_external_rest_endpoint` is a SQL stored procedure that allows T-SQL code to make **HTTPS REST API calls directly from within SQL** — enabling AI model calls without external application code.

**Platforms:** Azure SQL Database, SQL databases in Microsoft Fabric

```sql
EXEC sp_invoke_external_rest_endpoint
    @url        = N'https://...',                    -- REST endpoint URL
    @method     = N'POST',                           -- HTTP method
    @headers    = N'{"Content-Type":"application/json", ...}',
    @payload    = N'{...}',                          -- JSON request body
    @credential = N'MyDatabaseScopedCredential',     -- Auth credential
    @response   = @response_json OUTPUT;             -- OUTPUT: full response JSON
```

### ⚙️ Setup: Database Scoped Credential for Azure OpenAI

```sql
-- Step 1: Create a master key (required for credentials)
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword123!';

-- Step 2a: API Key authentication (development)
CREATE DATABASE SCOPED CREDENTIAL AzureOpenAI_Credential
    WITH IDENTITY = 'HTTPEndpointHeaders',
    SECRET = '{"api-key": "YOUR_AZURE_OPENAI_API_KEY"}';

-- Step 2b: Managed Identity (production — passwordless, preferred)
CREATE DATABASE SCOPED CREDENTIAL AzureOpenAI_MI
    WITH IDENTITY = 'Managed Identity',
    SECRET = '{"resourceid": "https://cognitiveservices.azure.com"}';
-- Requires: Azure SQL Managed Identity must have 'Cognitive Services User' RBAC role
```

### 📐 OpenAI Chat Completion Prompt Format

OpenAI-compatible APIs use a **chat completion** message format:

```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant. Answer ONLY based on the provided context."
    },
    {
      "role": "user",
      "content": "Context: [retrieved documents here]\n\nQuestion: [user question here]"
    }
  ],
  "max_tokens": 500,
  "temperature": 0.3
}
```

```sql
-- Construct the prompt payload in T-SQL using JSON_OBJECT + JSON_ARRAY
DECLARE @user_question NVARCHAR(1000) = 'What is the return policy for electronics?';
DECLARE @context NVARCHAR(MAX);  -- Populated from vector search (see Section 3)

DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT(
    'model': 'gpt-4o',
    'messages': JSON_ARRAY(
        JSON_OBJECT(
            'role': 'system',
            'content': 'You are a helpful assistant. Answer ONLY based on the provided context. If the context does not contain the answer, say "I don''t know."'
        ),
        JSON_OBJECT(
            'role': 'user',
            'content': 'Context:' + CHAR(10) + @context + CHAR(10) + CHAR(10) + 'Question: ' + @user_question
        )
    ),
    'max_tokens': 800,
    'temperature': 0.3
);
```

### 🔧 Prompt Engineering Best Practices

```
System prompt should:
✅ Define the assistant's role and constraints
✅ Instruct to answer ONLY from context (reduces hallucination)
✅ Handle no-answer case ("say I don't know")
✅ Define output format if structured response is needed

User prompt should:
✅ Include retrieved context (formatted clearly as JSON or plain text)
✅ Include the actual user question
✅ Clearly separate context from question (use labels/headings)
✅ Keep within context window limits (truncate long docs)
```

---

## 3. Convert Structured Data to JSON for Language Model Processing

### 🔑 Why JSON?

LLMs receive text input. SQL result sets must be **serialized to text (JSON)** before inclusion in the prompt context.

### 📐 FOR JSON PATH — Converting Retrieved Documents

```sql
DECLARE @query_vector VECTOR(1536) = '[...]';
DECLARE @context NVARCHAR(MAX);

-- Option 1: FOR JSON PATH (controlled structure, most readable)
SELECT @context = (
    SELECT TOP 5
        d.DocumentTitle  AS 'document.title',
        LEFT(d.DocumentContent, 800)
            + CASE WHEN LEN(d.DocumentContent) > 800 THEN '...' ELSE '' END
            AS 'document.content',
        d.Category       AS 'document.category'
    FROM PolicyDocuments d
    ORDER BY VECTOR_DISTANCE('cosine', d.Embedding, @query_vector) ASC
    FOR JSON PATH, ROOT('context')
);

-- Option 2: JSON_ARRAYAGG (inline aggregation)
SELECT @context = (
    SELECT JSON_ARRAYAGG(
        JSON_OBJECT(
            'title':   d.DocumentTitle,
            'content': LEFT(d.Content, 500)
        )
    )
    FROM (
        SELECT TOP 5 DocumentTitle, Content
        FROM PolicyDocuments
        ORDER BY VECTOR_DISTANCE('cosine', Embedding, @query_vector) ASC
    ) d
);
```

### 📐 Structured Data: Customer Order History for LLM

```sql
DECLARE @customer_id INT = 12345;
DECLARE @customer_context NVARCHAR(MAX);

SELECT @customer_context = (
    SELECT
        c.CustomerName,
        c.CustomerTier,
        (
            SELECT TOP 5
                o.OrderID    AS 'id',
                o.OrderDate  AS 'date',
                o.OrderTotal AS 'total',
                o.Status     AS 'status'
            FROM Orders o
            WHERE o.CustomerID = c.CustomerID
            ORDER BY o.OrderDate DESC
            FOR JSON PATH
        ) AS RecentOrders
    FROM Customers c
    WHERE c.CustomerID = @customer_id
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
);
-- Result: {"CustomerName":"Alice","CustomerTier":"Gold","RecentOrders":[{...}]}
```

### 🎯 Context Length Management

```
LLM context window limits (approximate):
  GPT-4o:         128,000 tokens (~96,000 words)
  GPT-4o-mini:    128,000 tokens
  GPT-3.5-turbo:   16,000 tokens

1 token ≈ 4 characters ≈ 0.75 words

Best practices:
✅ Limit retrieved documents to TOP 3–5 (not all matches)
✅ Truncate very long documents (first N characters)
✅ Include only relevant columns in JSON (not SELECT *)
✅ Order by relevance — most relevant document first
```

```sql
-- Truncate long content for context window management
SELECT TOP 5
    DocumentTitle,
    LEFT(DocumentContent, 500) +
        CASE WHEN LEN(DocumentContent) > 500 THEN '...' ELSE '' END AS Content
FROM PolicyDocuments
ORDER BY VECTOR_DISTANCE('cosine', Embedding, @query_vector) ASC;
```

---

## 4. Send Results to Language Model

### 📐 Full Call to Azure OpenAI via sp_invoke_external_rest_endpoint

```sql
CREATE OR ALTER PROCEDURE usp_RagQuery
    @user_question NVARCHAR(1000),
    @query_vector  VECTOR(1536)
AS
BEGIN
    SET NOCOUNT ON;

    -- Step 1: Retrieve relevant documents via vector search
    DECLARE @context NVARCHAR(MAX);

    SELECT @context = (
        SELECT TOP 5
            d.Title         AS 'document.title',
            LEFT(d.Content, 1000) AS 'document.content',
            d.Source        AS 'document.source'
        FROM Documents d
        ORDER BY VECTOR_DISTANCE('cosine', d.Embedding, @query_vector) ASC
        FOR JSON PATH, ROOT('context')
    );

    -- Step 2: Build the prompt payload
    DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT(
        'model':       'gpt-4o',
        'messages':    JSON_ARRAY(
            JSON_OBJECT(
                'role':    'system',
                'content': 'You are an expert assistant. Answer questions ONLY based on the provided context. If the context does not contain the answer, respond with "I cannot find relevant information in the available documents."'
            ),
            JSON_OBJECT(
                'role':    'user',
                'content': CONCAT(
                    'Context documents:', CHAR(10),
                    @context, CHAR(10), CHAR(10),
                    'User question: ', @user_question
                )
            )
        ),
        'max_tokens':  800,
        'temperature': 0.3
    );

    -- Step 3: Call Azure OpenAI
    DECLARE @response NVARCHAR(MAX);
    DECLARE @openai_url NVARCHAR(500) =
        N'https://YOUR_RESOURCE.openai.azure.com' +
        N'/openai/deployments/gpt-4o/chat/completions?api-version=2024-10-21';

    EXEC sp_invoke_external_rest_endpoint
        @url        = @openai_url,
        @method     = N'POST',
        @headers    = N'{"Content-Type": "application/json"}',
        @credential = N'AzureOpenAI_Credential',
        @payload    = @payload,
        @response   = @response OUTPUT;

    -- Step 4: Return the raw response (parse in next step)
    SELECT @response AS RawResponse;
END;
```

### 📐 Calling Embeddings API to Vectorize User Query

```sql
-- Generate embedding for user query at runtime
CREATE OR ALTER PROCEDURE usp_GetEmbedding
    @input_text NVARCHAR(MAX),
    @embedding  VECTOR(1536) OUTPUT
AS
BEGIN
    DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT(
        'input': @input_text,
        'model': 'text-embedding-3-small'
    );

    DECLARE @response    NVARCHAR(MAX);
    DECLARE @embed_url   NVARCHAR(500) =
        N'https://YOUR_RESOURCE.openai.azure.com' +
        N'/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-10-21';

    EXEC sp_invoke_external_rest_endpoint
        @url        = @embed_url,
        @method     = N'POST',
        @headers    = N'{"Content-Type": "application/json"}',
        @credential = N'AzureOpenAI_Credential',
        @payload    = @payload,
        @response   = @response OUTPUT;

    -- Response format: {"result":{"data":[{"embedding":[0.023,...]}]}}
    DECLARE @embedding_json NVARCHAR(MAX) =
        JSON_QUERY(@response, '$.result.data[0].embedding');

    SET @embedding = CAST(@embedding_json AS VECTOR(1536));
END;
```

### 📊 Authentication Options

| Method | Credential Type | Best For |
|--------|----------------|----------|
| **API Key** | `HTTPEndpointHeaders` secret | Development, quick setup |
| **Managed Identity** | `Managed Identity` | Production (passwordless, no key rotation) |
| **Service Principal** | `SHARED ACCESS SIGNATURE` variant | CI/CD pipelines |

---

## 5. Extract Language Model Responses

### 🔑 Azure OpenAI Response Format

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Based on the provided context, the return policy for electronics is..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 450,
    "completion_tokens": 120,
    "total_tokens": 570
  }
}
```

> **Note:** `sp_invoke_external_rest_endpoint` wraps the full API response under a `result` property:
> ```json
> {"status": 200, "headers": {...}, "result": {<actual API response>}}
> ```

### 📐 Extracting the Response

```sql
-- @response is populated by sp_invoke_external_rest_endpoint
DECLARE @response NVARCHAR(MAX);  -- (from the stored procedure OUTPUT parameter)

-- Extract the LLM answer text
DECLARE @answer NVARCHAR(MAX) =
    JSON_VALUE(@response, '$.result.choices[0].message.content');

-- Extract token usage
DECLARE @prompt_tokens     INT = JSON_VALUE(@response, '$.result.usage.prompt_tokens');
DECLARE @completion_tokens INT = JSON_VALUE(@response, '$.result.usage.completion_tokens');
DECLARE @total_tokens      INT = JSON_VALUE(@response, '$.result.usage.total_tokens');

-- Extract finish reason:
-- 'stop'           = normal completion
-- 'length'         = truncated (increase max_tokens)
-- 'content_filter' = blocked by content policy
DECLARE @finish_reason NVARCHAR(50) =
    JSON_VALUE(@response, '$.result.choices[0].finish_reason');

-- Check HTTP status
DECLARE @http_status INT = JSON_VALUE(@response, '$.status');

SELECT
    @answer         AS Answer,
    @finish_reason  AS FinishReason,
    @total_tokens   AS TotalTokens,
    @http_status    AS HTTPStatus;
```

### 📐 Error Handling for LLM Responses

```sql
DECLARE @response    NVARCHAR(MAX);
DECLARE @http_status INT;
DECLARE @answer      NVARCHAR(MAX);

BEGIN TRY
    EXEC sp_invoke_external_rest_endpoint
        @url        = @openai_url,
        @method     = N'POST',
        @headers    = N'{"Content-Type": "application/json"}',
        @credential = N'AzureOpenAI_Credential',
        @payload    = @payload,
        @response   = @response OUTPUT;

    SET @http_status = JSON_VALUE(@response, '$.status');

    IF @http_status = 200
    BEGIN
        SET @answer = JSON_VALUE(@response, '$.result.choices[0].message.content');
        IF @answer IS NULL
            SET @answer = 'Response was filtered by content policy.';
        SELECT @answer AS Answer, 'Success' AS Status;
    END
    ELSE IF @http_status = 429
        SELECT 'Rate limit exceeded. Please retry.' AS Answer, 'RateLimited' AS Status;
    ELSE IF @http_status = 401
        SELECT 'Authentication failed. Check credentials.' AS Answer, 'AuthError' AS Status;
    ELSE
        SELECT 'Error: ' + ISNULL(JSON_VALUE(@response, '$.result.error.message'), 'Unknown error')
            AS Answer, 'Error' AS Status;

END TRY
BEGIN CATCH
    SELECT 'SQL Error: ' + ERROR_MESSAGE() AS Answer, 'SQLError' AS Status;
END CATCH;
```

### 📐 Structured JSON Output from LLM

```sql
-- Instruct LLM to return JSON, then parse it with JSON_VALUE
SET @payload = JSON_OBJECT(
    'model': 'gpt-4o',
    'messages': JSON_ARRAY(
        JSON_OBJECT(
            'role': 'system',
            'content': 'You are a data extraction assistant. Always respond with valid JSON matching this schema: {"sentiment":"positive|negative|neutral","confidence":0.0-1.0,"summary":"string"}'
        ),
        JSON_OBJECT('role': 'user', 'content': 'Analyze this review: ' + @review_text)
    ),
    'response_format': JSON_OBJECT('type': 'json_object'),  -- Force JSON output mode
    'temperature': 0.0
);

-- After calling the API, parse the structured response:
SELECT
    JSON_VALUE(@response, '$.result.choices[0].message.content.sentiment')  AS Sentiment,
    JSON_VALUE(@response, '$.result.choices[0].message.content.confidence') AS Confidence,
    JSON_VALUE(@response, '$.result.choices[0].message.content.summary')    AS Summary;
```

---

## 🏆 End-to-End RAG in SQL — Complete Example

```sql
-- Complete RAG stored procedure: Q&A over a product knowledge base
CREATE OR ALTER PROCEDURE usp_ProductQA
    @user_question NVARCHAR(1000)
AS
BEGIN
    SET NOCOUNT ON;

    -- ============================================================
    -- STEP 1: Embed the user question → query vector
    -- ============================================================
    DECLARE @query_vector VECTOR(1536);
    EXEC usp_GetEmbedding
        @input_text = @user_question,
        @embedding  = @query_vector OUTPUT;

    IF @query_vector IS NULL
    BEGIN
        SELECT 'Failed to generate embedding for the query.' AS Answer;
        RETURN;
    END;

    -- ============================================================
    -- STEP 2: Retrieve top-5 relevant documents via vector search
    -- ============================================================
    DECLARE @context NVARCHAR(MAX);

    SELECT @context = (
        SELECT TOP 5
            d.Title      AS 'doc.title',
            LEFT(d.Content, 800)
                + CASE WHEN LEN(d.Content) > 800 THEN '...' ELSE '' END AS 'doc.content',
            d.Category   AS 'doc.category',
            d.LastUpdated AS 'doc.updated'
        FROM ProductDocuments d
        ORDER BY VECTOR_DISTANCE('cosine', d.Embedding, @query_vector) ASC
        FOR JSON PATH, ROOT('retrieved_documents')
    );

    -- ============================================================
    -- STEP 3: Build the RAG prompt
    -- ============================================================
    DECLARE @system_prompt NVARCHAR(2000) =
        'You are a product support specialist. ' +
        'Answer the user''s question using ONLY the information in the retrieved documents. ' +
        'If the documents do not contain the answer, say "I cannot find this information in our documentation." ' +
        'Be concise and accurate. Cite the document title when relevant.';

    DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT(
        'model':       'gpt-4o',
        'messages':    JSON_ARRAY(
            JSON_OBJECT('role': 'system', 'content': @system_prompt),
            JSON_OBJECT('role': 'user',
                'content': CONCAT(
                    'Retrieved context:', CHAR(10), @context, CHAR(10), CHAR(10),
                    'Question: ', @user_question
                )
            )
        ),
        'max_tokens':  600,
        'temperature': 0.2
    );

    -- ============================================================
    -- STEP 4: Call Azure OpenAI
    -- ============================================================
    DECLARE @response   NVARCHAR(MAX);
    DECLARE @openai_url NVARCHAR(500) =
        N'https://YOUR_OPENAI_RESOURCE.openai.azure.com' +
        N'/openai/deployments/gpt-4o/chat/completions?api-version=2024-10-21';

    BEGIN TRY
        EXEC sp_invoke_external_rest_endpoint
            @url        = @openai_url,
            @method     = N'POST',
            @headers    = N'{"Content-Type": "application/json"}',
            @credential = N'AzureOpenAI_Credential',
            @payload    = @payload,
            @response   = @response OUTPUT;
    END TRY
    BEGIN CATCH
        SELECT 'API call failed: ' + ERROR_MESSAGE() AS Answer;
        RETURN;
    END CATCH;

    -- ============================================================
    -- STEP 5: Extract and return the response
    -- ============================================================
    DECLARE @http_status   INT          = JSON_VALUE(@response, '$.status');
    DECLARE @answer        NVARCHAR(MAX) = JSON_VALUE(@response, '$.result.choices[0].message.content');
    DECLARE @finish_reason NVARCHAR(50)  = JSON_VALUE(@response, '$.result.choices[0].finish_reason');
    DECLARE @total_tokens  INT          = JSON_VALUE(@response, '$.result.usage.total_tokens');

    IF @http_status <> 200
    BEGIN
        SELECT JSON_VALUE(@response, '$.result.error.message') AS Answer;
        RETURN;
    END;

    -- Log the interaction for audit
    INSERT INTO RAGInteractionLog
        (UserQuestion, ContextUsed, Answer, TotalTokens, FinishReason, CreatedAt)
    VALUES
        (@user_question, @context, @answer, @total_tokens, @finish_reason, GETUTCDATE());

    SELECT
        @answer        AS Answer,
        @total_tokens  AS TokensUsed,
        @finish_reason AS FinishReason;
END;

-- Usage:
-- EXEC usp_ProductQA @user_question = 'What is the warranty period for the ProMax 3000?';
```

---

## 📐 RAG Design Patterns

### Pattern 1: Simple RAG (Basic)
```
Query → Embed → Vector Search → LLM → Answer
```

### Pattern 2: Hybrid RAG (Production)
```
Query → [Embed + FTS keywords] → [Vector Search + FTS] → RRF Merge → LLM → Answer
```

### Pattern 3: Multi-Turn RAG (Conversational)
```sql
-- Store conversation history for multi-turn context
CREATE TABLE ConversationHistory (
    SessionID   UNIQUEIDENTIFIER,
    TurnNumber  INT,
    Role        NVARCHAR(20),    -- 'user' or 'assistant'
    Content     NVARCHAR(MAX),
    CreatedAt   DATETIME2 DEFAULT GETUTCDATE()
);

-- Include history in messages array:
DECLARE @history NVARCHAR(MAX);
SELECT @history = (
    SELECT role, content
    FROM ConversationHistory
    WHERE SessionID = @session_id
    ORDER BY TurnNumber
    FOR JSON PATH
);
-- Then append new user message and retrieved context to history
```

### Pattern 4: Re-ranking RAG
```
Query → Vector Search (top-20) → SQL re-rank by metadata → LLM (top-5) → Answer
```

### 📋 RAG Quality Checklist

```
Retrieval quality:
✅ Using appropriate embedding model for your domain
✅ Chunking documents at semantic boundaries (not mid-sentence)
✅ Storing metadata (date, category, source) for pre-filtering
✅ Refreshing embeddings when source documents change

Generation quality:
✅ System prompt instructs grounding in context only
✅ System prompt handles no-answer case explicitly
✅ temperature ≤ 0.3 for factual Q&A (lower = more deterministic)
✅ max_tokens sufficient for complete answers

Operational:
✅ Logging all interactions (question, context, answer, tokens)
✅ Monitoring token usage and costs
✅ Error handling for API failures, rate limits (HTTP 429)
✅ Using Managed Identity for production credentials
✅ Testing finish_reason = 'length' (increase max_tokens if truncated)
```

---

## ❓ Practice Questions

**Q1:** A company has a private product knowledge base. Chatbot users get wrong answers because the LLM doesn't know company products. What is the BEST solution?
- A) Fine-tune the LLM on product documents
- B) **Implement RAG — retrieve relevant docs from DB, pass to LLM as context** ✅
- C) Add direct SQL queries to the chatbot
- D) Use Full-Text Search only (no LLM)

> **Explanation:** RAG grounds the LLM in your private, current data without expensive fine-tuning.

---

**Q2:** Which stored procedure enables T-SQL code to call Azure OpenAI's REST API directly from within Azure SQL?
- A) `sp_execute_external_script`
- B) `sp_send_dbmail`
- C) **`sp_invoke_external_rest_endpoint`** ✅
- D) `sp_OACreate`

---

**Q3:** When using `sp_invoke_external_rest_endpoint` to call Azure OpenAI, what is the JSON path to extract the generated text from the first choice?
- A) `$.result.content`
- B) `$.choices[0].message`
- C) **`$.result.choices[0].message.content`** ✅
- D) `$.result.message.content`

> **Explanation:** `sp_invoke_external_rest_endpoint` wraps the API response under `$.result`, so the path must include `.result.`.

---

**Q4:** To minimize LLM hallucinations in a RAG system, which system prompt instruction is MOST effective?
- A) "Be creative and elaborate in your answers."
- B) "Use general knowledge to supplement the context."
- C) **"Answer ONLY based on the provided context. If the context doesn't contain the answer, say 'I don't know.'"** ✅
- D) "Search the internet for the latest information."

---

**Q5:** Your RAG system's LLM response is truncated. The `finish_reason` in the response is `"length"`. What should you adjust?
- A) Increase `temperature`
- B) **Increase `max_tokens`** ✅
- C) Reduce the number of context documents
- D) Switch to a smaller model

---

**Q6:** For a production Azure SQL environment, what is the RECOMMENDED authentication method for `sp_invoke_external_rest_endpoint` to avoid API key rotation?
- A) API Key stored in a database scoped credential
- B) Hard-coded API key in the stored procedure
- C) **Managed Identity with 'Cognitive Services User' RBAC role** ✅
- D) SQL Server login with Azure credentials

---

**Q7:** You need to convert a SQL result set containing retrieved document titles and content into a JSON string for an LLM prompt. Which T-SQL clause should you use?
- A) `STRING_AGG`
- B) `XML PATH`
- C) **`FOR JSON PATH` or `JSON_ARRAYAGG` with `JSON_OBJECT`** ✅
- D) `CAST AS NVARCHAR(MAX)`

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| sp_invoke_external_rest_endpoint | [learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql) |
| RAG with Azure SQL | [learn.microsoft.com/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications](https://learn.microsoft.com/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications) |
| Azure OpenAI REST API | [learn.microsoft.com/azure/ai-services/openai/reference](https://learn.microsoft.com/azure/ai-services/openai/reference) |
| Database scoped credentials | [learn.microsoft.com/sql/t-sql/statements/create-database-scoped-credential-transact-sql](https://learn.microsoft.com/sql/t-sql/statements/create-database-scoped-credential-transact-sql) |
| FOR JSON clause | [learn.microsoft.com/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server](https://learn.microsoft.com/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server) |
| Managed Identity for Azure SQL | [learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure) |
| RAG pattern overview | [learn.microsoft.com/azure/search/retrieval-augmented-generation-overview](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview) |

---

> 📌 **Study Tip:** Memorize the complete 5-step RAG flow in T-SQL: embed → vector search → `FOR JSON` → `sp_invoke_external_rest_endpoint` → `JSON_VALUE(response, '$.result.choices[0].message.content')`. This path comes up frequently in exam questions.

---

*Last updated: September 2026 | DP-800 Skills measured as of March 12, 2026*
