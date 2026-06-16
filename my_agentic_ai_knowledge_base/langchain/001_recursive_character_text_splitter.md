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

---

# Other LangChain Splitters

## CharacterTextSplitter
Splits on a **single** separator only. Simpler but less robust than recursive.
```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=500,
    chunk_overlap=50,
)
chunks = splitter.split_text(text)
```
- Use when your text has a clear, consistent delimiter
- Falls back to no split if separator not found (may exceed `chunk_size`)

---

## MarkdownHeaderTextSplitter
Splits markdown by header hierarchy and stores header info as metadata.
```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#",   "Header_1"),
    ("##",  "Header_2"),
    ("###", "Header_3"),
]

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on,
    strip_headers=True,   # remove header line from chunk body
)
chunks = splitter.split_text(md_text)
# chunks[0].metadata → {"Header_1": "Intro", "Header_2": "Background"}
```
- Each chunk carries full header breadcrumb in metadata
- Combine with `RecursiveCharacterTextSplitter` to further split large sections
- Best for: markdown docs, wikis, README files

---

## HTMLHeaderTextSplitter
Splits HTML by header tags (`h1`–`h6`), preserving header context as metadata.
```python
from langchain_text_splitters import HTMLHeaderTextSplitter

headers_to_split_on = [
    ("h1", "Header_1"),
    ("h2", "Header_2"),
]

splitter = HTMLHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
chunks = splitter.split_text(html_string)
# or from URL:
chunks = splitter.split_text_from_url("https://example.com/page")
```
- Works on raw HTML strings or URLs
- Best for: web-scraped content, HTML documentation

---

## HTMLSectionSplitter
Splits HTML into sections based on header tags using `xslt` transforms. Returns `Document` objects with richer metadata than `HTMLHeaderTextSplitter`.
```python
from langchain_text_splitters import HTMLSectionSplitter

headers_to_split_on = [("h1", "Header_1"), ("h2", "Header_2")]
splitter = HTMLSectionSplitter(headers_to_split_on=headers_to_split_on)
chunks = splitter.split_text(html_string)
```

---

## TokenTextSplitter
Splits by token count using `tiktoken`. Unlike `from_tiktoken_encoder()`, this splits purely on token boundaries without trying separators.
```python
from langchain_text_splitters import TokenTextSplitter

splitter = TokenTextSplitter(
    encoding_name="cl100k_base",   # gpt-4 / gpt-3.5 encoding
    chunk_size=500,
    chunk_overlap=50,
)
chunks = splitter.split_text(text)
```
- Guarantees chunks never exceed token limit
- May cut mid-sentence — use `RecursiveCharacterTextSplitter.from_tiktoken_encoder()` for cleaner splits

---

## SentenceTransformersTokenTextSplitter
Splits by token count using the **SentenceTransformers** tokenizer (not tiktoken). Use when your embedding model is from `sentence-transformers`.
```python
from langchain_text_splitters import SentenceTransformersTokenTextSplitter

splitter = SentenceTransformersTokenTextSplitter(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    chunk_size=256,       # most SBERT models max at 256–512 tokens
    chunk_overlap=20,
)
chunks = splitter.split_text(text)
```
- Critical: SBERT models silently truncate at their max token limit (often 256/512)
- Use this splitter to ensure chunks fit within the embedding model's window

---

## SemanticChunker
Creates chunks based on **embedding similarity** between sentences — splits where meaning shifts, not at fixed character/token counts.
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # or "standard_deviation", "interquartile"
    breakpoint_threshold_amount=95,
)
chunks = splitter.split_text(text)
```
- Chunk boundaries = semantic topic shifts
- Slower — requires embedding every sentence
- Best for: long mixed-topic documents (articles, books)
- Poor for: uniform text (financial tables, logs)

---

## NLTKTextSplitter
Uses NLTK's sentence tokenizer for sentence-aware splitting.
```python
from langchain_text_splitters import NLTKTextSplitter

splitter = NLTKTextSplitter(chunk_size=500)
chunks = splitter.split_text(text)
```
- Requires: `pip install nltk` + `nltk.download('punkt')`
- Better sentence boundary detection than splitting on `.`
- Handles abbreviations (e.g. `Dr.`, `U.S.`) correctly

---

## SpacyTextSplitter
Uses spaCy's NLP pipeline for sentence segmentation.
```python
from langchain_text_splitters import SpacyTextSplitter

splitter = SpacyTextSplitter(chunk_size=500)
chunks = splitter.split_text(text)
```
- Requires: `pip install spacy` + `python -m spacy download en_core_web_sm`
- Most accurate sentence splitting, handles complex punctuation
- Slowest option — not ideal for large-scale batch processing

---

## LatexTextSplitter
Splits LaTeX documents at logical LaTeX structure boundaries.
```python
from langchain_text_splitters import LatexTextSplitter

splitter = LatexTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_text(latex_text)
```
- Splits on `\chapter`, `\section`, `\subsection`, `\paragraph`, etc.
- Best for: academic papers, technical documents in LaTeX

---

## Quick Decision Guide
| Situation | Splitter |
|---|---|
| General text / PDFs | `RecursiveCharacterTextSplitter` |
| Single clear delimiter | `CharacterTextSplitter` |
| Markdown files | `MarkdownHeaderTextSplitter` |
| HTML / web pages | `HTMLHeaderTextSplitter` |
| Token budget for LLM (OpenAI) | `RecursiveCharacterTextSplitter.from_tiktoken_encoder()` |
| Token budget for SBERT embeddings | `SentenceTransformersTokenTextSplitter` |
| Semantic topic-based chunks | `SemanticChunker` |
| Best sentence detection (accuracy) | `SpacyTextSplitter` |
| Good sentence detection (lighter) | `NLTKTextSplitter` |
| Code files | `RecursiveCharacterTextSplitter.from_language()` |
| LaTeX documents | `LatexTextSplitter` |
