# LangChain Cheatsheet

## Mental Model

LangChain is a framework for composing LLM-powered applications. The core abstraction is the **chain**: a sequence of components (prompts, models, retrievers, parsers) connected by the `|` pipe operator via **LCEL** (LangChain Expression Language). Think of it as functional composition where each component is a `Runnable` with a `.invoke()` interface. The most important pattern is RAG: retrieve relevant context, inject it into a prompt, call the LLM.

---

## Install & Minimal Setup

```bash
pip install langchain langchain-community langchain-openai
pip install chromadb                    # vector store
pip install sentence-transformers       # local embeddings
pip install ollama                      # local LLM bridge

# Optional but useful
pip install langchain-core              # base types
pip install langgraph                   # stateful agent graphs
```

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."         # or use python-dotenv
os.environ["ANTHROPIC_API_KEY"] = "sk-ant-..."  # for Claude
```

---

## Core Concepts

### 1. LLMs & Chat Models

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Chat model (always prefer over LLM)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)

# Local via Ollama
from langchain_ollama import ChatOllama
llm = ChatOllama(model="granite3.2", temperature=0)

# Invoke
response = llm.invoke("Explain RAG in one sentence.")
print(response.content)
```

### 2. Prompt Templates

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant specialized in {domain}."),
    ("human", "{question}")
])

# Format and inspect
messages = prompt.format_messages(domain="MLOps", question="What is drift?")

# Works as a Runnable — pipe directly into a model
chain = prompt | llm
response = chain.invoke({"domain": "MLOps", "question": "What is drift?"})
```

### 3. LCEL — LangChain Expression Language
The pipe operator `|` connects Runnables. Data flows left to right.

```python
from langchain_core.output_parsers import StrOutputParser

# Basic chain: prompt → llm → parse to string
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"domain": "RAG", "question": "What is chunking?"})
# result is now a plain string

# Streaming
for chunk in chain.stream({"domain": "RAG", "question": "Explain embeddings"}):
    print(chunk, end="", flush=True)

# Async
result = await chain.ainvoke({"domain": "RAG", "question": "..."})

# Batch
results = chain.batch([
    {"domain": "RAG", "question": "Question 1"},
    {"domain": "RAG", "question": "Question 2"},
])
```

### 4. Output Parsers

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

# String output
parser = StrOutputParser()

# Structured JSON output via Pydantic
class Classification(BaseModel):
    sentiment: str = Field(description="positive, negative, or neutral")
    confidence: float = Field(description="confidence score 0.0-1.0")

parser = JsonOutputParser(pydantic_object=Classification)
prompt = ChatPromptTemplate.from_template(
    "Classify the sentiment of this text. {format_instructions}\n\nText: {text}",
    partial_variables={"format_instructions": parser.get_format_instructions()}
)
chain = prompt | llm | parser
result = chain.invoke({"text": "This product is amazing!"})
# result is a Classification instance
```

### 5. Embeddings

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.embeddings import OllamaEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
embeddings = OllamaEmbeddings(model="nomic-embed-text")

# Embed a single string
vector = embeddings.embed_query("What is MLOps?")     # list[float]

# Embed multiple documents
vectors = embeddings.embed_documents(["doc 1", "doc 2"])
```

### 6. Vector Stores

```python
from langchain_community.vectorstores import Chroma
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient

# ChromaDB — in-memory (dev)
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    collection_name="my_docs"
)

# ChromaDB — persistent
vectorstore = Chroma(
    collection_name="my_docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

# Qdrant — local
client = QdrantClient(":memory:")   # or QdrantClient(url="http://localhost:6333")
vectorstore = QdrantVectorStore(
    client=client,
    collection_name="my_docs",
    embedding=embeddings
)

# Add documents
vectorstore.add_documents(docs)

# Similarity search
results = vectorstore.similarity_search("What is drift?", k=4)
results = vectorstore.similarity_search_with_score("What is drift?", k=4)
```

### 7. Document Loaders & Text Splitters

```python
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Load
loader = PyPDFLoader("document.pdf")
documents = loader.load()

# Split
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,       # overlap preserves context at chunk boundaries
    length_function=len
)
chunks = splitter.split_documents(documents)

# Each chunk has page_content and metadata
print(chunks[0].page_content)
print(chunks[0].metadata)     # {"source": "document.pdf", "page": 0}
```

### 8. Retrievers

```python
# Basic retriever from vector store
retriever = vectorstore.as_retriever(
    search_type="similarity",   # or "mmr" (diversity) or "similarity_score_threshold"
    search_kwargs={"k": 4}
)

# Invoke directly
docs = retriever.invoke("What is model drift?")

# MMR — Maximal Marginal Relevance (reduces redundancy in results)
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 4, "fetch_k": 20}
)
```

### 9. Full RAG Chain (LCEL)

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the following context:

{context}

Question: {question}
""")

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()
    }
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("What causes model drift?")
```

---

## Most-Used Patterns

### RunnableParallel — run branches in parallel

```python
from langchain_core.runnables import RunnableParallel

parallel = RunnableParallel(
    summary=summary_chain,
    keywords=keyword_chain
)
result = parallel.invoke({"text": "..."})
# result = {"summary": "...", "keywords": [...]}
```

### Memory / Chat History

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

history = InMemoryChatMessageHistory()

chain_with_history = RunnableWithMessageHistory(
    chain,
    lambda session_id: history,
    input_messages_key="question",
    history_messages_key="chat_history"
)

chain_with_history.invoke(
    {"question": "What is RAG?"},
    config={"configurable": {"session_id": "user-1"}}
)
```

### Tools & Tool Calling

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"Sunny, 25°C in {city}"

llm_with_tools = llm.bind_tools([get_weather])
response = llm_with_tools.invoke("What's the weather in Mexico City?")
```

---

## Gotchas

- **LCEL input/output types must match** — each component receives the output of the previous one. A dict output won't work if the next component expects a string.
- **`RunnablePassthrough` vs dict** — use `RunnablePassthrough()` to forward the input unchanged, use `{"key": RunnablePassthrough()}` to route parts of the input.
- **Chunk overlap too low** — answers split across chunk boundaries get missed. 10–20% overlap is a safe starting point.
- **Retriever `k` too small** — if relevant context is in document 5 and you set `k=3`, the model never sees it. Start with `k=5` or `k=8` and tune.
- **Embeddings and vector store must match** — you can't index with OpenAI embeddings and query with Ollama embeddings. Lock them together.
- **LangChain versions move fast** — imports shift between major versions. Pin your versions in `requirements.txt`.

---

## Quick Links

- [LangChain Docs](https://python.langchain.com)
- [LCEL Conceptual Guide](https://python.langchain.com/docs/concepts/lcel/)
- [LangSmith](https://smith.langchain.com) — tracing and debugging for chains
- [LangGraph Docs](https://langchain-ai.github.io/langgraph/) — stateful multi-agent graphs
- [RAGAS](https://docs.ragas.io) — RAG evaluation framework
