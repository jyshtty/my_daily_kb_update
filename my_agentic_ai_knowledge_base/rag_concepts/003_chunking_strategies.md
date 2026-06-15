# Chunking Strategies in RAG

## Why Chunking Matters
Too large → exceeds context window, noisy retrieval
Too small → loses context, retrieval misses meaning

## Strategy 1: Fixed Size (Character Split)
```python
from langchain.text_splitter import CharacterTextSplitter
splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=50)
```
- Simple, fast
- Ignores document structure
- Good for: plain text, logs

## Strategy 2: Recursive Character Split (Default Choice)
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["

", "
", " ", ""]  # tries each in order
)
```
- Respects paragraph → sentence → word boundaries
- Best general-purpose strategy
- Good for: most documents

## Strategy 3: Semantic Chunking
```python
from langchain_experimental.text_splitter import SemanticChunker
splitter = SemanticChunker(embeddings)
```
- Groups sentences with similar meaning together
- Chunk boundaries = topic shifts
- Good for: long documents with mixed topics

## Strategy 4: Document-Structure Aware
```python
from langchain.text_splitter import MarkdownHeaderTextSplitter
splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[("#", "h1"), ("##", "h2")]
)
```
- Splits on document structure (headers, sections)
- Preserves metadata (which section it came from)
- Good for: markdown, HTML, code docs

## Chunk Overlap
Always set overlap so context is not lost at boundaries:
```python
# chunk_overlap = 10-20% of chunk_size is a good rule
chunk_size=500, chunk_overlap=50  # 10% overlap
```

## Quick Reference

| Strategy | Best For | Tradeoff |
|---|---|---|
| Fixed size | Simple text | Ignores structure |
| Recursive | General use | No semantic awareness |
| Semantic | Mixed-topic docs | Slower, needs embeddings |
| Structure-aware | Markdown/HTML | Document-format specific |

