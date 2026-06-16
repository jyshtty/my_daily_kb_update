# Simple RAG with ChromaDB (Vanilla + Hybrid Search)

> From notebook: `simple_rag_unstructured.ipynb`

## RAG Components (4 Fundamentals)
1. **Document Processing** — load, extract, chunk
2. **Retrieval** — embed + similarity search
3. **Re-Rank** (optional) — re-order top-n results
4. **Result Representation** — LLM generates final answer

---

## Step 1: Document Processing
```python
from langchain.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

url_pdf = "https://example.com/document.pdf"
loader = PyPDFLoader(url_pdf)
documents = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1024, chunk_overlap=0)
docs = text_splitter.split_documents(documents)

print(f"Total chunks: {len(docs)}")
```

### Characters vs Tokens Reminder
```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
print("Characters:", len(docs[0].page_content))
print("Tokens:", len(enc.encode(docs[0].page_content)))
```

---

## Step 2: Embeddings

### OpenAI / Azure Embeddings
```python
from langchain_openai import AzureOpenAIEmbeddings

embedding = AzureOpenAIEmbeddings(
    model="text-embedding-ada-002",
    api_key=os.environ["OPENAI_API_KEY"],
    azure_endpoint="https://ai-proxy.lab.epam.com",
)
```

### Open-Source HuggingFace Embeddings (LangChain wrapper)
```python
from sentence_transformers import SentenceTransformer
from langchain.embeddings.base import Embeddings

class SentenceTransformerEmbeddings(Embeddings):
    def __init__(self, model_name: str):
        self.model = SentenceTransformer(model_name)

    def embed_documents(self, documents):
        return [self.model.encode(d).tolist() for d in documents]

    def embed_query(self, query):
        return self.model.encode([query])[0].tolist()

embedding = SentenceTransformerEmbeddings("BAAI/bge-base-en-v1.5")
```

---

## Step 3: ChromaDB Vector Store
```python
from langchain.vectorstores import Chroma

db = Chroma.from_documents(
    docs,
    embedding,
    collection_metadata={"hnsw:space": "cosine"},  # use cosine distance
)
```

### Vanilla Similarity Search
```python
query = "what are the side effects of taking Delstrigo?"
retrieved_docs = db.similarity_search_with_score(query)
```

### Hybrid Search (Similarity + Keyword Filter)
Combine dense vector search with a keyword text filter:
```python
docs_hybrid = db.similarity_search_with_score(
    query,
    k=10,
    where_document={"$contains": "side effects"},  # keyword must appear in chunk
)
```
This is a "vanilla" hybrid — not full BM25+dense fusion, but a practical approximation.

---

## Step 4: Answer Generation
```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key=os.environ["OPENAI_API_KEY"],
    api_version="2023-07-01-preview",
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    azure_deployment="gpt-4-1106-preview",
)

system_prompt = (
    "You are an assistant for question-answering tasks. "
    "Use the retrieved context to answer the question. "
    "If you don't know the answer, say so. "
    "Keep answers concise — three sentences max."
)

response = client.chat.completions.create(
    model="gpt-4-1106-preview",
    temperature=0,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": f"Query: {query}\nDocs: {retrieved_docs}"},
    ],
)
print(response.choices[0].message.content)
```

---

## Choosing an Embedding Model
Key factors when selecting an embedding model:

| Factor | Consideration |
|---|---|
| Context length | Max tokens the model handles per chunk |
| Model size | Larger = more powerful, more compute |
| Cost | Commercial APIs vs free open-source |
| Domain specificity | General vs domain-trained models |

Benchmark reference: [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard)

---

## ChromaDB Distance Modes

| Mode | `hnsw:space` | Lower score = ? |
|---|---|---|
| L2 (default) | `"l2"` | More similar |
| Cosine | `"cosine"` | More similar |
| Inner product | `"ip"` | More similar |

Set `"cosine"` for most RAG use cases — bounded and consistent across document sets.

---

## `db.delete_collection()`
If you change the embedding model or distance metric, you must delete and re-index:
```python
db.delete_collection()
# Then re-run Chroma.from_documents(...)
```
