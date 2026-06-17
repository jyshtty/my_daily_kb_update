# Beginner LLM Notebooks — Overview

> From notebook: `000_overview.ipynb`  
> A summary index of all beginner notebooks in `llm_notebooks/`.

---

## 001 — First Meet with LLMs
`llm_basics/001_beginner_first_meet.ipynb`

How to connect to and use LLMs via OpenAI SDK, Azure OpenAI SDK, and LangChain.
- Configure API keys and make basic API calls
- Parse LLM responses
- Manage conversation history
- Understand token usage and model selection

---

## 001 — Simple First Connection
`llm_basics/001_beginner_simple-first-connection.ipynb`

Step-by-step guide to connecting to LLMs across three platforms:

| Platform | Required Config |
|---|---|
| OpenAI | `OPENAI_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, API version, endpoint |
| EPAM DIAL | DIAL Key from EPAM chat lab |

Covers: running prompts, parsing responses, counting tokens.

---

## 002 — Embeddings
Text → numeric vectors via embedding models.
- Embedding models generate vectors where semantically similar words are close
- Context shapes meaning — same word, different vectors in different contexts
- Models covered: SBERT (`all-MiniLM-L6-v2`), OpenAI Ada
- Tokens can represent words, subwords, punctuation, or special characters
- Practical tips: offline mode, caching embeddings

---

## 003 — Simple Formatted Output
`prompt_engineering/003_beginner_simple-formatted-ouput.ipynb`

Getting LLMs to return structured output (JSON, XML):
- Instruct the model to output a specific schema
- Handle JSON corruption in long outputs with fix/cleanup functions
- Useful for building pipelines that need machine-readable responses

---

## 004 — Pydantic Output
`prompt_engineering/004_beginner_pydantic-output.ipynb`

Using Pydantic models to parse and validate LLM output:
- Define input/output schemas as Pydantic classes
- Use `PydanticOutputParser` to enforce structure
- Combine with `PromptTemplate` to inject format instructions automatically

---

## 005 — LLM Caching
Cache LLM responses to save cost and time during development:
- `SQLiteCache` — persist responses locally
- Repeated identical queries return instantly from cache
- Best practice: enable caching in dev, disable or scope in production

```python
from langchain.cache import SQLiteCache
from langchain.globals import set_llm_cache

set_llm_cache(SQLiteCache(database_path=".langchain.db"))
```

---

## 006 — LLM Memory
`llm_basics/006_beginner_llm_memory.ipynb`

LLMs are stateless — memory is managed externally by injecting history into the prompt:

| Memory Type | How It Works |
|---|---|
| `ConversationBufferWindowMemory` | Keeps last N turns only |
| `ConversationSummaryMemory` | Summarises older turns to save tokens |

Supports save/load so conversation context can be restored across sessions.

---

## 007 — Document Loaders
Extract plain text from files — essential preprocessing step before chunking and RAG:

| Loader | Use Case |
|---|---|
| `PyPDFLoader` | Standard PDFs |
| `UnstructuredPDFLoader` | PDFs with complex layouts |
| `UnstructuredFileLoader` | General files |
| `OnlinePDFLoader` | PDFs from URLs |
| BeautifulSoup | HTML pages |

Clean text in → better RAG out. "Garbage in, garbage out."

---

## 008 — Chunking
`rag/008_beginner_chunks.ipynb`

Splitting large documents into smaller chunks for embedding and retrieval:
- **Fixed size** — simple, ignores structure
- **Sentence-based** — split on `.`, `!`, `?`, `\n`
- **Recursive** — tries separators in order (default choice)
- **Structure-aware** — Markdown headers, HTML tags
- **Semantic** — split where meaning shifts (embedding-based)

Key insight: chunk too large → noisy embeddings; chunk too small → loses context.  
Use `chunk_overlap` (10–20% of `chunk_size`) to preserve boundary context.

See also: `langchain/001_recursive_character_text_splitter.md`

---

## 009 — Vector Databases
`rag/009_beginner_vectorDB.ipynb`

Store chunk embeddings in a vector DB for fast similarity search (k-NN / ANN):
- **FAISS** — in-process, file-backed, no server needed
- **Qdrant** — dedicated DB, supports native score threshold and rich filters

Key operations: create index, persist, load, search, filter by metadata.

See also: `rag_concepts/013_faiss_and_qdrant.md`

---

## 010 — RAG (Retrieval-Augmented Generation)
`rag/` notebooks

Combines all the above into a full pipeline:
```
Documents → Load → Chunk → Embed → Vector Store
                                        ↓
User Query → Embed → Similarity Search → Relevant Chunks → LLM → Answer
```

See also: `rag_concepts/015_rag_pipeline_hands_on.md`
