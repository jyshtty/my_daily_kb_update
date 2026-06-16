# RAG Pipeline Hands-On (ChromaDB + LangChain)

> From notebook: `011_rag_exploration.ipynb`

## RAG in 3 Steps
1. **Document Processing & Splitting** — load, clean, chunk
2. **Vector Store & Retrieval** — embed chunks, find relevant ones by similarity
3. **Answer Generation** — pass retrieved context + query to LLM

---

## Step 1: Load and Split PDFs
```python
from langchain.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

def read_split_into_docs(path: str):
    loader = PyPDFLoader(path)
    documents = loader.load()
    text_splitter = RecursiveCharacterTextSplitter(chunk_size=1024, chunk_overlap=0)
    return text_splitter.split_documents(documents)

structured_docs   = read_split_into_docs("./data/LLM-RISKS.pdf")
unstructured_docs = read_split_into_docs("./data/DELSTRIGO-Patient-Education-Brochure-PDF.pdf")
```

### chunk_size: Characters vs Tokens
`RecursiveCharacterTextSplitter` counts **characters**, but LLMs use **tokens**:
```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
char_count  = len(text)          # e.g. 951
token_count = len(enc.encode(text))  # e.g. 562 — roughly 0.6× chars
```
Rule of thumb: 1 token ≈ 4 characters. Always verify against your model's context limit.

---

## Structured vs Unstructured PDFs
- **Structured** (reports, papers): `PyPDFLoader` works reasonably well
- **Unstructured** (brochures, images, forms): text extraction fails — images return garbled text

Example failure: a brochure image showing "9 out of 10 adults" was extracted as:
```
STAYED UNDETECTABLEMORE\nTHANOUT\nOFADULTS WHO\nCOMPLETED THE STUDY910
```
A simple splitter cannot fix layout-aware documents — use specialized parsers (Unstructured.io, Azure Document Intelligence, etc.) for those.

---

## Step 2: ChromaDB Vector Store
```python
from langchain_chroma import Chroma
from langchain_openai import AzureOpenAIEmbeddings

embeddings = AzureOpenAIEmbeddings(
    deployment="text-embedding-3-small-1",
    dimensions=512,
)

vector_store = Chroma(
    collection_name="my-docs",
    embedding_function=embeddings,
    persist_directory="./data/chroma-db",
)

# Avoid duplicates on re-run
if len(vector_store.get()["ids"]) == 0:
    vector_store.add_documents(structured_docs + unstructured_docs)
```

### Similarity Search
```python
results = vector_store.similarity_search_with_score(
    "Types of Prompt Injection Vulnerabilities",
    k=3,
)
# returns list of (Document, distance_score)
# Lower score = more similar in Chroma (L2 distance by default)
```

### Choosing k
Plot scores across all k results to find the "elbow" where relevance drops off:
```python
results = vector_store.similarity_search_with_score(query, k=140)
scores = [r[1] for r in results]
# Plot scores — set threshold at the elbow point
```
There's no universal k — experiment with your specific data and queries.

---

## Step 3: Answer Generation Chain
```python
from langchain_core.prompts import PromptTemplate
from langchain_core.runnables import chain, RunnablePassthrough

RAG_PROMPT = """Answer the user query based on the context below. Return Markdown.

**Context**: {context}
**Query**: {query}
"""
prompt = PromptTemplate.from_template(RAG_PROMPT)

@chain
def get_documents(request: dict) -> str:
    docs = vector_store.similarity_search_with_score(query=request["query"], k=2)
    return "\n".join([doc[0].page_content for doc in docs])

rag_chain = (
    {"context": get_documents, "query": RunnablePassthrough()}
    | prompt
    | llm
)

response = rag_chain.invoke({"query": "What are types of Prompt Injection Vulnerabilities?"})
```

---

## Distance Metrics in Retrieval

| Metric | Range | Best when |
|---|---|---|
| Cosine Similarity | -1 to 1 | Most RAG use cases (bounded, model-agnostic) |
| L2 (Euclidean) | 0 to ∞ | Normalized vectors |
| Inner Product | -∞ to ∞ | When magnitude carries meaning |

Cosine is safest default — bounded range lets you set consistent thresholds across different document sets.

---

## Garbage In, Garbage Out
The single most impactful RAG improvement is document processing quality:
- Bad extraction → bad chunks → bad retrieval → bad answers
- Fix extraction before tuning k, chunk size, or prompt
