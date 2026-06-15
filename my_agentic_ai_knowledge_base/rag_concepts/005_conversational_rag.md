# Conversational RAG (Chat + RAG)

## The Problem

Basic RAG has no memory — every query is treated independently.

```
User: "What is LangChain?"
Bot: [answers]

User: "How do I install it?"  ← "it" refers to LangChain, but RAG does not know that
Bot: [confused, retrieves wrong docs]
```

## Solution: Rephrase Query Using Chat History

Before retrieval, use the LLM to rewrite the query with full context.

```
Chat history: "User asked about LangChain"
Current query: "How do I install it?"
    → Rephrased: "How do I install LangChain?"
    → Now retrieve docs for the rephrased query
```

## LangChain Implementation

```python
from langchain.chains import create_history_aware_retriever
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

# Step 1: Rephrase query using history
contextualize_prompt = ChatPromptTemplate.from_messages([
    ("system", "Rephrase the question to be standalone given the chat history."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])
history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_prompt
)

# Step 2: Answer using retrieved docs
qa_prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer using context: {context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])
qa_chain = create_stuff_documents_chain(llm, qa_prompt)

# Step 3: Combine
rag_chain = create_retrieval_chain(history_aware_retriever, qa_chain)
```

## Passing Chat History

```python
chat_history = []

# Turn 1
response = rag_chain.invoke({"input": "What is LangChain?", "chat_history": chat_history})
chat_history.extend([HumanMessage("What is LangChain?"), AIMessage(response["answer"])])

# Turn 2 — history passed in, query will be rephrased
response = rag_chain.invoke({"input": "How do I install it?", "chat_history": chat_history})
```

## Key Components

| Component | Role |
|---|---|
| `create_history_aware_retriever` | Rephrases query using chat history |
| `create_stuff_documents_chain` | Injects retrieved docs into prompt |
| `create_retrieval_chain` | Wires retriever + QA chain together |
| `chat_history` | List of HumanMessage + AIMessage pairs |

