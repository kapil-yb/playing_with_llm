# API and Function Reference

This document provides comprehensive documentation for the public APIs, functions, and components in this repository. It covers:
- Airline AI Assistant (Gradio + OpenAI)
- Local Text-to-SQL API (Flask + DuckDB + Ollama)

---

## Airline AI Assistant

Source: `Multiple Tools calls LLM.ipynb`

### Overview
A Gradio-based chat UI backed by OpenAI's Chat Completions API. The assistant can call two tools to retrieve ticket prices and flight availability.

### Environment Variables
- `OPENAI_API_KEY` (required): OpenAI key with Chat Completions access.
- `MODEL` (in-notebook constant): Defaults to `gpt-4o-mini`.

### Public Functions

#### `chat(message, history)`
- **Description**: Main chat function that manages conversation flow and tool usage.
- **Parameters**:
  - `message` (str): The latest user message.
  - `history` (list[dict]): Conversation history in `{role, content}` format.
- **Returns**: `str` final assistant message.
- **Behavior**:
  - Calls OpenAI Chat Completions.
  - If the model requests tool calls, delegates to `handle_tool_call`, then performs a follow-up model call incorporating tool responses.

Example (programmatic):
```python
from openai import OpenAI

# assumes env var OPENAI_API_KEY is set
openai = OpenAI()
MODEL = "gpt-4o-mini"
system_message = "You are a helpful assistant for an Airline called FlightAI. Give short, courteous answers."

history = []
user_message = "Do you have flights to Pune and how much is a ticket to Tokyo?"

# Equivalent of the notebook's chat() behavior
messages = ([{"role": "system", "content": system_message}] +
            history +
            [{"role": "user", "content": user_message}])

# tools defined below
response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)
```

#### `handle_tool_call(message)`
- **Description**: Executes any tool calls requested by the model and formats tool responses.
- **Parameters**:
  - `message` (OpenAI message object): Assistant message containing `tool_calls`.
- **Returns**: `list[dict]` tool response messages, each including `role="tool"`, `tool_call_id`, `name`, and JSON `content`.

### Tools (Functions callable by the model)

#### `get_ticket_price(destination_city)`
- **Description**: Returns a price string for a round-trip ticket.
- **Parameters**: `destination_city` (str)
- **Returns**: `str` price or `"Unknown"` if not found.
- **Example**:
```python
get_ticket_price("Berlin")  # -> "$499" (example); prints tool call log
```

JSON schema advertised to the model:
```python
price_function = {
  "name": "get_ticket_price",
  "description": "Get the price of a return ticket to the destination city.",
  "parameters": {
    "type": "object",
    "properties": {"destination_city": {"type": "string"}},
    "required": ["destination_city"],
    "additionalProperties": False
  }
}
```

#### `get_available_flight(destination_city)`
- **Description**: Returns availability status.
- **Parameters**: `destination_city` (str)
- **Returns**: `str` one of `"Yes"`, `"No"`, or `"Unknown"`.
- **Example**:
```python
get_available_flight("Pune")  # -> "Yes"; prints tool call log
```

JSON schema advertised to the model:
```python
available_function = {
  "name": "get_available_flight",
  "description": "Get the availability of flight to the destination city.",
  "parameters": {
    "type": "object",
    "properties": {"destination_city": {"type": "string"}},
    "required": ["destination_city"],
    "additionalProperties": False
  }
}
```

### Gradio UI
Launch with:
```python
gr.ChatInterface(fn=chat, type="messages").launch()
```
This starts a local web UI where users can converse with the assistant.

---

## Local Text-to-SQL API

Reference and setup: `mcp-local-duckdb.md`

### Overview
A Flask application exposes a REST endpoint that accepts a natural language question, has a local Ollama model translate it into SQL (for DuckDB), executes the SQL, and returns JSON results.

### Environment and Configuration
- DuckDB database file: `my_local_db.duckdb` (path configurable in code)
- Ollama endpoint: `http://localhost:11434/api/generate`
- Flask app default port: `5001`

### HTTP API

#### POST `/query`
- **Description**: Translate a natural language question into SQL using Ollama, execute in DuckDB, return results.
- **Request**:
  - Headers: `Content-Type: application/json`
  - Body:
    ```json
    { "question": "What is the average salary for the Engineering department?" }
    ```
- **Responses**:
  - 200 OK
    ```json
    {
      "data": [ { "avg(salary)": 92500.0 } ],
      "generated_sql": "SELECT avg(salary) FROM employees WHERE department = 'Engineering'"
    }
    ```
  - 400 Bad Request
    ```json
    { "error": "No question provided." }
    ```
  - 500 Internal Server Error
    ```json
    { "error": "An error occurred: ..." }
    ```

### Server-side Functions

#### `get_db_schema()`
- **Description**: Connects to DuckDB and builds a text description of tables and columns.
- **Returns**: `str` schema description; on error returns string with error message.

#### `execute_sql(sql_query)`
- **Description**: Executes a SQL query against DuckDB and returns JSON-serializable results.
- **Parameters**: `sql_query` (str)
- **Returns**: `(list[dict] | None, str | None)` tuple of results and an error string if any.

### End-to-end Example
```bash
# Start Flask app (after following mcp-local-duckdb.md)
flask run --port=5001

# Ask a question
curl -X POST http://localhost:5001/query \
  -H "Content-Type: application/json" \
  -d '{"question": "Show all employees hired after 2022-01-01"}'
```

### Error Handling
- Input validation for missing `question`.
- SQL execution errors returned with `generated_sql` for debugging.

---

## Versioning and Stability
- Notebook utilities and schemas are intended for demonstration purposes and may change.
- The Text-to-SQL API surface is stable at `POST /query` as documented above.

## License
See `LICENSE`.