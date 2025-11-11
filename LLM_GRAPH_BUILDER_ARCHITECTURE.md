# LLM Graph Builder - Complete Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [Backend Architecture](#backend-architecture)
3. [GraphRAG Implementation](#graphrag-implementation)
4. [Entity Extraction & Merging](#entity-extraction--merging)
5. [Retrieval Strategies](#retrieval-strategies)
6. [API Reference](#api-reference)
7. [Configuration](#configuration)
8. [Key Files Reference](#key-files-reference)

---

## Overview

**LLM Graph Builder** is a sophisticated system that transforms unstructured documents into a structured Knowledge Graph, enabling advanced Retrieval-Augmented Generation (RAG) with graph traversal capabilities.

### Core Features

- **Multi-Source Ingestion**: Local files, S3, GCS, YouTube, Wikipedia, Web pages
- **LLM-Powered Extraction**: Uses GPT-4, Gemini, Claude, or other LLMs for entity/relationship extraction
- **Automatic Entity Deduplication**: Smart merging of duplicate entities across chunks
- **6 RAG Modes**: From simple vector search to multi-hop graph traversal
- **Community Detection**: Hierarchical clustering with Leiden algorithm
- **Hybrid Search**: Combines vector, full-text, and graph-based retrieval

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Backend Framework** | FastAPI + Uvicorn | Async REST API |
| **Database** | Neo4j 5.23+ | Graph database |
| **LLM Framework** | LangChain 0.3.25 | LLM orchestration |
| **Embeddings** | Sentence Transformers | Vector representations |
| **Document Processing** | Unstructured.io | Multi-format parsing |
| **Graph Algorithms** | Neo4j GDS | Community detection |

---

## Backend Architecture

### Project Structure

```
backend/
├── score.py                    # FastAPI entry point (1095 lines)
├── src/
│   ├── main.py                 # Document processing pipeline
│   ├── llm.py                  # LLM factory and management
│   ├── QA_integration.py       # RAG implementation
│   ├── graphDB_dataAccess.py   # Neo4j data access layer
│   ├── create_chunks.py        # Document chunking
│   ├── post_processing.py      # Indexing and consolidation
│   ├── communities.py          # Community detection
│   ├── make_relationships.py   # Entity-chunk linking
│   ├── document_sources/       # Source-specific handlers
│   │   ├── local_file.py
│   │   ├── s3_bucket.py
│   │   ├── gcs_bucket.py
│   │   ├── youtube.py
│   │   ├── wikipedia.py
│   │   └── web_pages.py
│   └── shared/
│       ├── constants.py        # Cypher queries and constants
│       └── common_fn.py        # Utility functions
```

### Architectural Patterns

#### 1. Layered Architecture
```
┌─────────────────────────────────────┐
│   Presentation Layer (FastAPI)     │
├─────────────────────────────────────┤
│   Business Logic (main.py)         │
├─────────────────────────────────────┤
│   Data Access (graphDB_dataAccess) │
├─────────────────────────────────────┤
│   Database (Neo4j)                 │
└─────────────────────────────────────┘
```

#### 2. Factory Pattern
- **LLM Factory** (`llm.py`): Creates appropriate LLM instances based on configuration
- **Document Source Factory**: Handles different data sources uniformly

#### 3. Repository Pattern
- `graphDB_dataAccess` class abstracts all Neo4j operations
- Encapsulates Cypher queries and database logic

#### 4. Pipeline Pattern
- Sequential processing stages for document transformation
- Each stage can be retried or rolled back independently

---

## GraphRAG Implementation

### End-to-End Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│ 1. DOCUMENT INGESTION                                       │
│    └─ Upload/Fetch → Parse → Extract Text                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. CHUNKING                                                 │
│    └─ TokenTextSplitter → Chunks (~100 tokens each)       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. CHUNK COMBINATION                                        │
│    └─ Combine N chunks (default: 6) for broader context   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. ENTITY EXTRACTION (LLM)                                 │
│    └─ LLMGraphTransformer → Entities + Relationships      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. ENTITY MERGING                                          │
│    └─ APOC merge.node → Deduplication by ID               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. CHUNK-ENTITY LINKING                                    │
│    └─ Create HAS_ENTITY relationships                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. EMBEDDING GENERATION                                     │
│    └─ sentence-transformers → Vector embeddings            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. POST-PROCESSING (Optional)                              │
│    ├─ KNN Similarity (chunk-chunk)                        │
│    ├─ Community Detection (Leiden)                        │
│    ├─ Full-text Indexing                                  │
│    └─ Schema Consolidation                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. RAG RETRIEVAL                                           │
│    └─ Vector + Graph + Full-text Search                   │
└─────────────────────────────────────────────────────────────┘
```

### Graph Structure

The final knowledge graph has this structure:

```cypher
# Core Structure
(Document)-[:PART_OF]->(Chunk)
(Chunk)-[:NEXT_CHUNK]->(Chunk)
(Chunk)-[:HAS_ENTITY]->(Entity)
(Entity)-[:RELATIONSHIP_TYPE]->(Entity)

# Community Structure (post-processing)
(Entity)-[:IN_COMMUNITY]->(Community_L0)
(Community_L0)-[:PARENT_COMMUNITY]->(Community_L1)
(Community_L1)-[:PARENT_COMMUNITY]->(Community_L2)

# Similarity Structure (post-processing)
(Chunk)-[:SIMILAR {score}]->(Chunk)
```

---

## Entity Extraction & Merging

### 1. Chunk Combination Strategy

**File**: [`backend/src/llm.py:140-164`](backend/src/llm.py#L140-L164)

Before LLM extraction, consecutive chunks are combined to provide broader context:

```python
def get_combined_chunks(chunkId_chunkDoc_list, chunks_to_combine=6):
    """
    Combines consecutive chunks to provide more context to the LLM.

    Example: 100 chunks → 17 combined chunks (6 chunks each)
    """
    combined_chunks = []
    for i in range(0, len(chunkId_chunkDoc_list), chunks_to_combine):
        # Concatenate text from N consecutive chunks
        text = "".join(
            doc["chunk_doc"].page_content
            for doc in chunkId_chunkDoc_list[i : i + chunks_to_combine]
        )

        # Track original chunk IDs
        chunk_ids = [
            doc["chunk_id"]
            for doc in chunkId_chunkDoc_list[i : i + chunks_to_combine]
        ]

        combined_chunks.append({
            "text": text,
            "metadata": {"combined_chunk_ids": chunk_ids}
        })

    return combined_chunks
```

**Benefits**:
- ✅ **Broader context**: LLM sees relationships across chunk boundaries
- ✅ **Fewer API calls**: 100 chunks → only 17 LLM calls
- ✅ **Better relationships**: Entities mentioned in different chunks get connected

### 2. LLM Graph Transformer

**File**: [`backend/src/llm.py:196-204`](backend/src/llm.py#L196-L204)

The heart of entity extraction uses LangChain's `LLMGraphTransformer`:

```python
llm_transformer = LLMGraphTransformer(
    llm=llm,  # GPT-4, Gemini, Claude, etc.

    # Properties to extract
    node_properties=["description"],
    relationship_properties=["description"],

    # Optional schema constraints
    allowed_nodes=["Person", "Organization", "Technology", "Location"],
    allowed_relationships=[
        ("Person", "WORKS_FOR", "Organization"),
        ("Organization", "USES", "Technology"),
        ("Person", "LOCATED_IN", "Location")
    ],

    # Custom instructions
    additional_instructions="""
    Your goal is to identify and categorize entities while ensuring that
    specific data types such as dates, numbers, and revenues are not
    extracted as separate nodes. Instead, treat these as properties
    associated with the relevant entities.
    """
)

# Async extraction
graph_documents = await llm_transformer.aconvert_to_graph_documents(
    combined_chunk_documents
)
```

### 3. Concrete Extraction Example

**Input** (combined chunk):
```text
John Smith is the CEO of Acme Corporation. The company was founded in 2020
and has 500 employees. Acme develops AI software using OpenAI's GPT models.
John previously worked at Tech Innovations and graduated from MIT in 1995.
```

**Output** (GraphDocument):
```python
GraphDocument(
    nodes=[
        Node(
            id="John Smith",
            type="Person",
            properties={
                "description": "CEO of Acme Corporation, MIT graduate (1995)"
            }
        ),
        Node(
            id="Acme Corporation",
            type="Organization",
            properties={
                "description": "AI software company founded in 2020, 500 employees"
            }
        ),
        Node(
            id="Tech Innovations",
            type="Organization",
            properties={
                "description": "Previous employer of John Smith"
            }
        ),
        Node(
            id="OpenAI GPT",
            type="Technology",
            properties={
                "description": "AI models used for software development"
            }
        ),
        Node(
            id="MIT",
            type="Organization",
            properties={
                "description": "University where John Smith graduated"
            }
        )
    ],
    relationships=[
        Relationship(
            source=Node(id="John Smith"),
            target=Node(id="Acme Corporation"),
            type="WORKS_FOR",
            properties={"description": "Current CEO position"}
        ),
        Relationship(
            source=Node(id="John Smith"),
            target=Node(id="Tech Innovations"),
            type="PREVIOUSLY_WORKED_AT",
            properties={"description": "Former employment"}
        ),
        Relationship(
            source=Node(id="Acme Corporation"),
            target=Node(id="OpenAI GPT"),
            type="USES",
            properties={"description": "Uses for AI software development"}
        ),
        Relationship(
            source=Node(id="John Smith"),
            target=Node(id="MIT"),
            type="GRADUATED_FROM",
            properties={"description": "Graduated in 1995"}
        )
    ],
    source=Document(metadata={"combined_chunk_ids": ["chunk_1", "chunk_2", ...]})
)
```

### 4. Entity Merging Strategy

#### The Deduplication Problem

When processing multiple chunks, the same entity may be mentioned repeatedly:

```
Chunk 1: "John Smith is CEO of Acme"
Chunk 2: "John Smith graduated from MIT in 1995"
Chunk 3: "Smith, J. published a paper on AI"
```

**Without merging**: 3 separate nodes ❌
**With merging**: 1 unified node ✅

#### APOC Merge: The Magic Solution

**File**: [`backend/src/make_relationships.py:30-38`](backend/src/make_relationships.py#L30-L38)

```cypher
UNWIND $batch_data AS data
MATCH (c:Chunk {id: data.chunk_id})

-- ⭐ THE KEY LINE: APOC merge prevents duplicates ⭐
CALL apoc.merge.node([data.node_type], {id: data.node_id})
    YIELD node AS n

MERGE (c)-[:HAS_ENTITY]->(n)
```

**How `apoc.merge.node` works**:

```cypher
apoc.merge.node(
    ["Person"],              -- Node label(s)
    {id: "John Smith"}       -- Unique property for matching
)
```

**Behavior**:
1. Searches for a node with label `Person` AND `id = "John Smith"`
2. If **exists** → **reuses** it (no duplicates!)
3. If **doesn't exist** → **creates** it

#### Visual Example of Merging

**Processing Chunk 1**:
```
Chunk_1 -[:HAS_ENTITY]-> (Person:John Smith) [CREATED]
```

**Processing Chunk 2** (same person):
```
Chunk_2 -[:HAS_ENTITY]-> (Person:John Smith) [REUSED - not duplicated!]
```

**Processing Chunk 3**:
```
Chunk_3 -[:HAS_ENTITY]-> (Person:John Smith) [REUSED]
```

**Final graph structure**:
```
        ┌─────────────────────┐
        │  Person:John Smith  │  ← SINGLE NODE
        └─────────────────────┘
                 ↑   ↑   ↑
                 │   │   │
         HAS_ENTITY from 3 chunks
                 │   │   │
          ┌──────┘   │   └──────┐
          │          │          │
     ┌────┴───┐  ┌───┴────┐  ┌──┴─────┐
     │Chunk_1 │  │Chunk_2 │  │Chunk_3 │
     └────────┘  └────────┘  └────────┘
```

### 5. Property Management During Merge

**File**: [`backend/src/common_fn.py:95-109`](backend/src/common_fn.py#L95-L109)

```python
graph.add_graph_documents(
    graph_document_list,
    baseEntityLabel=True  # ⭐ Critical for merging
)
```

**What `baseEntityLabel=True` does**:

1. Adds `__Entity__` label to all entity nodes
2. Uses internal **MERGE** with `ON CREATE` and `ON MATCH` strategies:

```cypher
MERGE (n:Person:__Entity__ {id: "John Smith"})
ON CREATE SET
    n.description = "CEO of Acme Corporation"
ON MATCH SET
    n.description = n.description + "; CEO of Acme Corporation"
    -- Can concatenate or overwrite based on implementation
```

**Result**:
- **First time**: Creates node with initial description
- **Subsequent times**: **Updates** existing description (can concatenate or replace)

### 6. Relationship Deduplication

Relationships are also automatically deduplicated:

```cypher
-- Chunk 1 extracts:
MERGE (john:Person {id: "John Smith"})
MERGE (acme:Organization {id: "Acme Corporation"})
MERGE (john)-[r:WORKS_FOR]->(acme)
ON CREATE SET r.description = "CEO position"

-- Chunk 2 extracts the same relationship:
MERGE (john:Person {id: "John Smith"})         -- REUSES existing node
MERGE (acme:Organization {id: "Acme Corporation"})  -- REUSES existing node
MERGE (john)-[r:WORKS_FOR]->(acme)             -- REUSES existing relationship
ON MATCH SET r.description = "CEO position; mentioned again in chunk 2"
```

**Key**: `MERGE` on pattern `(source)-[TYPE]->(target)` prevents duplicates!

---

## Retrieval Strategies

### Overview of 6 RAG Modes

| Mode | Index | Graph Expansion | Best For |
|------|-------|-----------------|----------|
| **vector** | Chunk embeddings | None | Simple Q&A, fact retrieval |
| **fulltext** | Chunk embeddings + keyword | None | Exact term matching |
| **graph_vector** | Chunk embeddings | 1-2 hop dynamic | Complex Q&A, relationships |
| **graph_vector_fulltext** | Chunk + keyword | 1-2 hop dynamic | Most comprehensive |
| **entity_vector** | Entity embeddings | Local community | Entity-centric queries |
| **global_vector** | Community embeddings | Global communities | Topic overview |

### 1. VECTOR Mode (Baseline)

**Index**: `vector` on `Chunk.embedding`

**Query** ([`constants.py:724-786`](backend/src/shared/constants.py#L724-L786)):

```cypher
-- 1. Vector similarity search
CALL db.index.vector.queryNodes('vector', $top_k, $embedding)
YIELD node AS chunk, score

-- 2. Retrieve document metadata
MATCH (chunk)-[:PART_OF]->(d:Document)
WHERE d.fileName IN $fileNames

-- 3. Aggregate and return
WITH d,
     collect(DISTINCT {chunk: chunk, score: score}) AS chunks,
     avg(score) AS avg_score
WITH d, avg_score,
     [c IN chunks | c.chunk.text] AS texts,
     [c IN chunks | {id: c.chunk.id, score: c.score}] AS chunkdetails

RETURN
    reduce(text = "", t IN texts | text + t + "\n") AS text,
    avg_score AS score,
    {
        source: d.fileName,
        chunkdetails: chunkdetails
    } AS metadata
```

**Use case**: Simple similarity-based retrieval, fast and straightforward.

---

### 2. GRAPH_VECTOR Mode (Most Powerful)

**File**: [`backend/src/shared/constants.py:341-505`](backend/src/shared/constants.py#L341-L505)

This is the most sophisticated mode, combining vector search with intelligent graph expansion.

#### Complete Query (Simplified)

```cypher
-- ============================================
-- STEP 1: Vector Search on Chunks
-- ============================================
CALL db.index.vector.queryNodes('vector', $top_k, $embedding)
YIELD node AS chunk, score

MATCH (chunk)-[:PART_OF]->(d:Document)
WHERE d.fileName IN $fileNames

WITH d,
     collect(DISTINCT {chunk: chunk, score: score}) AS chunks,
     avg(score) AS avg_score

-- ============================================
-- STEP 2: Extract Entities from Chunks
-- ============================================
CALL {
    WITH chunks
    UNWIND chunks AS chunkScore
    WITH chunkScore.chunk AS chunk

    -- Get entities from chunk
    OPTIONAL MATCH (chunk)-[:HAS_ENTITY]->(e:__Entity__)

    -- Count frequency and limit
    WITH e, count(*) AS freq
    WHERE freq >= 1
    ORDER BY freq DESC
    LIMIT 40  -- VECTOR_GRAPH_SEARCH_ENTITY_LIMIT

    RETURN collect(e) AS entities
}

-- ============================================
-- STEP 3: Calculate Entity Similarity
-- ============================================
UNWIND entities AS e
WITH e, chunks,
     vector.similarity.cosine(
         e.embedding,
         $embedding
     ) AS entity_similarity

-- ============================================
-- STEP 4: ⭐ INTELLIGENT GRAPH EXPANSION ⭐
-- ============================================
WITH e, entity_similarity, chunks,
     CASE
         -- Low similarity (<30%): Ignore entity
         WHEN entity_similarity < 0.3 OR e.embedding IS NULL THEN
             []

         -- Medium similarity (30-90%): Expand 1 hop
         WHEN entity_similarity >= 0.3 AND entity_similarity <= 0.9 THEN
             collect {
                 OPTIONAL MATCH path = (e)-[r:!HAS_ENTITY&!PART_OF]-
                                       (neighbor:!Chunk&!Document)
                 RETURN path
                 LIMIT 20
             }

         -- High similarity (>90%): Expand 2 hops
         WHEN entity_similarity > 0.9 THEN
             collect {
                 OPTIONAL MATCH path = (e)-[r*1..2]-
                                       (neighbor:!Chunk&!Document)
                 RETURN path
                 LIMIT 40
             }
     END AS expanded_paths

-- ============================================
-- STEP 5: Build Final Context
-- ============================================
WITH chunks,
     collect(DISTINCT e) AS entities,
     apoc.coll.flatten(collect(expanded_paths)) AS paths

WITH chunks,
     [e IN entities | {id: e.id, description: e.description}] AS entity_info,
     [p IN paths | relationships(p)] AS relationship_info

-- ============================================
-- STEP 6: Format and Return
-- ============================================
RETURN
    -- Chunk text
    reduce(text = "", c IN chunks | text + c.chunk.text + "\n") AS text,

    -- Average similarity score
    avg_score AS score,

    -- Rich metadata
    {
        source: d.fileName,
        chunkdetails: [...],
        entities: {
            entityids: [elementId(e) for e IN entities],
            relationshipids: [elementId(r) for r IN relationships]
        }
    } AS metadata
```

#### Expansion Strategy Visualization

**Query**: "What are John Smith's achievements?"

```
Step 1: Vector Search
  └─> Chunk: "John Smith is CEO of Acme Corporation..."

Step 2: Extract Entities
  └─> Person:John Smith (frequency: 3 chunks)

Step 3: Calculate Similarity
  └─> entity_similarity = 0.95 (high!)

Step 4: Expand 2 Hops (because similarity > 0.9)

  Person:John Smith
      │
      ├─ [WORKS_FOR] ──> Organization:Acme Corporation (1 hop)
      │                      │
      │                      └─ [USES] ──> Technology:OpenAI GPT (2 hops)
      │
      ├─ [GRADUATED_FROM] ──> Organization:MIT (1 hop)
      │                            │
      │                            └─ [LOCATED_IN] ──> Location:Cambridge (2 hops)
      │
      └─ [PREVIOUSLY_WORKED_AT] ──> Organization:Tech Innovations (1 hop)
```

**Final context includes**:
- ✅ Original chunk text
- ✅ Entity: John Smith (with description)
- ✅ Entity: Acme Corporation (1 hop)
- ✅ Entity: OpenAI GPT (2 hops)
- ✅ Entity: MIT (1 hop)
- ✅ Entity: Cambridge (2 hops)
- ✅ Entity: Tech Innovations (1 hop)
- ✅ All relationships between them

---

### 3. ENTITY_VECTOR Mode (Local Community)

**Index**: `entity_vector` on `Entity.embedding`

**Query** ([`constants.py:521-563`](backend/src/shared/constants.py#L521-L563)):

```cypher
-- Vector search directly on entities
CALL db.index.vector.queryNodes('entity_vector', $top_k, $embedding)
YIELD node, score

WITH collect(node) AS entities, avg(score) AS avg_score

-- Retrieve related chunks (by entity frequency)
WITH entities, avg_score,
     collect {
         UNWIND entities AS e
         MATCH (e)<-[:HAS_ENTITY]-(c:Chunk)
         WITH c, count(DISTINCT e) AS freq
         ORDER BY freq DESC
         LIMIT 3
         RETURN c
     } AS chunks,

     -- Retrieve local communities
     collect {
         UNWIND entities AS e
         OPTIONAL MATCH (e)-[:IN_COMMUNITY]->(comm:__Community__)
         WITH comm, comm.community_rank AS rank, comm.weight AS weight
         WHERE comm IS NOT NULL
         ORDER BY rank, weight DESC
         LIMIT 3
         RETURN comm
     } AS communities,

     -- Retrieve external relationships
     collect {
         UNWIND entities AS e
         MATCH (e)-[r]-(external:__Entity__)
         WHERE NOT external IN entities
         WITH external, collect(DISTINCT r) AS rels, count(*) AS freq
         ORDER BY freq DESC
         LIMIT 10
         RETURN {node: external, rels: rels}
     } AS outside

RETURN {
    chunks: [c IN chunks | c.text],
    communities: [comm IN communities | comm.summary],
    entities: [e IN entities | {id: e.id, description: e.description}],
    outside: outside
}
```

**Use case**: Entity-centric queries like "Tell me about Tesla" or "Who is Elon Musk?"

---

### 4. GLOBAL_VECTOR Mode (Global Communities)

**Index**: `community_vector` on `Community.embedding`

**Query** ([`constants.py:564-610`](backend/src/shared/constants.py#L564-L610)):

```cypher
-- Vector search on community summaries
CALL db.index.vector.queryNodes('community_vector', $top_k, $embedding)
YIELD node, score

WITH collect(DISTINCT {community: node, score: score}) AS communities,
     avg(score) AS avg_score

WITH avg_score,
     [c IN communities | c.community.summary] AS texts,
     [c IN communities | {
         id: elementId(c.community),
         score: c.score,
         rank: c.community.community_rank
     }] AS communityDetails

RETURN
    reduce(text = "", t IN texts | text + t + "\n") AS text,
    avg_score AS score,
    {communitydetails: communityDetails} AS metadata
```

**Use case**: High-level questions about document themes: "What are the main topics?" or "Summarize the key concepts"

---

### Retrieval Pipeline in QA_integration.py

#### Retriever Setup

**File**: [`backend/src/QA_integration.py:398-410`](backend/src/QA_integration.py#L398-L410)

```python
def get_neo4j_retriever(graph, document_names, chat_mode_settings, score_threshold=0.5):
    """
    Creates a Neo4j vector store retriever with custom Cypher query.
    """
    neo_db = Neo4jVector.from_existing_graph(
        embedding=EMBEDDING_FUNCTION,
        index_name=chat_mode_settings["index_name"],  # "vector", "entity_vector", etc.
        retrieval_query=chat_mode_settings["retrieval_query"],  # Custom Cypher
        graph=graph,
        search_type="hybrid",  # Combines vector + keyword if available
        node_label=chat_mode_settings["node_label"],  # "Chunk", "__Entity__", etc.
        embedding_node_property="embedding",
        text_node_properties=["text"],
        keyword_index_name=chat_mode_settings.get("keyword_index")
    )

    retriever = neo_db.as_retriever(
        search_type="similarity_score_threshold",
        search_kwargs={
            'top_k': chat_mode_settings["top_k"],
            'score_threshold': score_threshold,
            'filter': {'fileName': {'$in': document_names}}  # Document filtering
        }
    )

    return retriever
```

#### Compression Pipeline

**File**: [`backend/src/QA_integration.py:293-333`](backend/src/QA_integration.py#L293-L333)

To reduce noise and stay within token limits:

```python
def create_document_retriever_chain(llm, retriever):
    """
    Creates a retrieval chain with compression and query transformation.
    """
    # Transform query based on chat history
    query_transform_prompt = ChatPromptTemplate.from_messages([
        ("system", QUESTION_TRANSFORM_TEMPLATE),
        MessagesPlaceholder(variable_name="messages")
    ])

    # Compression pipeline
    splitter = TokenTextSplitter(
        chunk_size=3000,
        chunk_overlap=0
    )

    embeddings_filter = EmbeddingsFilter(
        embeddings=EMBEDDING_FUNCTION,
        similarity_threshold=0.10  # Filter out low-relevance docs
    )

    pipeline_compressor = DocumentCompressorPipeline(
        transformers=[splitter, embeddings_filter]
    )

    compression_retriever = ContextualCompressionRetriever(
        base_compressor=pipeline_compressor,
        base_retriever=retriever
    )

    # Query transformation branch
    query_transforming_retriever_chain = RunnableBranch(
        (
            # First question: use as-is
            lambda x: len(x.get("messages", [])) == 1,
            (lambda x: x["messages"][-1].content) | compression_retriever,
        ),
        # Subsequent questions: transform using chat history
        query_transform_prompt | llm | output_parser | compression_retriever,
    )

    return query_transforming_retriever_chain
```

---

## API Reference

### Document Processing Endpoints

#### POST `/upload`
Upload a file (supports chunked uploads for large files).

**Parameters**:
- `file`: File data (multipart/form-data)
- `chunkNumber`: Current chunk number
- `totalChunks`: Total number of chunks
- `originalname`: Original filename
- Neo4j connection params (uri, userName, password, database)

**Response**:
```json
{
  "status": "Success",
  "file_name": "document.pdf",
  "data": {...}
}
```

---

#### POST `/url/scan`
Scan and create source nodes from remote sources.

**Parameters**:
```json
{
  "uri": "neo4j://localhost:7687",
  "userName": "neo4j",
  "password": "password",
  "database": "neo4j",
  "source_url": "https://example.com",
  "source_type": "s3 bucket" | "gcs bucket" | "web-url" | "youtube" | "Wikipedia",
  "model": "openai_gpt_4o",
  "aws_access_key_id": "...",  // For S3
  "gcs_project_id": "...",  // For GCS
  "wiki_query": "..."  // For Wikipedia
}
```

**Response**:
```json
{
  "status": "Success",
  "data": [
    {"fileName": "file1.pdf", "status": "Success"},
    {"fileName": "file2.pdf", "status": "Failed"}
  ]
}
```

---

#### POST `/extract`
Extract knowledge graph from a document.

**Parameters**:
```json
{
  "uri": "neo4j://localhost:7687",
  "userName": "neo4j",
  "password": "password",
  "database": "neo4j",
  "file_name": "document.pdf",
  "model": "openai_gpt_4o",
  "allowedNodes": "Person,Organization,Technology",
  "allowedRelationship": "Person,WORKS_FOR,Organization,Organization,USES,Technology",
  "token_chunk_size": 100,
  "chunk_overlap": 20,
  "chunks_to_combine": 6,
  "retry_condition": "start_from_beginning" | "start_from_last_processed_position",
  "additional_instructions": "Custom instructions for LLM..."
}
```

**Response**:
```json
{
  "status": "Completed",
  "fileName": "document.pdf",
  "nodeCount": 150,
  "relationshipCount": 230,
  "chunkNodeCount": 25,
  "entityNodeCount": 125,
  "communityNodeCount": 15,
  "total_processing_time": 45.32
}
```

---

#### GET `/update_extract_status/{file_name}` (SSE)
Real-time progress updates via Server-Sent Events.

**Parameters**: Query params (uri, userName, password, database)

**Response Stream**:
```json
{"fileName": "doc.pdf", "status": "Processing", "processed_chunk": 20, "nodeCount": 45}
{"fileName": "doc.pdf", "status": "Processing", "processed_chunk": 40, "nodeCount": 87}
{"fileName": "doc.pdf", "status": "Completed", "processed_chunk": 100, "nodeCount": 150}
```

---

### Query Endpoints

#### POST `/sources_list`
List all documents in the database.

**Response**:
```json
{
  "status": "Success",
  "data": [
    {
      "fileName": "doc1.pdf",
      "fileSize": 1024000,
      "status": "Completed",
      "nodeCount": 150,
      "model": "openai_gpt_4o",
      "created_at": "2024-01-15T10:30:00"
    }
  ]
}
```

---

#### POST `/graph_query`
Query the graph for visualization.

**Parameters**:
```json
{
  "uri": "neo4j://localhost:7687",
  "userName": "neo4j",
  "password": "password",
  "database": "neo4j",
  "document_names": ["doc1.pdf", "doc2.pdf"]
}
```

**Response**:
```json
{
  "status": "Success",
  "data": {
    "nodes": [
      {"id": "elem_id_1", "labels": ["Person"], "properties": {"id": "John Smith", "description": "..."}},
      {"id": "elem_id_2", "labels": ["Organization"], "properties": {"id": "Acme Corp", "description": "..."}}
    ],
    "relationships": [
      {"id": "rel_id_1", "type": "WORKS_FOR", "startNode": "elem_id_1", "endNode": "elem_id_2"}
    ]
  }
}
```

---

### Chat Endpoints

#### POST `/chat_bot`
Ask questions using RAG.

**Parameters**:
```json
{
  "uri": "neo4j://localhost:7687",
  "userName": "neo4j",
  "password": "password",
  "database": "neo4j",
  "question": "What does John Smith do?",
  "document_names": ["doc1.pdf"],
  "session_id": "session-uuid",
  "mode": "graph_vector_fulltext",
  "model": "openai_gpt_4o"
}
```

**Response**:
```json
{
  "status": "Success",
  "data": {
    "message": "John Smith is the CEO of Acme Corporation...",
    "info": {
      "sources": ["doc1.pdf"],
      "model": "openai_gpt_4o",
      "entities": {
        "entityids": ["elem_id_1", "elem_id_2"],
        "relationshipids": ["rel_id_1"]
      },
      "total_tokens": 1250,
      "response_time": 2.5
    }
  }
}
```

---

### Post-Processing Endpoints

#### POST `/post_processing`
Execute post-processing tasks.

**Parameters**:
```json
{
  "uri": "neo4j://localhost:7687",
  "userName": "neo4j",
  "password": "password",
  "database": "neo4j",
  "tasks": [
    "materialize_text_chunk_similarities",
    "enable_hybrid_search_and_fulltext_search_in_bloom",
    "enable_communities",
    "materialize_entity_similarities"
  ],
  "document_names": ["doc1.pdf"]
}
```

**Response**:
```json
{
  "status": "Success",
  "message": "Post-processing completed",
  "data": {
    "communityCount": 15,
    "similarityCount": 120
  }
}
```

---

## Configuration

### Environment Variables

#### Neo4j Configuration
```bash
NEO4J_URI=neo4j://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password
NEO4J_DATABASE=neo4j
NEO4J_USER_AGENT=llm-graph-builder
ENABLE_USER_AGENT=true
```

#### LLM API Keys
```bash
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=...
ANTHROPIC_API_KEY=...
DIFFBOT_API_KEY=...
GROQ_API_KEY=...
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

#### LLM Model Configuration
```bash
# Format: LLM_MODEL_CONFIG_{model_name}=model_id,api_key_var
LLM_MODEL_CONFIG_openai_gpt_4o=gpt-4o-2024-11-20,openai_api_key
LLM_MODEL_CONFIG_gemini_1.5_pro=gemini-1.5-pro-002
LLM_MODEL_CONFIG_anthropic_claude_4_sonnet=claude-sonnet-4,api_key
LLM_MODEL_CONFIG_diffbot=diffbot
```

#### Embedding Configuration
```bash
EMBEDDING_MODEL=all-MiniLM-L6-v2  # or "openai" or "vertexai"
IS_EMBEDDING=true
KNN_MIN_SCORE=0.94
ENTITY_EMBEDDING=true
```

#### Processing Parameters
```bash
NUMBER_OF_CHUNKS_TO_COMBINE=6
UPDATE_GRAPH_CHUNKS_PROCESSED=20
MAX_TOKEN_CHUNK_SIZE=10000
CHAT_TOKEN_CUT_OFF=28  # For GPT-4
```

#### Cloud Storage
```bash
GCS_FILE_CACHE=False  # True for GCS, False for local
BUCKET=llm-graph-builder-upload
```

#### Logging and Tracing
```bash
GCP_LOG_METRICS_ENABLED=False
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=...
LANGCHAIN_PROJECT=llm-graph-builder
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
```

### Docker Deployment

**docker-compose.yml**:
```yaml
services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - NEO4J_URI=${NEO4J_URI-neo4j://database:7687}
      - NEO4J_PASSWORD=${NEO4J_PASSWORD}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - EMBEDDING_MODEL=all-MiniLM-L6-v2
    networks:
      - net

  frontend:
    build: ./frontend
    ports:
      - "8080:8080"
    environment:
      - VITE_BACKEND_API_URL=http://localhost:8000
    depends_on:
      - backend
    networks:
      - net

networks:
  net:
```

**Start the stack**:
```bash
docker-compose up -d
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| [`backend/score.py`](backend/score.py) | 1095 | FastAPI entry point, all API endpoints |
| [`backend/src/main.py`](backend/src/main.py) | ~900 | Document processing pipeline |
| [`backend/src/llm.py`](backend/src/llm.py) | ~270 | LLM factory and graph transformer |
| [`backend/src/graphDB_dataAccess.py`](backend/src/graphDB_dataAccess.py) | ~800 | Neo4j data access layer |
| [`backend/src/QA_integration.py`](backend/src/QA_integration.py) | ~800 | RAG implementation (6 modes) |
| [`backend/src/make_relationships.py`](backend/src/make_relationships.py) | ~90 | Entity merging and chunk-entity linking |
| [`backend/src/shared/constants.py`](backend/src/shared/constants.py) | ~910 | Cypher queries for all RAG modes |
| [`backend/src/communities.py`](backend/src/communities.py) | ~300 | Community detection with Leiden |
| [`backend/src/post_processing.py`](backend/src/post_processing.py) | ~240 | Indexing and schema consolidation |

---

## Key Takeaways

### 🎯 Entity Merging
- **`apoc.merge.node`** prevents duplicates based on `id` property
- **`baseEntityLabel=True`** enables automatic merging across chunks
- Properties are **updated** on each match (can concatenate or replace)

### 🔗 Chunk-Entity Linking
- **`HAS_ENTITY`** relationships track which chunks mention which entities
- Enables **provenance** tracking and **frequency scoring**
- Critical for retrieval and context expansion

### 🧠 Intelligent Graph Expansion
- **Low similarity** (<30%): Ignore entity
- **Medium similarity** (30-90%): Expand **1 hop** (max 20 relationships)
- **High similarity** (>90%): Expand **2 hops** (max 40 relationships)

### 🎚️ Quality Control
- **Compression pipeline** filters low-relevance documents (score < 0.10)
- **Token cutoff** based on LLM model (GPT-4: 28, Gemini: 120)
- **Sorting** by similarity score (descending)

### 🚀 Performance Optimization
- **Chunk combination** reduces LLM API calls (100 chunks → 17 calls)
- **Batch processing** (update every 20 chunks)
- **Async operations** for I/O-bound tasks
- **8 Gunicorn workers** with 8 threads each (64 concurrent requests)

---

## Best Practices

### When to Use Each RAG Mode

| Scenario | Recommended Mode | Reason |
|----------|------------------|--------|
| Simple fact lookup | `vector` | Fast, direct similarity |
| Technical documentation | `fulltext` | Exact term matching |
| Complex reasoning | `graph_vector_fulltext` | Most comprehensive |
| Entity-focused query | `entity_vector` | Direct entity search |
| Topic overview | `global_vector` | High-level summaries |

### Schema Design Tips

1. **Keep allowed_nodes broad** initially, then refine based on results
2. **Use semantic relationships**: Prefer `WORKS_FOR` over generic `RELATED_TO`
3. **Limit relationship types** to avoid graph explosion
4. **Use properties wisely**: Dates, numbers, etc. as properties, not nodes

### Performance Tuning

1. **Chunk size**: Smaller chunks (100 tokens) = better precision, more API calls
2. **Chunks to combine**: Higher (6-10) = better context, fewer calls, more cost
3. **Top K**: Start with 5-10 for retrieval, adjust based on precision/recall
4. **Score threshold**: 0.5 default, increase (0.7) for precision, decrease (0.3) for recall

---

## Conclusion

This **LLM Graph Builder** represents a state-of-the-art implementation of GraphRAG, combining:

- **Smart entity deduplication** via APOC merge
- **Multi-hop graph traversal** with dynamic expansion
- **Hybrid search** (vector + full-text + graph)
- **Hierarchical community detection**
- **Comprehensive RAG modes** for different use cases

The system is production-ready, scalable, and highly configurable for various knowledge extraction and Q&A scenarios.

---

**For questions or contributions**, please visit the [GitHub repository](https://github.com/neo4j-labs/llm-graph-builder).
