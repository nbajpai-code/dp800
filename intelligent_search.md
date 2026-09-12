# 🔍 Design and Implement Intelligent Search

> **DP-800 Exam Domain:** Implement AI Capabilities in Database Solutions (25–30%)  
> **Subtopic:** Design and Implement Intelligent Search  
> **Skills Measured (as of March 12, 2026)**

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [Search Type Overview](#-search-type-overview)
- [1. Choose Between Full-Text, Vector, and Hybrid Search](#1-choose-from-full-text-semantic-vector-and-hybrid-search)
- [2. Implement Full-Text Search](#2-implement-full-text-search)
- [3. Design for Vector Data](#3-design-for-vector-data)
- [4. Vector Functions Reference](#4-vector-related-types-and-functions)
- [5. ANN vs ENN](#5-choose-between-ann-and-enn-for-vector-search)
- [6. Vector Index Types and Metrics](#6-evaluate-vector-index-types-and-metrics)
- [7. Implement Vector Search](#7-implement-vector-search)
- [8. Implement Hybrid Search](#8-implement-hybrid-search)
- [9. Reciprocal Rank Fusion (RRF)](#9-implement-reciprocal-rank-fusion-rrf)
- [10. Evaluate Search Performance](#10-evaluate-performance-of-vector-and-hybrid-search)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Choose from full-text, semantic vector, and hybrid search | ⭐⭐⭐ |
| 2 | Implement full-text search | ⭐⭐⭐ |
| 3 | Design for vector data (type, indexes, size) | ⭐⭐⭐⭐ |
| 4 | Use VECTOR_NORMALIZE, VECTOR_DISTANCE, VECTORPROPERTY, VECTOR_SEARCH | ⭐⭐⭐⭐ |
| 5 | Choose between ANN and ENN for vector search | ⭐⭐⭐ |
| 6 | Evaluate vector index types and metrics | ⭐⭐⭐⭐ |
| 7 | Implement vector search | ⭐⭐⭐⭐ |
| 8 | Implement hybrid search | ⭐⭐⭐⭐ |
| 9 | Implement reciprocal rank fusion (RRF) | ⭐⭐⭐⭐ |
| 10 | Evaluate performance of vector and hybrid search | ⭐⭐⭐ |

---

## 🗺️ Search Type Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Search Strategy Decision                        │
├─────────────────────────────────────────────────────────────────────┤
│  Full-Text Search (FTS)   │ Keyword matching, exact terms           │
│                           │ Fast, no ML required                    │
│                           │ Best: product codes, document search    │
├─────────────────────────────────────────────────────────────────────┤
│  Vector/Semantic Search   │ Meaning-based matching                  │
│                           │ Requires embeddings (ML model)          │
│                           │ Best: Q&A, recommendations, RAG         │
├─────────────────────────────────────────────────────────────────────┤
│  Hybrid Search            │ FTS + Vector combined                   │
│  (FTS + Vector + RRF)     │ Best of both: precise + semantic        │
│                           │ Best: enterprise search, chatbots       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Choose from Full-Text, Semantic Vector, and Hybrid Search

### Decision Matrix

| Criteria | Full-Text Search | Vector Search | Hybrid Search |
|----------|-----------------|---------------|---------------|
| **Query type** | Keywords, exact phrases | Natural language, intent | Both |
| **Requires ML model** | ❌ No | ✅ Yes (embedding model) | ✅ Yes |
| **Setup complexity** | Low | Medium-High | High |
| **Handles synonyms** | Limited (thesaurus only) | ✅ Yes (semantic) | ✅ Yes |
| **Handles typos** | ❌ Poor (exact match) | ✅ Yes (semantic similarity) | ✅ Yes |
| **Ranking quality** | Medium | High | Highest |
| **Performance** | ✅ Fast | ⚠️ Slower (ANN index) | ⚠️ Medium |
| **Use case** | Document search, product codes | Semantic Q&A, RAG | Enterprise search |

### When to Choose Each

```
Use Full-Text Search when:
  ✅ Exact keyword matching is required
  ✅ No ML infrastructure available
  ✅ Query is known terminology (product IDs, legal terms)
  ✅ Performance is critical, dataset is very large

Use Vector Search when:
  ✅ Users ask questions in natural language
  ✅ Semantic understanding is needed (synonyms, intent)
  ✅ Building RAG / chatbot applications
  ✅ Recommendations (similar items, content)

Use Hybrid Search when:
  ✅ Need best precision + recall combined
  ✅ Diverse query types (keyword AND semantic)
  ✅ Enterprise search with high accuracy requirements
  ✅ Building production-grade AI search
```

---

## 2. Implement Full-Text Search

### ⚙️ Setup: Full-Text Index

```sql
-- Step 1: Verify Full-Text Search feature is installed
SELECT FULLTEXTSERVICEPROPERTY('IsFullTextInstalled');
-- Returns 1 if installed

-- Step 2: Create a Full-Text Catalog
CREATE FULLTEXT CATALOG FTCatalog AS DEFAULT;

-- Step 3: Create Full-Text Index on the table
CREATE FULLTEXT INDEX ON Products
(
    ProductName        LANGUAGE 1033,   -- 1033 = English
    ProductDescription LANGUAGE 1033
)
KEY INDEX PK_Products                  -- Must reference a unique index
ON FTCatalog
WITH STOPLIST = SYSTEM,               -- Ignore common words (a, the, is...)
     CHANGE_TRACKING AUTO;            -- Auto-update index on data changes
```

### 📐 Full-Text Search Predicates (CONTAINS / FREETEXT)

```sql
-- CONTAINS: exact word, phrase, wildcard, proximity, boolean
SELECT ProductName, ProductDescription
FROM Products
WHERE CONTAINS(ProductDescription, 'wireless');

-- CONTAINS with exact phrase
SELECT * FROM Products
WHERE CONTAINS(ProductDescription, '"wireless mouse"');

-- CONTAINS with prefix wildcard
SELECT * FROM Products
WHERE CONTAINS(ProductName, '"wire*"');  -- matches wireless, wired, wire

-- CONTAINS with proximity (NEAR)
SELECT * FROM Products
WHERE CONTAINS(ProductDescription, 'wireless NEAR mouse');

-- CONTAINS with Boolean operators
SELECT * FROM Products
WHERE CONTAINS(ProductDescription,
    'wireless AND (mouse OR keyboard) AND NOT refurbished');

-- FREETEXT: natural language (automatic stemming + thesaurus)
SELECT * FROM Products
WHERE FREETEXT(ProductDescription, 'fast powerful laptop computer');
-- Matches 'fast' → 'fastest', 'powerful' → 'power', etc.
```

### 📐 Full-Text Functions (Return Relevance Rank Scores)

```sql
-- CONTAINSTABLE: returns [KEY] and RANK for each match
SELECT
    p.ProductName,
    p.ProductDescription,
    ft.RANK AS Relevance
FROM Products p
INNER JOIN CONTAINSTABLE(Products, ProductDescription, 'wireless mouse') AS ft
    ON p.ProductID = ft.[KEY]
ORDER BY ft.RANK DESC;

-- FREETEXTTABLE: natural language + rank scores
SELECT
    p.ProductName,
    ft.RANK
FROM Products p
INNER JOIN FREETEXTTABLE(Products, *, 'fast laptop computer') AS ft
    ON p.ProductID = ft.[KEY]
ORDER BY ft.RANK DESC;
```

### 📊 FTS Key Concepts

| Concept | Description |
|---------|-------------|
| **Stopwords** | Common words ignored by FTS (the, a, is) — configurable via STOPLIST |
| **Stemming** | Matches word variations (run → runs, running, ran) |
| **Thesaurus** | Custom synonym dictionary (car → automobile, vehicle) |
| **Change tracking** | AUTO = auto-update; MANUAL = explicit update needed |
| **Catalog** | Logical container for full-text indexes |

---

## 3. Design for Vector Data

### 🔑 The `VECTOR` Data Type

The `VECTOR` data type stores fixed-dimension floating-point arrays (embeddings), available in **Azure SQL Database**, **SQL Server 2025+**, and **SQL databases in Fabric**.

```sql
-- Create a table with a vector column
CREATE TABLE ProductEmbeddings (
    ProductID       INT PRIMARY KEY,
    ProductName     NVARCHAR(500),
    EmbeddingText   NVARCHAR(MAX),  -- Source text used for embedding
    Embedding       VECTOR(1536)    -- 1536-dimension vector (OpenAI ada-002 size)
);

-- Common embedding dimensions:
-- text-embedding-ada-002  (OpenAI):           1536
-- text-embedding-3-small  (OpenAI):           1536
-- text-embedding-3-large  (OpenAI):           3072
-- all-MiniLM-L6-v2 (sentence-transformers):   384
-- e5-large-v2:                               1024
```

### ⚙️ Choosing Vector Dimension Size

| Dimension | Model | Trade-off |
|-----------|-------|----------|
| 384 | MiniLM, small models | Fast, less accurate, less storage |
| 768 | BERT-base | Balanced quality and performance |
| 1536 | OpenAI ada-002, text-3-small | High quality — industry standard |
| 3072 | OpenAI text-3-large | Highest quality, more storage |

**Storage calculation:**
- Each dimension = 4 bytes (float32)
- 1536 dimensions × 4 bytes = **6,144 bytes per embedding**
- 1M rows × 6 KB = **~6 GB** for the embedding column alone

### 📐 Inserting Embeddings

```sql
-- Insert a vector from a JSON array string representation
INSERT INTO ProductEmbeddings (ProductID, ProductName, Embedding)
VALUES (
    1,
    'Wireless Mouse',
    '[0.023, -0.145, 0.678, ...]'  -- 1536 floats as JSON array string
);
```

---

## 4. Vector-Related Types and Functions

### 🔑 Core Vector Functions

| Function | Purpose | Return Type |
|----------|---------|-------------|
| `VECTOR_NORMALIZE(v)` | Normalize vector to unit length (L2 norm = 1) | `VECTOR` |
| `VECTOR_DISTANCE(metric, v1, v2)` | Calculate distance between two vectors | `FLOAT` |
| `VECTORPROPERTY(v, property)` | Get vector metadata (dimensions, base type) | varies |
| `VECTOR_SEARCH(...)` | Perform ANN vector search using an index | table |

### 📐 VECTOR_NORMALIZE

```sql
-- Normalize vectors to unit length before using dot product as cosine similarity
INSERT INTO ProductEmbeddings (ProductID, Embedding)
SELECT ProductID, VECTOR_NORMALIZE(Embedding)
FROM ProductEmbeddingsRaw;

-- Verify normalization: self dot product of normalized vector = 1.0
SELECT
    ProductID,
    VECTOR_DISTANCE('dot', Embedding, Embedding) AS SelfDotProduct
FROM ProductEmbeddings;
-- If normalized correctly: SelfDotProduct = 1.0
```

### 📐 VECTOR_DISTANCE

```sql
-- Syntax: VECTOR_DISTANCE(metric, vector1, vector2) → float (lower = more similar)
DECLARE @query_vector VECTOR(1536) = '[0.023, -0.145, 0.678, ...]';

SELECT
    ProductID,
    ProductName,
    VECTOR_DISTANCE('cosine',    Embedding, @query_vector) AS CosineDist,
    VECTOR_DISTANCE('euclidean', Embedding, @query_vector) AS EuclideanDist,
    VECTOR_DISTANCE('dot',       Embedding, @query_vector) AS DotProduct
FROM ProductEmbeddings
ORDER BY CosineDist ASC  -- Lower distance = more similar
FETCH FIRST 10 ROWS ONLY;
```

### 📊 Distance Metrics Comparison

| Metric | Range | Interpretation | Best For |
|--------|-------|----------------|----------|
| **cosine** | 0–2 | 0 = identical direction, 2 = opposite | Text embeddings (most common) |
| **euclidean** | 0–∞ | 0 = same point in space | Image embeddings, spatial data |
| **dot** | -∞ to ∞ | Higher = more similar | Pre-normalized vectors only |

> 💡 **Best practice:** Use `cosine` for text embeddings. If vectors are normalized, `dot` product = 1 − cosine distance (faster computation).

### 📐 VECTORPROPERTY

```sql
-- Get metadata about a vector column
SELECT
    ProductID,
    VECTORPROPERTY(Embedding, 'Dimensions') AS NumDimensions,  -- e.g., 1536
    VECTORPROPERTY(Embedding, 'BaseType')   AS DataType         -- e.g., float32
FROM ProductEmbeddings
WHERE ProductID = 1;
```

---

## 5. Choose Between ANN and ENN for Vector Search

### 🔑 Definitions

| Term | Full Name | Description |
|------|-----------|-------------|
| **ENN** | Exact Nearest Neighbor | Brute-force scan of ALL vectors — always finds the true nearest neighbor |
| **ANN** | Approximate Nearest Neighbor | Uses a vector index to find near-nearest neighbors quickly — may miss a few |

### 📊 Comparison

| Aspect | ENN | ANN |
|--------|-----|-----|
| **Accuracy** | ✅ 100% exact | ⚠️ ~95–99% recall (approximate) |
| **Performance** | ❌ O(n) — slow at scale | ✅ O(log n) — fast |
| **Index required** | ❌ No index | ✅ Vector index (DiskANN, IVFFlat) |
| **Good for** | Small datasets (< 100K rows) | Large datasets (100K+ rows) |
| **SQL implementation** | `VECTOR_DISTANCE` with `ORDER BY` | `VECTOR_SEARCH` with index |

### 🎯 Decision Guide

```
Dataset < 50,000 rows?            → ENN (simple, exact, fast enough)
Dataset > 50,000 rows?            → ANN (index-based, scalable)
Need 100% recall (zero misses)?   → ENN
Need sub-second response time?    → ANN
Production RAG / chatbot?         → ANN
Batch comparison / evaluation?    → ENN
```

---

## 6. Evaluate Vector Index Types and Metrics

### 🔑 DiskANN (Disk-based Approximate Nearest Neighbor)

DiskANN is the primary vector index type in **Azure SQL Database** and **Fabric SQL database**.

```sql
-- Create a DiskANN index
CREATE INDEX idx_product_embedding
    ON ProductEmbeddings (Embedding)
    WITH (
        TYPE   = DISKANN,    -- Index algorithm
        METRIC = 'cosine',   -- Distance metric (must match query metric)
        maxdop = 1           -- Degree of parallelism during build
    );
```

**DiskANN characteristics:**
- Designed for large-scale datasets (millions of vectors)
- Graph-based structure stored partially on SSD — low memory footprint
- High recall (~98–99%) at production-grade performance
- Supports `cosine`, `euclidean`, `dot` metrics

### 📊 Vector Index Type Comparison

| Property | DiskANN | IVFFlat |
|----------|---------|---------|
| **Algorithm** | Graph-based (HNSW variant) | Inverted file with flat quantization |
| **Recall** | ~98–99% | ~90–95% |
| **Build time** | Medium | Fast |
| **Memory usage** | Low (disk-backed) | Medium |
| **Best for** | Large production datasets | Moderate datasets, faster build |

### 📊 Distance Metrics for Indexes

| Metric | Best for | Notes |
|--------|---------|-------|
| **cosine** | Text embeddings | Most common for NLP tasks |
| **euclidean** | Image embeddings, spatial data | Sensitive to vector magnitude |
| **dot** | Pre-normalized vectors | Fastest when all vectors are unit-normalized |

> ⚠️ **Critical:** The metric used in `VECTOR_SEARCH` **must match** the metric used when creating the index.

---

## 7. Implement Vector Search

### 📐 ENN (Exact) Vector Search

```sql
-- Exact Nearest Neighbor: scan all rows, compute distance, sort
DECLARE @query_vector VECTOR(1536) = '[0.023, -0.145, 0.678, ...]';

SELECT TOP 10
    p.ProductID,
    p.ProductName,
    VECTOR_DISTANCE('cosine', p.Embedding, @query_vector) AS Distance
FROM ProductEmbeddings p
ORDER BY Distance ASC;  -- Lower = more similar
```

### 📐 ANN (Approximate) Vector Search with VECTOR_SEARCH

```sql
-- Approximate Nearest Neighbor: uses DiskANN index for fast search
DECLARE @query_vector VECTOR(1536) = '[0.023, -0.145, 0.678, ...]';

SELECT
    v.ProductID,
    p.ProductName,
    p.ProductDescription,
    v.distance
FROM VECTOR_SEARCH(
    TABLE  = ProductEmbeddings AS p,  -- Table with vector index
    COLUMN = Embedding,               -- Column with vector index
    SIMILAR_TO = @query_vector,       -- Query vector
    METRIC = 'cosine',               -- Must match index metric
    TOP_N  = 10,                      -- Number of results
    WITH_DISTANCE = TRUE              -- Include distance scores
) AS v
INNER JOIN ProductEmbeddings p ON v.ProductID = p.ProductID
ORDER BY v.distance ASC;
```

### 📐 Vector Search with Metadata Pre-filter

```sql
-- Combine vector search with SQL WHERE filter (filter by category first)
DECLARE @query_vector VECTOR(1536) = '[...]';

SELECT TOP 10
    p.ProductID,
    p.ProductName,
    p.Category,
    VECTOR_DISTANCE('cosine', p.Embedding, @query_vector) AS Distance
FROM ProductEmbeddings p
WHERE p.Category = 'Electronics'      -- Pre-filter reduces search space
  AND p.IsActive = 1
ORDER BY Distance ASC;
```

---

## 8. Implement Hybrid Search

### 🔑 What is Hybrid Search?

Hybrid search combines **Full-Text Search** (keyword precision) and **Vector Search** (semantic relevance) into a single unified result set.

```
Query: "wireless mouse for gaming"

FTS results:     [Mouse Pro X, Wireless Desktop Kit, Mouse Pad...]
Vector results:  [Gaming Mouse 3000, Pro Gaming Peripherals, Mouse Pro X...]

Hybrid (RRF):    [Mouse Pro X ⭐ #1, Gaming Mouse 3000 #2, Wireless Desktop Kit #3...]
```

### 📐 Implementing Hybrid Search

```sql
DECLARE @query        NVARCHAR(500) = 'wireless mouse gaming';
DECLARE @query_vector VECTOR(1536)  = '[...]';  -- Embedding of @query

WITH FTSResults AS (
    -- Step 1: Full-Text Search with rank positions
    SELECT
        p.ProductID,
        ft.RANK AS FTS_Rank,
        ROW_NUMBER() OVER (ORDER BY ft.RANK DESC) AS FTS_Position
    FROM Products p
    INNER JOIN CONTAINSTABLE(Products, *, @query) AS ft
        ON p.ProductID = ft.[KEY]
),
VectorResults AS (
    -- Step 2: Vector Search with distance-based rank positions
    SELECT TOP 20
        p.ProductID,
        VECTOR_DISTANCE('cosine', p.Embedding, @query_vector) AS Vec_Distance,
        ROW_NUMBER() OVER (ORDER BY
            VECTOR_DISTANCE('cosine', p.Embedding, @query_vector) ASC
        ) AS Vec_Position
    FROM ProductEmbeddings p
),
RRF AS (
    -- Step 3: Reciprocal Rank Fusion merge (k=60)
    SELECT
        COALESCE(f.ProductID, v.ProductID) AS ProductID,
        COALESCE(1.0 / (60 + f.FTS_Position), 0) +
        COALESCE(1.0 / (60 + v.Vec_Position), 0) AS RRF_Score
    FROM FTSResults f
    FULL OUTER JOIN VectorResults v ON f.ProductID = v.ProductID
)
SELECT TOP 10
    r.ProductID,
    p.ProductName,
    r.RRF_Score
FROM RRF r
JOIN Products p ON r.ProductID = p.ProductID
ORDER BY r.RRF_Score DESC;
```

---

## 9. Implement Reciprocal Rank Fusion (RRF)

### 🔑 What is RRF?

**Reciprocal Rank Fusion** is a rank aggregation algorithm that merges multiple ranked result lists into a single unified ranking — **without requiring score normalization** across different scales.

### 📐 RRF Formula

```
RRF_score(d) = Σ  1 / (k + rank_r(d))
               r∈R

Where:
  d        = document/result being scored
  R        = set of ranked result lists (FTS, vector, etc.)
  rank_r(d) = position of document d in result list r
  k        = smoothing constant (standard default: 60)
```

### 📐 RRF Example Walkthrough

```
Query: "wireless gaming mouse"

FTS results:                Vector results:
  Rank 1: Mouse Pro X         Rank 1: Gaming Gear Ultra
  Rank 2: Gaming Gear Ultra   Rank 2: Mouse Pro X
  Rank 3: Wireless Kit        Rank 3: Budget Mouse

RRF Scores (k=60):
  Mouse Pro X:       1/(60+1) + 1/(60+2) = 0.01639 + 0.01613 = 0.03252 → #1
  Gaming Gear Ultra: 1/(60+2) + 1/(60+1) = 0.01613 + 0.01639 = 0.03252 → #1 (tie)
  Wireless Kit:      1/(60+3) + 0         = 0.01587               → #3
  Budget Mouse:      0        + 1/(60+3)  = 0.01587               → #3 (tie)
```

```sql
-- Standalone RRF implementation
DECLARE @k INT = 60;

WITH
List1 AS (
    SELECT ProductID,
        ROW_NUMBER() OVER (ORDER BY FTS_Score DESC) AS Rank1
    FROM FTSSearchResults
),
List2 AS (
    SELECT ProductID,
        ROW_NUMBER() OVER (ORDER BY Vec_Distance ASC) AS Rank2
    FROM VectorSearchResults
)
SELECT
    COALESCE(l1.ProductID, l2.ProductID) AS ProductID,
    COALESCE(1.0/(@k + l1.Rank1), 0) +
    COALESCE(1.0/(@k + l2.Rank2), 0) AS RRF_Score
FROM List1 l1
FULL OUTER JOIN List2 l2 ON l1.ProductID = l2.ProductID
ORDER BY RRF_Score DESC;
```

### 🎯 RRF Key Properties

| Property | Detail |
|----------|--------|
| **No score normalization needed** | Works with incompatible score scales (e.g., FTS RANK 1–1000 vs cosine distance 0–2) |
| **k = 60 is the standard default** | Empirically proven — prevents extreme top ranks from dominating |
| **Supports 3+ ranked lists** | Just add more `1/(k + rank)` terms |
| **Robust to outliers** | One list assigning a wrong high rank doesn't dominate final score |
| **Monotonic** | Being ranked #1 in any list always improves the final RRF score |

---

## 10. Evaluate Performance of Vector and Hybrid Search

### 📊 Key Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **Recall@K** | % of true top-K results found in returned top-K | > 95% for ANN |
| **Precision@K** | % of returned top-K that are truly relevant | Depends on use case |
| **Latency (P50/P99)** | Query response time percentiles | < 100ms P99 for production |
| **QPS** | Queries per second throughput | Depends on SLA |
| **Index build time** | Time to build vector index after bulk inserts | Monitor post-load |

### 📐 Measuring Recall@K (ANN vs ENN Comparison)

```sql
-- Compare ANN results vs ENN (exact) to measure recall quality
DECLARE @query_vector VECTOR(1536) = '[...]';
DECLARE @K INT = 10;

WITH ExactTop AS (
    -- ENN: true top-K
    SELECT TOP (@K) ProductID
    FROM ProductEmbeddings
    ORDER BY VECTOR_DISTANCE('cosine', Embedding, @query_vector) ASC
),
ApproxTop AS (
    -- ANN: approximate top-K
    SELECT ProductID
    FROM VECTOR_SEARCH(
        TABLE = ProductEmbeddings,
        COLUMN = Embedding,
        SIMILAR_TO = @query_vector,
        METRIC = 'cosine',
        TOP_N = @K
    )
)
SELECT
    COUNT(*) AS MatchCount,
    CAST(COUNT(*) AS FLOAT) / @K * 100 AS RecallPct
FROM ExactTop e
WHERE EXISTS (SELECT 1 FROM ApproxTop a WHERE a.ProductID = e.ProductID);
```

### 🔧 Performance Tuning Tips

| Issue | Solution |
|-------|----------|
| Slow ANN search | Verify DiskANN index exists and metric matches; use `VECTOR_SEARCH` not `ORDER BY VECTOR_DISTANCE` |
| Low recall | Increase `TOP_N`; check index build status; verify metric alignment |
| Slow index build | Increase `maxdop`; build during off-peak hours |
| High memory usage | Use lower-dimension embedding model |
| Slow hybrid search | Optimize FTS index with `CHANGE_TRACKING AUTO`; add pre-filter on metadata |

---

## ❓ Practice Questions

**Q1:** A user searches your product catalog with "something to help me type faster" — they don't use the word 'keyboard'. Which search type BEST handles this?
- A) Full-Text Search
- B) **Vector/Semantic Search** ✅
- C) Fuzzy string matching
- D) LIKE with wildcards

> **Explanation:** Vector search understands semantic intent — 'type faster' maps semantically to 'keyboard/typing aids' even without exact keyword matches.

---

**Q2:** You have 10 million product embeddings and need sub-100ms search latency. What should you use?
- A) ENN with `ORDER BY VECTOR_DISTANCE`
- B) Full-Text Search
- C) **ANN with DiskANN vector index and `VECTOR_SEARCH`** ✅
- D) Fuzzy search with `EDIT_DISTANCE`

---

**Q3:** Before using dot product as the distance metric for your embeddings, what is required?
- A) All embeddings have the same dimension
- B) The DiskANN index is rebuilt
- C) **Vectors are normalized to unit length using VECTOR_NORMALIZE** ✅
- D) The metric is set to 'euclidean' in the index

---

**Q4:** A hybrid search returns FTS rank positions and vector distance values. You need to merge them into one ranked list without normalizing scores. What algorithm should you use?
- A) Simple average of scores
- B) Weighted sum of distances
- C) **Reciprocal Rank Fusion (RRF)** ✅
- D) UNION ALL with ORDER BY

---

**Q5:** The standard default value for the `k` constant in RRF is:
- A) k=1 — maximum weight to top-ranked results
- B) **k=60 — prevents top ranks from dominating the score** ✅
- C) k=100 — for large datasets
- D) k is always dataset-specific and has no standard

---

**Q6:** You created a DiskANN index with `METRIC = 'cosine'`. A developer runs `VECTOR_SEARCH` with `METRIC = 'euclidean'`. What happens?
- A) Results returned correctly with euclidean distances
- B) The index is automatically rebuilt
- C) **The query may error or fail to use the index (metric mismatch)** ✅
- D) SQL converts between metrics automatically

---

**Q7:** Which function returns the number of dimensions in a vector column?
- A) `VECTOR_DISTANCE`
- B) `LEN(Embedding)`
- C) **`VECTORPROPERTY(Embedding, 'Dimensions')`** ✅
- D) `DATALENGTH(Embedding) / 4`

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| Vector data type (Azure SQL) | [learn.microsoft.com/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications](https://learn.microsoft.com/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications) |
| VECTOR_DISTANCE function | [learn.microsoft.com/sql/t-sql/functions/vector-distance-transact-sql](https://learn.microsoft.com/sql/t-sql/functions/vector-distance-transact-sql) |
| VECTOR_SEARCH function | [learn.microsoft.com/sql/t-sql/functions/vector-search-transact-sql](https://learn.microsoft.com/sql/t-sql/functions/vector-search-transact-sql) |
| VECTOR_NORMALIZE function | [learn.microsoft.com/sql/t-sql/functions/vector-normalize-transact-sql](https://learn.microsoft.com/sql/t-sql/functions/vector-normalize-transact-sql) |
| DiskANN indexes | [learn.microsoft.com/azure/azure-sql/database/vector-search-diskann](https://learn.microsoft.com/azure/azure-sql/database/vector-search-diskann) |
| Full-Text Search | [learn.microsoft.com/sql/relational-databases/search/full-text-search](https://learn.microsoft.com/sql/relational-databases/search/full-text-search) |
| Reciprocal Rank Fusion | [learn.microsoft.com/azure/search/hybrid-search-ranking](https://learn.microsoft.com/azure/search/hybrid-search-ranking) |
| Intelligent search in Fabric SQL | [learn.microsoft.com/fabric/database/sql/intelligent-search](https://learn.microsoft.com/fabric/database/sql/intelligent-search) |

---

> 📌 **Study Tip:** The exam tests you on the RRF formula (k=60 default), when to choose ANN vs ENN, and the key difference: `VECTOR_DISTANCE` = brute-force ENN; `VECTOR_SEARCH` = indexed ANN. Practice building a DiskANN index in Azure SQL or Fabric SQL free trial.

---

*Last updated: September 2026 | DP-800 Skills measured as of March 12, 2026*
