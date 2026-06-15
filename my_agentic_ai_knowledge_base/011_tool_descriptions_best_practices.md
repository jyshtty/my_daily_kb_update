# Writing Good Tool Descriptions for Agents

## Why Tool Descriptions Matter

The LLM decides which tool to call **based solely on the docstring**.
A bad description = wrong tool called = wrong answer.

## Rules for Good Tool Descriptions

### 1. Be explicit about WHEN to use the tool
```python
# Bad
@tool
def search(query: str) -> str:
    """Search for information."""

# Good
@tool
def search(query: str) -> str:
    """Search the web for current, real-time information.
    Use this when the user asks about recent events, live data,
    or anything that may have changed after your training cutoff."""
```

### 2. Describe inputs clearly
```python
# Bad
@tool
def get_file(path: str) -> str:
    """Get a file."""

# Good
@tool
def get_file(path: str) -> str:
    """Retrieve the contents of a file from the GitHub repository.
    path: relative path from repo root, e.g. src/main.py or README.md"""
```

### 3. Describe what the tool RETURNS
```python
@tool
def get_repository_info(repo_name: str) -> str:
    """Fetch metadata for a GitHub repository.
    Returns: repo name, description, stars, forks, open issues, default branch."""
```

### 4. Distinguish overlapping tools
```python
@tool
def search_docs(query: str) -> str:
    """Search local markdown documentation files for a keyword.
    Use this for internal docs. Use search_web() for external/live information."""

@tool
def search_web(query: str) -> str:
    """Search the internet for current information.
    Use this for real-time data. Use search_docs() for internal documentation."""
```

## Common Mistakes

| Mistake | Why it fails |
|---|---|
| One-word description | LLM cannot distinguish between tools |
| No input format guidance | LLM passes wrong format, tool errors |
| No return description | LLM does not know what to do with result |
| Missing when-NOT-to-use | LLM calls wrong tool for edge cases |

## Key Rule
Write tool descriptions as if explaining to a smart intern who has never seen your codebase.

