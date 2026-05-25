# pgvector Cheatsheet

## Mental Model

pgvector is a **PostgreSQL extension** that adds a `vector` data type and similarity search operators. Instead of a separate vector database, your embeddings live in the same Postgres instance as the rest of your data — joins, transactions, and SQL filters work natively. The tradeoff: simpler stack and operational overhead, at the cost of raw ANN throughput compared to dedicated stores like Qdrant at massive scale. For most production RAG applications under ~10M vectors, pgvector is the right default.

---

## Install & Minimal Setup

```bash
# Install the extension (Ubuntu / Debian)
sudo apt install postgresql-16-pgvector

# macOS
brew install pgvector

# Docker (easiest for dev)
docker run -d \
  --name pgvector-dev \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  pgvector/pgvector:pg16

# Python deps
pip install psycopg2-binary pgvector sqlalchemy asyncpg
```

```sql
-- Enable the extension in your database (run once)
CREATE EXTENSION IF NOT EXISTS vector;
```

---

## Core Concepts

### 1. The `vector` Type

```sql
-- Column stores a fixed-dimension float array
-- Dimension must match your embedding model
CREATE TABLE documents (
    id        BIGSERIAL PRIMARY KEY,
    content   TEXT NOT NULL,
    metadata  JSONB,
    embedding vector(1536)    -- OpenAI text-embedding-3-small
    -- embedding vector(768)  -- nomic-embed-text, sentence-transformers
    -- embedding vector(384)  -- all-MiniLM-L6-v2
);

-- Insert a vector
INSERT INTO documents (content, embedding)
VALUES ('Hello world', '[0.1, 0.2, 0.3, ...]');  -- must match declared dimension
```

### 2. Distance Operators

| Operator | Distance | Use case |
|---|---|---|
| `<->` | L2 / Euclidean | General purpose |
| `<#>` | Negative inner product | Dot product similarity (normalized vectors) |
| `<=>` | Cosine distance | Most common for text embeddings |
| `<+>` | L1 / Manhattan | Sparse vectors |

```sql
-- Find the 5 most similar documents to a query vector
SELECT id, content, embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- Cosine similarity (1 - cosine distance) if you want a score
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

### 3. Indexes

Without an index, every query does a full sequential scan (exact but slow at scale).

```sql
-- IVFFlat — partition-based ANN (good default)
-- Build AFTER inserting data for better cluster quality
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);   -- lists ≈ sqrt(total_rows) is a good starting point

-- HNSW — graph-based ANN (better recall, slower build, more memory)
-- Available since pgvector 0.5.0 — preferred for most use cases
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- ops class must match your distance operator:
-- vector_cosine_ops  → <=>
-- vector_l2_ops      → <->
-- vector_ip_ops      → <#>
```

```sql
-- Tune query-time recall (higher = more accurate, slower)
SET hnsw.ef_search = 100;    -- default 40
SET ivfflat.probes = 10;     -- default 1 — increase for better recall
```

### 4. Hybrid Search (Vector + SQL Filters)

```sql
-- Filter BEFORE vector search for efficiency (metadata filtering)
SELECT id, content, embedding <=> $1 AS distance
FROM documents
WHERE metadata->>'source' = 'legal'       -- SQL filter
  AND metadata->>'year' = '2024'
ORDER BY embedding <=> $1
LIMIT 5;

-- Full-text search + vector search combined
SELECT
    d.id,
    d.content,
    d.embedding <=> $1 AS vec_distance,
    ts_rank(to_tsvector('english', d.content), query) AS text_rank
FROM documents d, plainto_tsquery('english', $2) query
WHERE to_tsvector('english', d.content) @@ query
ORDER BY vec_distance
LIMIT 10;
```

### 5. Schema for RAG Applications

```sql
CREATE TABLE chunks (
    id          BIGSERIAL PRIMARY KEY,
    doc_id      BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    content     TEXT NOT NULL,
    chunk_index INTEGER NOT NULL,
    embedding   vector(768),
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Index for vector search
CREATE INDEX chunks_embedding_idx ON chunks
USING hnsw (embedding vector_cosine_ops);

-- Index for metadata filtering
CREATE INDEX chunks_metadata_idx ON chunks USING gin (metadata);

-- Index for source document lookup
CREATE INDEX chunks_doc_id_idx ON chunks (doc_id);
```

---

## Python Usage

### Raw psycopg2

```python
import psycopg2
from pgvector.psycopg2 import register_vector
import numpy as np

conn = psycopg2.connect("postgresql://user:password@localhost/mydb")
register_vector(conn)   # required — registers the vector type

cur = conn.cursor()

# Insert
embedding = np.array([0.1, 0.2, 0.3])  # your actual embedding
cur.execute(
    "INSERT INTO documents (content, embedding) VALUES (%s, %s)",
    ("Hello world", embedding)
)
conn.commit()

# Query
query_embedding = np.array([0.1, 0.2, 0.3])
cur.execute(
    """
    SELECT id, content, embedding <=> %s AS distance
    FROM documents
    ORDER BY embedding <=> %s
    LIMIT 5
    """,
    (query_embedding, query_embedding)
)
results = cur.fetchall()
```

### SQLAlchemy (ORM)

```python
from sqlalchemy import create_engine, Column, Integer, Text
from sqlalchemy.orm import declarative_base, Session
from pgvector.sqlalchemy import Vector

Base = declarative_base()

class Chunk(Base):
    __tablename__ = "chunks"

    id        = Column(Integer, primary_key=True)
    content   = Column(Text, nullable=False)
    embedding = Column(Vector(768))

engine = create_engine("postgresql+psycopg2://user:password@localhost/mydb")
Base.metadata.create_all(engine)

with Session(engine) as session:
    # Insert
    chunk = Chunk(content="Hello world", embedding=[0.1, 0.2, ...])
    session.add(chunk)
    session.commit()

    # Query — cosine similarity search
    from pgvector.sqlalchemy import cosine_distance

    query_vec = [0.1, 0.2, ...]
    results = (
        session.query(Chunk)
        .order_by(cosine_distance(Chunk.embedding, query_vec))
        .limit(5)
        .all()
    )
```

### Async with asyncpg

```python
import asyncpg
from pgvector.asyncpg import register_vector
import numpy as np

async def search(query_embedding: list[float], k: int = 5):
    conn = await asyncpg.connect("postgresql://user:password@localhost/mydb")
    await register_vector(conn)

    results = await conn.fetch(
        """
        SELECT id, content, embedding <=> $1 AS distance
        FROM chunks
        ORDER BY embedding <=> $1
        LIMIT $2
        """,
        np.array(query_embedding), k
    )
    return results
```

### LangChain Integration

```python
from langchain_community.vectorstores import PGVector
from langchain_openai import OpenAIEmbeddings

CONNECTION_STRING = "postgresql+psycopg2://user:password@localhost/mydb"

vectorstore = PGVector(
    connection_string=CONNECTION_STRING,
    collection_name="documents",
    embedding_function=OpenAIEmbeddings()
)

# Add documents
vectorstore.add_texts(
    texts=["chunk 1 content", "chunk 2 content"],
    metadatas=[{"source": "doc1.pdf"}, {"source": "doc1.pdf"}]
)

# Search
docs = vectorstore.similarity_search("query text", k=5)
docs_with_scores = vectorstore.similarity_search_with_score("query text", k=5)

# As retriever
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
```

---

## Most-Used Patterns

### Batch Insert for Performance

```python
# Use execute_values for bulk inserts — much faster than single inserts
from psycopg2.extras import execute_values

data = [(chunk.content, chunk.embedding) for chunk in chunks]
execute_values(
    cur,
    "INSERT INTO chunks (content, embedding) VALUES %s",
    data,
    template="(%s, %s::vector)"
)
conn.commit()
```

### Refresh Index After Bulk Load

```sql
-- After large bulk inserts, reindex for better ANN quality
REINDEX INDEX chunks_embedding_idx;

-- Or vacuum + analyze to update planner stats
VACUUM ANALYZE chunks;
```

---

## Gotchas

- **Dimension mismatch** — the vector dimension in the table definition must exactly match your embedding model output. Changing it requires `ALTER TABLE` (costly) or rebuilding the table.
- **Index build requires data** — build HNSW/IVFFlat indexes after inserting data, not before. An index built on an empty table won't cluster properly.
- **IVFFlat `lists` parameter** — too few lists → slow scan; too many → poor approximation. Rule of thumb: `lists = sqrt(row_count)`, minimum 10.
- **`register_vector` is required** — forget this in psycopg2/asyncpg and you'll get `can't adapt type 'numpy.ndarray'` errors.
- **Cosine vs L2** — for text embeddings, cosine distance (`<=>`) almost always outperforms L2 (`<->`). Stick to cosine unless you have a specific reason not to.
- **HNSW uses more memory** — each vector stores neighbor links. For 1M vectors at 768 dimensions, plan for ~6–8GB RAM for the index.

---

## Quick Links

- [pgvector GitHub](https://github.com/pgvector/pgvector) — README has benchmarks and tuning guide
- [pgvector-python](https://github.com/pgvector/pgvector-python) — psycopg2, asyncpg, SQLAlchemy adapters
- [LangChain PGVector](https://python.langchain.com/docs/integrations/vectorstores/pgvector/)
- [Supabase Vector](https://supabase.com/docs/guides/ai/vector-columns) — managed Postgres + pgvector with good docs
