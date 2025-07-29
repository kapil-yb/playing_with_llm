# Local Text-to-SQL with Ollama and DuckDB

This project sets up a simple "MCP Server" using Python and Flask to answer natural language questions by converting them into SQL queries that run against a local DuckDB database. The language-to-SQL translation is handled by a locally running Ollama instance.

## Prerequisites

Before you begin, ensure you have the following installed on your Mac:

- **Python 3.8+**
- **DuckDB CLI**: Can be installed via Homebrew (`brew install duckdb`).
- **Ollama**: Download and run the Ollama application from [ollama.com](https://ollama.com).

## Setup Instructions

### 1. Set Up Your Project and Environment

First, set up your project folder and a Python virtual environment to keep dependencies isolated.

```bash
# Create and enter the project directory
mkdir mcp-local-duckdb
cd mcp-local-duckdb

# Create a Python virtual environment
python3 -m venv venv

# Activate the virtual environment
source venv/bin/activate
```

Next, create a `requirements.txt` file for the Python packages.

```plaintext
# requirements.txt
flask
duckdb
requests
```

Now, install these packages.

```bash
pip install -r requirements.txt
```

### 2. Prepare Ollama and DuckDB

#### Ollama

Make sure the Ollama application is running. Then, pull the `llama3` model, which will be used for the SQL generation.

```bash
ollama pull llama3
```

#### DuckDB

Create and populate your local database file.

```bash
# This command creates the file and opens the DuckDB CLI
duckdb my_local_db.duckdb
```

Inside the DuckDB CLI, paste the following SQL to create and populate a sample table:

```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary INT,
    hire_date DATE
);

INSERT INTO employees (id, name, department, salary, hire_date) VALUES
(1, 'Alice', 'Engineering', 90000, '2022-08-01'),
(2, 'Bob', 'Sales', 75000, '2023-01-15'),
(3, 'Charlie', 'Engineering', 95000, '2021-05-20'),
(4, 'Diana', 'Marketing', 68000, '2023-03-10');

-- To exit the DuckDB CLI, type .exit and press Enter
.exit
```

You will now have a `my_local_db.duckdb` file in your project folder.

### 3. Create the MCP Server

Create a file named `app.py` in your project folder and add the following code.

```python
# filepath: app.py
import os
import requests
import json
from flask import Flask, request, jsonify
import duckdb

# --- CONFIGURATION ---
app = Flask(__name__)
DB_FILE = "my_local_db.duckdb"
OLLAMA_URL = "http://localhost:11434/api/generate"

# --- HELPER FUNCTIONS ---
def get_db_schema():
    """Connects to DuckDB and extracts the schema of all tables."""
    schema_str = "DuckDB Database Schema:\n"
    try:
        con = duckdb.connect(DB_FILE)
        tables = con.execute("SHOW TABLES;").fetchall()
        for table_tuple in tables:
            table_name = table_tuple[0]
            schema_str += f"- Table '{table_name}' has columns: "
            columns = con.execute(f"DESCRIBE {table_name};").fetchall()
            schema_str += ", ".join([f"{col[0]} ({col[1]})" for col in columns]) + "\n"
        con.close()
        return schema_str
    except Exception as e:
        return f"Error getting schema: {e}"

def execute_sql(sql_query):
    """Executes the given SQL query on DuckDB and returns the result."""
    try:
        con = duckdb.connect(DB_FILE)
        results_df = con.execute(sql_query).fetchdf()
        con.close()
        # Convert pandas DataFrame to a list of dictionaries for JSON compatibility
        json_results = results_df.to_dict(orient='records')
        return json_results, None
    except Exception as e:
        return None, f"Error executing SQL: {e}"

# --- FLASK ROUTE ---
@app.route('/query', methods=['POST'])
def handle_query():
    """Handles the incoming natural language query."""
    user_question = request.json.get('question')
    if not user_question:
        return jsonify({"error": "No question provided."}), 400

    schema = get_db_schema()

    # Construct the prompt for the LLM
    prompt = f"""
    You are an expert DuckDB SQL query writer.
    Based on the following database schema, write a single, valid DuckDB SQL query to answer the user's question.
    Only output the SQL query itself, with no explanation, preamble, or markdown formatting.

    {schema}

    User Question:
    {user_question}

    SQL Query:
    """

    try:
        # Generate the SQL query using the local Ollama LLM
        payload = {
            "model": "llama3",
            "prompt": prompt,
            "stream": False
        }
        response = requests.post(OLLAMA_URL, json=payload)
        response.raise_for_status()
        
        response_data = json.loads(response.text)
        generated_sql = response_data.get('response', '').strip()

        if not generated_sql:
             return jsonify({"error": "Failed to generate SQL from LLM."}), 500

        # Execute the generated SQL
        results, error = execute_sql(generated_sql)
        if error:
            return jsonify({"error": error, "generated_sql": generated_sql}), 500
        
        return jsonify({"data": results, "generated_sql": generated_sql})

    except Exception as e:
        return jsonify({"error": f"An error occurred: {e}"}), 500

# --- MAIN EXECUTION ---
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)
```

### Running the Application

#### Start the Server

Open a terminal, navigate to your project folder, activate the virtual environment, and run the Flask app.

```bash
source venv/bin/activate
flask run --port=5001
```

Leave this terminal window running.

#### Send a Request

Open a second terminal window and use `curl` to send a question to your server.

```bash
curl -X POST http://localhost:5001/query \
-H "Content-Type: application/json" \
-d '{
  "question": "What is the average salary for the Engineering department?"
}'
```

You should receive a JSON response with the answer from your database.

```json
{
  "data": [
    {
      "avg(salary)": 92500.0
    }
  ],
  "generated_sql": "SELECT avg(salary) FROM employees WHERE department = 'Engineering'"
}
```