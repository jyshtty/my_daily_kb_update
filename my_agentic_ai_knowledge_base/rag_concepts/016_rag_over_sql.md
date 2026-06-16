# RAG Over Structured Data (SQL)

> From notebook: `simple_rag_over_structured_sql.ipynb`

## What It Is
Instead of embedding documents into a vector store, the LLM translates a natural language query into SQL, executes it against a database, and uses the results as context for a final response.

**Flow:**
```
User Query → LLM generates SQL → Execute on DB → Results as context → LLM generates answer
```

---

## Setup

### LLM
```python
from langchain_openai import AzureChatOpenAI

llm = AzureChatOpenAI(
    api_key=os.environ["OPENAI_API_KEY"],
    api_version="2023-07-01-preview",
    azure_endpoint="https://ai-proxy.lab.epam.com",
    model="gpt-4o-mini-2024-07-18",
    temperature=0.0,
)
```

### Database Connection
Works with any SQLAlchemy-supported database (SQLite, PostgreSQL, MySQL, etc.):
```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///data/cars.db")

# Inspect the database
print(db.dialect)                    # sqlite
print(db.get_usable_table_names())   # ['cars']
print(db.run("SELECT * FROM cars LIMIT 5;"))
```

---

## SQLDatabaseChain
Handles the full NL → SQL → Result pipeline automatically:
```python
from langchain_experimental.sql import SQLDatabaseChain

db_chain = SQLDatabaseChain.from_llm(
    llm=llm,
    db=db,
    verbose=True,       # prints the generated SQL
    return_sql=False,   # return result, not SQL string
    return_direct=True, # skip LLM summarization of result
)

# Natural language queries
db_chain("get cars")
db_chain("give me 3 most expensive cars")
db_chain.run("give me cars without AC")
db_chain("get cars worth more than 1 million")["result"]
```

---

## RAG Chain Over SQL Results
Use SQL results as context for a more sophisticated LLM response:
```python
from langchain_core.prompts import ChatPromptTemplate

RAG_PROMPT = """
You are a professional AI car recommendation assistant.
Recommend cars matching the query from the list below.
If AVAILABLE_CARS is empty, say no cars match.
"""

prompt = ChatPromptTemplate.from_messages([
    ("system", RAG_PROMPT),
    ("human", "HUMAN_QUERY:{human_query}\nAVAILABLE_CARS:{available_cars}"),
])

chain = prompt | llm

# Two-step: SQL retrieval → LLM reasoning
query = "I am looking for SUV and cheap cars with AC (order by price)"
available_cars = db_chain.run(query)

response = chain.invoke({
    "available_cars": available_cars,
    "human_query": query,
})
print(response.content)
```

---

## When to Use SQL RAG vs Vector RAG

| | SQL RAG | Vector RAG |
|---|---|---|
| Data type | Structured (tables, rows) | Unstructured (text, PDFs) |
| Query type | Filtering, aggregation, sorting | Semantic similarity |
| Accuracy | Exact (if SQL is correct) | Approximate |
| Failure mode | LLM generates wrong SQL | Retrieves wrong chunks |
| Best for | Databases, CSVs, spreadsheets | Documents, articles, PDFs |

---

## Gotchas
- `SQLDatabaseChain` is in `langchain_experimental` — not production-hardened
- LLM can generate incorrect SQL for complex schemas — always validate with `verbose=True`
- For production, consider **LangChain SQL Agent** with tools for schema introspection and self-correction
- Large result sets can overflow the LLM context window — add a `LIMIT` or use `return_direct=False` to let the LLM summarize
