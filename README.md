## Project Overview

This repository contains two related pieces of functionality:

- Airline AI Assistant (Gradio + OpenAI): a simple assistant that can answer questions and call "tools" to look up ticket prices and flight availability.
- Local Text-to-SQL API (Flask + DuckDB + Ollama): a REST API that translates natural language to SQL for a local DuckDB database.

See `API.md` for comprehensive documentation of all public APIs, functions, and components, including examples.

### Repository Contents
- `Multiple Tools calls LLM.ipynb`: Notebook implementing the Airline AI Assistant with tool calling and a Gradio UI.
- `mcp-local-duckdb.md`: Setup guide and reference implementation for the Text-to-SQL API.
- `LICENSE`: License information.

## Quickstart: Airline AI Assistant (Gradio)

1) Prerequisites
- Python 3.9+
- An OpenAI API key with access to the Chat Completions API

2) Install dependencies
```bash
pip install -U openai gradio python-dotenv
```

3) Configure environment
Create a `.env` file in the repository root:
```bash
OPENAI_API_KEY=sk-...your-key...
```

4) Run the notebook
- Open `Multiple Tools calls LLM.ipynb` in Jupyter or VS Code and run all cells.
- The final cell launches a Gradio UI and prints a local URL.

Tip: The assistant can call two tools: `get_ticket_price(destination_city)` and `get_available_flight(destination_city)`.

## Quickstart: Local Text-to-SQL API (DuckDB + Ollama)

Follow the step-by-step guide in `mcp-local-duckdb.md`. It walks through:
- Installing prerequisites
- Creating a DuckDB database
- Running a Flask app that exposes `POST /query` for natural language questions
- Example `curl` requests and responses

## Full API and Function Reference

For signatures, parameters, return types, HTTP endpoints, and runnable code examples, see:
- `API.md`

## License
This project is licensed under the terms specified in `LICENSE`.