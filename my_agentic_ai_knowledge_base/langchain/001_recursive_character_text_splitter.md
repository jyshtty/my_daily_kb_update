# RecursiveCharacterTextSplitter

## What It Does
Splits text by trying a list of separators in order — from largest logical unit (paragraphs) down to individual characters. Stops splitting once chunks are within `chunk_size`.

## Default Separator Order
```
["\n\n", "\n", " ", ""]
```
Tries double newline first (paragraphs), then single newline, then space, then individual characters as a last resort.

## Basic Usage
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
)

chunks = splitter.split_text(text)
# or for Document objects:
chunks = splitter.split_documents(documents)
```

## Key Parameters

| Parameter | Description | Default |
|---|---|---|
| `chunk_size` | Max size of each chunk (chars or tokens) | 4000 |
| `chunk_overlap` | Shared characters between adjacent chunks | 200 |
| `separators` | List of separators tried in order | `["\n\n", "\n", " ", ""]` |
| `length_function` | How chunk size is measured | `len` |
| `keep_separator` | Keep separator at start/end of chunk | `False` |
| `add_start_index` | Add `start_index` metadata to each chunk | `False` |
| `is_separator_regex` | Treat separators as regex patterns | `False` |

## Custom Separators Example
```python
# Split by sentences (approximate)
splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=20,
    separators=[".", "!", "?", "\n\n"],
    keep_separator=True,
)
```

## Token-Based Splitting
Use when passing chunks to an LLM (models count tokens, not chars):
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    model_name="gpt-4o",
    chunk_size=500,
    chunk_overlap=50,
)
```

## Language-Aware Splitting
Built-in support for code splitting by language syntax:
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50,
)
```
Supported: `PYTHON`, `JS`, `TS`, `MARKDOWN`, `HTML`, `JAVA`, `GO`, `CPP`, `RUBY`, `RUST`, `SCALA`, `SWIFT`, `KOTLIN`, `SOL`

## With Metadata (add_start_index)
```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=20,
    add_start_index=True,   # adds {"start_index": N} to chunk metadata
)
chunks = splitter.split_documents(docs)
# chunks[0].metadata → {"source": "...", "start_index": 0}
```

## Why Use It Over CharacterTextSplitter
- `CharacterTextSplitter` splits on a **single** separator only
- `RecursiveCharacterTextSplitter` **falls back** through multiple separators — avoids mid-word or mid-sentence cuts
- Default choice for most RAG pipelines

## Key Insight from Notebook (008_beginner_chunks)
Chunk size affects similarity quality:
- Too large → embedding captures noise, similarity degrades
- Too small → loses context needed for accurate answers
- Splitting mid-sentence can reverse meaning (e.g. truncating negation)
- Use `chunk_overlap` to preserve context at boundaries (10–20% of `chunk_size`)

## Quick Decision Guide
| Situation | Recommended config |
|---|---|
| General text / PDFs | `chunk_size=500, chunk_overlap=50` |
| Code files | Use `from_language()` |
| LLM input (token budget matters) | Use `from_tiktoken_encoder()` |
| Markdown / structured docs | Consider `MarkdownHeaderTextSplitter` first |
| Need to track source position | `add_start_index=True` |
