# RAG Pipeline Components

## The 5 Stages of a RAG Pipeline

```
Documents → [1. Load] → [2. Chunk] → [3. Embed] → [4. Store]
                                                        ↓
User Query → [3. Embed] → [4. Retrieve] → [5. Generate] → Answer
```

## 1. Load
Ingest raw documents from various sources.
```python
from langchain_community.document_loaders import PyPDFLoader, WebBaseLoader
loader = PyPDFLoader("document.pdf")
docs = loader.load()
```

## 2. Chunk (Split)
Break documents into smaller pieces the LLM can fit in context.
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)
```

## 3. Embed
Convert text chunks into vectors (numbers) that capture semantic meaning.
```python
from langchain_openai import OpenAIEmbeddings
embeddings = OpenAIEmbeddings()
vectors = embeddings.embed_documents([c.page_content for c in chunks])
```

## 4. Store & Retrieve
Store vectors in a vector database; retrieve most similar at query time.
```python
from langchain_community.vectorstores import FAISS
vectorstore = FAISS.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
```

## 5. Generate
Feed retrieved chunks + user query to the LLM.
```python
from langchain.chains import RetrievalQA
chain = RetrievalQA.from_chain_type(llm=llm, retriever=retriever)
answer = chain.invoke({"query": user_question})
```

## Full Pipeline Summary

| Stage | Input | Output | Key Decision |
|---|---|---|---|
| Load | Raw files/URLs | Documents | Which loader to use |
| Chunk | Documents | Smaller chunks | Chunk size, overlap |
| Embed | Text chunks | Vectors | Which embedding model |
| Store/Retrieve | Vectors + query | Relevant chunks | k, similarity metric |
| Generate | Chunks + query | Answer | Prompt template |

