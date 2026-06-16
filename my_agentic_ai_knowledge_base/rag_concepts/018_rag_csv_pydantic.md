# RAG with CSV + Pydantic Structured Output

> From notebook: `rag_langchain.ipynb`

## What This Pattern Does
1. Load structured data (CSV) into a ChromaDB vector store
2. Use Pydantic + LLM to extract structured insights from free text (skill gap analysis)
3. Feed those structured insights as queries back into the RAG chain to get recommendations

---

## Step 1: Load CSV into Vector Store
```python
from langchain_community.document_loaders import CSVLoader
from langchain_community.embeddings import SentenceTransformerEmbeddings
from langchain_chroma import Chroma

# Load CSV — each row becomes a Document
loader = CSVLoader("./processed.csv", source_column="agenda name description")
documents = loader.load()

# Embed and store
embedding_function = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")
db = Chroma.from_documents(documents, embedding_function)

# Quick test
docs = db.similarity_search("Python")
print(docs[0].page_content)
```

---

## Step 2: Pydantic Structured Output (Skill Gap Extraction)
Parse free-text feedback into a structured `SkillGapList` object:

### Define Pydantic Models
```python
from pydantic import BaseModel, Field
from typing import Optional

class SkillGap(BaseModel):
    skill: Optional[list[str]] = Field(description="Skill gaps as short phrases")
    technology: Optional[str]  = Field(description="Technology area")

class SkillGapList(BaseModel):
    skill_gaps: Optional[list[SkillGap]] = Field(description="List of skill gaps")
```

### Build Parser Chain
```python
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)
parser = PydanticOutputParser(pydantic_object=SkillGapList)

prompt = ChatPromptTemplate.from_messages([
    ("system", """
Act as a skill gap analyser. Identify skill gaps from the feedback, grouped by technology.
If the input is not feedback, reply with NOT_AN_FEEDBACK.
{format_instructions}
"""),
    ("human", "Identify skill gaps for:\n{text}"),
])

chain = prompt | llm | parser
```

### Run Extraction
```python
feedback = """
Positives:
- Good Core Java and Java 8 knowledge.
- Familiar with design patterns and Spring Boot.
Needs improvement:
- Microservices and Cloud knowledge.
- Python Flask and REST API development.
"""

result = chain.invoke({
    "text": feedback,
    "format_instructions": parser.get_format_instructions(),
})

for gap in result.skill_gaps:
    print(f"Technology: {gap.technology}, Skills: {gap.skill}")
```

---

## Step 3: RAG Chain — Course Recommender
Use each structured skill gap as a query into the course vector store:
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

retriever = db.as_retriever()

template = """You are a course recommender. Recommend the most relevant course for the skill gap.
If no course matches, reply NO_RECOMMENDATION_AVAILABLE.

Available courses:
{context}

Skill gap: {question}
"""

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | ChatPromptTemplate.from_template(template)
    | llm
    | StrOutputParser()
)

# Loop over structured skill gaps and get recommendations
for gap in result.skill_gaps:
    query = f"{gap.technology}: {', '.join(gap.skill or [])}"
    recommendation = rag_chain.invoke(query)
    print(f"Gap:  {query}")
    print(f"Rec:  {recommendation}\n")
```

---

## Key Patterns

### CSVLoader
Each row in the CSV becomes a LangChain `Document`. Set `source_column` to the field that identifies each row:
```python
CSVLoader("file.csv", source_column="name")
```

### PydanticOutputParser
Converts raw LLM text output into a validated Pydantic object. Always pass `format_instructions` to the prompt so the LLM knows the expected JSON schema:
```python
parser.get_format_instructions()  # injects JSON schema into prompt
```

### `db.as_retriever()`
Converts a Chroma vector store into a LangChain `Retriever` interface — compatible with all LangChain chain operators:
```python
retriever = db.as_retriever(search_kwargs={"k": 3})
```

---

## Full Pattern Summary
```
Free-text feedback
    → PydanticOutputParser (structured extraction)
    → SkillGapList (typed Python objects)
    → Loop: each gap → RAG chain query
    → ChromaDB retrieves matching courses
    → LLM generates recommendation
```
This pattern combines **structured extraction** and **semantic retrieval** — useful for any workflow that needs to extract entities from text and then look them up in a knowledge base.
