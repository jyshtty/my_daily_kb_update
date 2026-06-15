## Pydantic vs TypedDict

| Feature | TypedDict | Pydantic |
|---|---|---|
| **Validation** | None (type hints only) | Runtime validation |
| **Error handling** | Silent / type checker only | Raises `ValidationError` |
| **Default values** | Not supported | Supported |
| **Serialization** | Manual | Built-in `.model_dump()`, `.model_dump_json()` |
| **Performance** | Faster (no overhead) | Slower (validation cost) |
| **Use case** | Static typing, lightweight structs | APIs, LLM outputs, data parsing |

---

## Examples

### 1. Validation

**TypedDict** — no runtime validation, wrong types pass silently:

```python
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int

user: User = {"name": "Alice", "age": "not_a_number"}  # No error raised!
print(user)  # {'name': 'Alice', 'age': 'not_a_number'}
```

**Pydantic** — validates at runtime and raises an error:

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Alice", age="not_a_number")  # Raises ValidationError!
```

---

### 2. Default Values

**TypedDict** — does not support default values:

```python
from typing import TypedDict

class Config(TypedDict):
    host: str
    port: int  # Cannot set a default here

config: Config = {"host": "localhost", "port": 8080}  # Must always provide all keys
```

**Pydantic** — supports default values:

```python
from pydantic import BaseModel

class Config(BaseModel):
    host: str = "localhost"
    port: int = 8080

config = Config()  # Uses defaults
print(config)  # host='localhost' port=8080
```

---

### 3. Serialization

**TypedDict** — manual serialization with `json.dumps`:

```python
import json
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int

user: User = {"name": "Alice", "age": 30}
print(json.dumps(user))  # '{"name": "Alice", "age": 30}'
```

**Pydantic** — built-in `.model_dump()` and `.model_dump_json()`:

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Alice", age=30)
print(user.model_dump())       # {'name': 'Alice', 'age': 30}
print(user.model_dump_json())  # '{"name":"Alice","age":30}'
```

---

### 4. Error Handling

**TypedDict** — no error, wrong data goes unnoticed at runtime:

```python
from typing import TypedDict

class Order(TypedDict):
    item: str
    quantity: int

order: Order = {"item": "book", "quantity": "five"}  # Silent — no error
```

**Pydantic** — raises `ValidationError` with detailed info:

```python
from pydantic import BaseModel, ValidationError

class Order(BaseModel):
    item: str
    quantity: int

try:
    order = Order(item="book", quantity="five")
except ValidationError as e:
    print(e)
# 1 validation error for Order
# quantity
#   Input should be a valid integer [type=int_parsing ...]
```

---

### 5. Use Case: LLM Output Parsing (Pydantic shines here)

```python
from pydantic import BaseModel
from anthropic import Anthropic

client = Anthropic()

class MovieReview(BaseModel):
    title: str
    rating: float
    summary: str

# Pydantic ensures the LLM output is valid and typed
data = {"title": "Inception", "rating": 9.2, "summary": "A mind-bending thriller."}
review = MovieReview(**data)
print(review.model_dump_json())
```

---

### When to Use Which?

- Use **TypedDict** when you need lightweight struct definitions with no overhead (e.g., internal configs, simple data containers).
- Use **Pydantic** when you need runtime validation, default values, or are working with external data like APIs or LLM outputs.
