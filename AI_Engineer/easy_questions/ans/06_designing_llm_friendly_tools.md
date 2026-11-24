# Q6. Is there a standard design approach for making tools or APIs understandable to an LLM, including during failures?

[← Back to questions](../questions.md)

## TL;DR

- No single formal standard exists. The common practice, followed by Anthropic, OpenAI and the MCP spec, is to **design tools for an agent the way you would design them for a new junior engineer** who only reads the docs.
- **Names and descriptions:** use clear `verb_noun` names. The description says what the tool does, when to use it and when not to, what each parameter means with formats and examples, and what it returns.
- **Schema:** strict JSON Schema with types, `enum`s, ranges, patterns, required fields and `additionalProperties: false`. Prefer a **few well-scoped, task-level tools** over many thin CRUD wrappers.
- **Outputs:** return concise, relevant, human-readable fields (names, not just UUIDs), with **pagination and truncation** plus a note on how to get more.
- **Failures:** return **structured, actionable errors** (code, message, hint, retryable) as tool results, not exceptions or stack traces. Make write tools **idempotent**.
- Iterate with **evals**: watch real traces and refine descriptions and errors based on where the model goes wrong.

## Detailed Answer

### 1. Why tool design matters

The model only sees the tool **name, description, schema and results**. All of it goes into the context window and competes for attention. Tool definitions are effectively prompts. Poorly designed tools cause wrong tool choice, malformed arguments, hallucinated parameters, loops and wasted tokens.

### 2. Principles

| Principle | Do | Avoid |
|---|---|---|
| **Clear naming** | `search_customer_orders`, `create_support_ticket` | `do_action`, `api_call`, `tool1` |
| **Namespacing** | `github_create_issue`, `jira_create_issue` | Two `create_issue` tools from different servers |
| **Rich description** | Purpose, when to use, when NOT to use, an example | "Gets orders." |
| **Few, task-level tools** | `schedule_meeting(attendees, duration, window)` | `list_users` + `get_calendar` + `find_slot` + `create_event` for one task |
| **Constrained inputs** | `enum`, `pattern`, `minimum`, `maxLength`, defaults | Free-form `params: str` containing JSON |
| **Unambiguous param names** | `user_id`, `start_date (YYYY-MM-DD)` | `user`, `date` |
| **Concise, meaningful output** | `{"name": "Alice", "status": "shipped"}` | 200-field raw API dumps or bare UUIDs |
| **Response format control** | `response_format: "concise" \| "detailed"` | Always returning everything |
| **Pagination / truncation** | `limit`, `cursor`, "showing 20 of 340, pass cursor=..." | Silently truncating |
| **Actionable errors** | "start_date must be YYYY-MM-DD, got '24/11'" | "400 Bad Request" or a stack trace |
| **Idempotency** | `idempotency_key` on writes, safe retries | Duplicate charges when retried |
| **Safety metadata** | MCP annotations `readOnlyHint`, `destructiveHint` | Hiding side effects |

### 3. Anatomy of a good tool definition

```json
{
  "name": "search_orders",
  "description": "Search a customer's orders by status and date range. Use this when the user asks about the status, history or details of their orders. Do NOT use it for refunds (use create_refund). Returns at most `limit` orders, newest first, with a next_cursor if more exist.",
  "input_schema": {
    "type": "object",
    "properties": {
      "customer_id": {"type": "string", "pattern": "^C\\d{6}$",
                      "description": "Customer ID like C000123. Get it from get_customer if you only have an email."},
      "status": {"type": "string", "enum": ["pending", "shipped", "delivered", "cancelled", "any"],
                 "default": "any"},
      "since": {"type": "string", "format": "date",
                "description": "ISO date YYYY-MM-DD. Defaults to 90 days ago."},
      "limit": {"type": "integer", "minimum": 1, "maximum": 50, "default": 10},
      "cursor": {"type": "string", "description": "Opaque cursor from a previous response."}
    },
    "required": ["customer_id"],
    "additionalProperties": false
  }
}
```

### 4. Same tool in Python (MCP / FastMCP with Pydantic)

```python
from typing import Literal, Annotated
from datetime import date, timedelta
from pydantic import BaseModel, Field
from mcp.server.fastmcp import FastMCP
from mcp.types import ToolAnnotations

mcp = FastMCP("orders")

class Order(BaseModel):
    order_id: str
    item_summary: str           # human-readable, not just SKU IDs
    status: str
    placed_on: date
    total: str                  # "$42.10"

class SearchResult(BaseModel):
    orders: list[Order]
    total_matches: int
    next_cursor: str | None = None
    note: str | None = None     # guidance for the model

@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True, idempotentHint=True))
def search_orders(
    customer_id: Annotated[str, Field(pattern=r"^C\d{6}$",
                                      description="Customer ID like C000123")],
    status: Literal["pending", "shipped", "delivered", "cancelled", "any"] = "any",
    since: Annotated[date | None, Field(description="YYYY-MM-DD; default 90 days ago")] = None,
    limit: Annotated[int, Field(ge=1, le=50)] = 10,
    cursor: str | None = None,
) -> SearchResult:
    """Search a customer's orders by status and date range.

    Use for order status/history questions. Do NOT use for refunds (use create_refund).
    """
    since = since or date.today() - timedelta(days=90)
    rows, total, nxt = repo.search(customer_id, status, since, limit, cursor)
    note = None
    if total == 0:
        note = (f"No {status} orders since {since}. Try status='any' or an earlier "
                "'since' date, or confirm the customer_id with get_customer.")
    elif nxt:
        note = f"Showing {len(rows)} of {total}. Call again with cursor='{nxt}' for more."
    return SearchResult(orders=rows, total_matches=total, next_cursor=nxt, note=note)
```

FastMCP turns the type hints, `Literal` and `Field` constraints into the JSON Schema, and the Pydantic return type into a structured `outputSchema`, so the model sees all of it.

### 5. Designing for failure

Errors are prompts too. The model should be able to **recover on its own** from the error message.

```python
from enum import Enum

class ErrorCode(str, Enum):
    INVALID_ARGUMENT = "INVALID_ARGUMENT"
    NOT_FOUND = "NOT_FOUND"
    PERMISSION_DENIED = "PERMISSION_DENIED"
    RATE_LIMITED = "RATE_LIMITED"
    UPSTREAM_UNAVAILABLE = "UPSTREAM_UNAVAILABLE"

def tool_error(code: ErrorCode, message: str, hint: str, retryable: bool) -> dict:
    return {"error": {"code": code.value, "message": message,
                      "hint": hint, "retryable": retryable}}

# Examples returned to the model:
tool_error(ErrorCode.NOT_FOUND, "Order ORD-1 does not exist.",
           "Use search_orders to find valid order IDs for this customer.", False)
tool_error(ErrorCode.PERMISSION_DENIED, "You cannot refund orders over $500.",
           "Tell the user a supervisor must approve; do not retry.", False)
tool_error(ErrorCode.RATE_LIMITED, "GitHub API rate limit hit.",
           "Wait; the system will retry automatically. Do not call again this turn.", True)
```

In MCP, tool execution errors should come back as a **result with `isError: true`** so the model sees them. Protocol errors (unknown tool, malformed request) are JSON-RPC errors. FastMCP converts exceptions raised in a tool into `isError` results. Raise exceptions with a **helpful message** and keep internals out of it.

| Failure type | What the model should get |
|---|---|
| Validation | Which field, what was received, the expected format, an example |
| Not found | That nothing matched plus how to find valid values (which tool to use) |
| Permission | That access is denied, **do not retry**, and what to tell the user |
| Transient | Retry handled internally with backoff. If still failing, say so with `retryable` and a suggested next step |
| Partial success | What succeeded, what failed, and the IDs of each |
| Timeout / async | A job ID plus how and when to check status |

Never return **raw stack traces** (they waste tokens and leak internals), **empty strings** (they cause loops) or **HTML error pages**.

### 6. Output design

- Return **semantic fields** the model can reason with: `"assignee": "Alice Chen"` and not `"assignee_id": "8b1f..."`. Include IDs too when follow-up calls need them.
- **Truncate** large outputs with an explicit marker (`"... 1,240 more lines truncated; use read_file with offset=200"`).
- Offer a **concise vs detailed** mode to save tokens.
- Keep formats consistent across tools: the same date format and the same error envelope.

### 7. Tool set design

```
Too granular (bad)                    Task-level (good)
------------------                    -----------------
list_users                            find_free_slot(attendees, duration, window)
get_user_calendar   ──── merge ───▶   schedule_meeting(attendees, start, duration, title)
compute_overlap
create_event
send_invite
```

- Aim for a **small set**. Large tool lists degrade selection accuracy. Load tools dynamically per task, or use tool search when there are many.
- Avoid **overlapping** tools. If two tools could both fit, merge them or sharpen the "use when" sections.
- Wrap APIs for the agent's job rather than mirroring every REST endpoint one to one.

### 8. Iterate with evaluation

1. Write realistic tasks, including multi-step ones and failure cases.
2. Run the agent and capture traces: tool choice accuracy, argument errors, number of calls, tokens.
3. Read the failures and fix names, descriptions, schemas and errors. Have an LLM critique and rewrite tool descriptions using the traces.
4. Re-run the eval and keep it as a regression suite.

## Common Pitfalls

- Using the backend function's docstring ("Internal helper, see JIRA-123") as the LLM description.
- Untyped `**kwargs` or a `query: str` parameter that expects a JSON blob.
- Returning raw API payloads with hundreds of fields, which exhaust the context.
- Raising generic exceptions so the model just sees `"Error"`.
- Making non-idempotent write tools that the agent will retry.
- Keeping too many similar tools, or the same name from two MCP servers without namespacing.
- Never testing tools with the actual model. Descriptions that look clear to humans can still confuse the model.

## Likely Follow-up Questions

1. **Should I put examples in tool descriptions?**
   Yes, short ones for non-obvious formats. Some APIs support `input_examples` as well. Keep them brief, since they cost tokens on every call.
2. **How many tools is too many?**
   It depends on the model, but accuracy often drops beyond a few dozen. Group tools by task, load them dynamically, or add a router step.
3. **Exceptions or error objects?**
   The model should see a structured, actionable error in the tool result (MCP `isError: true`). Internally you can raise exceptions and convert them at the boundary.
4. **How do you handle huge outputs like logs?**
   Paginate, filter server-side (grep or a query parameter), summarize, or return a resource link and let the model fetch slices.
5. **What are MCP tool annotations?**
   Hints such as `readOnlyHint`, `destructiveHint`, `idempotentHint` and `openWorldHint` that help hosts decide on confirmations. They are not security controls.

[← Back to questions](../questions.md)
