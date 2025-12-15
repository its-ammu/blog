---
title: "Build Lightweight MCP Servers Easily with FastMCP"
date: 2025-07-02T16:33:12+05:30
draft: false
tags:
    - agents
    - ai
    - mcp
---

Hi there!

In this post, I'll show you how to create MCP servers easily with FastMCP. To know the basics of MCP, you can read my previous post [here](/posts/what_is_mcp).

## What is FastMCP?

FastMCP is a new open-source library that makes it easy to create MCP servers. It's built on top of the FastAPI framework and uses the latest version of the MCP protocol.

## How to install FastMCP?

```bash
pip install fastmcp
```

## How to create an MCP server with FastMCP?

Creating an MCP server with FastMCP is straightforward. Let's build a simple example that exposes a file system tool and a resource.

### Basic Setup

First, create a new Python file for your MCP server:

```python
from fastmcp import FastMCP

# Create a new MCP server instance
mcp = FastMCP("My MCP Server")
```

That's it! You've created an MCP server. Now let's add some functionality.

### Adding Tools

Tools are functions that the MCP client (like an AI agent) can call. Let's add a tool to read files:

```python
from fastmcp import FastMCP
import os

mcp = FastMCP("File System Server")

@mcp.tool()
def read_file(file_path: str) -> str:
    """Read the contents of a file from the local filesystem.
    
    Args:
        file_path: The path to the file to read
        
    Returns:
        The contents of the file as a string
    """
    if not os.path.exists(file_path):
        return f"Error: File '{file_path}' not found"
    
    with open(file_path, 'r', encoding='utf-8') as f:
        return f.read()
```

### Adding Resources

Resources are data sources that agents can access. Let's add a resource that lists files in a directory:

```python
@mcp.resource("file://directory/{directory}")
def list_directory(directory: str) -> str:
    """List all files in the specified directory.
    
    Args:
        directory: The directory path to list
        
    Returns:
        A formatted string listing all files
    """
    if not os.path.isdir(directory):
        return f"Error: '{directory}' is not a valid directory"
    
    files = os.listdir(directory)
    return "\n".join(f"- {f}" for f in files)
```

### Adding Prompts

Prompts are reusable templates that help guide the agent's behavior:

```python
@mcp.prompt()
def analyze_codebase() -> str:
    """Prompt template for analyzing a codebase.
    
    Returns:
        A prompt string for code analysis
    """
    return """Analyze the codebase structure:
1. Identify the main entry points
2. List all dependencies
3. Find potential security issues
4. Suggest improvements"""
```

### Complete Example

Here's a complete MCP server that combines all these features:

```python
from fastmcp import FastMCP
import os
import json
from datetime import datetime

mcp = FastMCP("Development Tools Server")

@mcp.tool()
def read_file(file_path: str) -> str:
    """Read the contents of a file from the local filesystem."""
    if not os.path.exists(file_path):
        return f"Error: File '{file_path}' not found"
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            return f.read()
    except Exception as e:
        return f"Error reading file: {str(e)}"

@mcp.tool()
def write_file(file_path: str, content: str) -> str:
    """Write content to a file on the local filesystem.
    
    Args:
        file_path: The path where to write the file
        content: The content to write
    """
    try:
        os.makedirs(os.path.dirname(file_path), exist_ok=True)
        with open(file_path, 'w', encoding='utf-8') as f:
            f.write(content)
        return f"Successfully wrote to '{file_path}'"
    except Exception as e:
        return f"Error writing file: {str(e)}"

@mcp.tool()
def get_file_info(file_path: str) -> str:
    """Get metadata about a file.
    
    Args:
        file_path: The path to the file
        
    Returns:
        JSON string with file information
    """
    if not os.path.exists(file_path):
        return json.dumps({"error": "File not found"})
    
    stat = os.stat(file_path)
    info = {
        "path": file_path,
        "size": stat.st_size,
        "modified": datetime.fromtimestamp(stat.st_mtime).isoformat(),
        "is_file": os.path.isfile(file_path),
        "is_dir": os.path.isdir(file_path)
    }
    return json.dumps(info, indent=2)

@mcp.resource("file://directory/{directory}")
def list_directory(directory: str) -> str:
    """List all files and subdirectories in the specified directory."""
    if not os.path.isdir(directory):
        return f"Error: '{directory}' is not a valid directory"
    
    items = []
    for item in os.listdir(directory):
        item_path = os.path.join(directory, item)
        item_type = "directory" if os.path.isdir(item_path) else "file"
        items.append(f"{item_type}: {item}")
    
    return "\n".join(items) if items else "Directory is empty"

@mcp.prompt()
def code_review_prompt() -> str:
    """Generate a prompt for code review tasks."""
    return """Review the provided code and check for:
1. Code quality and best practices
2. Potential bugs or errors
3. Security vulnerabilities
4. Performance optimizations
5. Documentation completeness

Provide specific, actionable feedback."""
```

### Running Your Server

To run your MCP server, add this at the end of your file:

```python
if __name__ == "__main__":
    mcp.run()
```

Then run it with:

```bash
python your_server.py
```

Or use the FastMCP CLI:

```bash
fastmcp serve your_server.py
```

### Connecting to an Agent

Once your server is running, you can connect it to MCP-compatible agents like:

- **Claude Desktop**: Update claude settings and add your server to the `claude_desktop_config.json` file
- **Amazon Q**: Configure it through Q CLI
- **Cursor IDE**: Add it to your cursor IDE MCP settings

### Best Practices

1. **Error Handling**: Always wrap file operations in try-except blocks and return clear error messages
2. **Type Hints**: Use Python type hints in your function signatures for better documentation
3. **Docstrings**: Write clear docstrings that explain what each tool/resource does
4. **Security**: Be careful about file paths - validate and sanitize inputs to prevent directory traversal attacks
5. **Resource Limits**: Consider adding limits on file sizes or operation timeouts for production use

### Advanced Features

FastMCP also supports:

- **Async tools**: Use `async def` for non-blocking operations
- **Streaming responses**: For large data transfers
- **Custom schemas**: Define complex input/output types
- **Middleware**: Add authentication, logging, or rate limiting

### Example: Database MCP Server

Here's a more advanced example that connects to a database:

```python
from fastmcp import FastMCP
import sqlite3
from typing import List, Dict

mcp = FastMCP("Database Server")

@mcp.tool()
def query_database(db_path: str, query: str) -> str:
    """Execute a SQL query on a SQLite database.
    
    Args:
        db_path: Path to the SQLite database file
        query: SQL query to execute
        
    Returns:
        JSON string with query results
    """
    try:
        conn = sqlite3.connect(db_path)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()
        cursor.execute(query)
        
        rows = cursor.fetchall()
        results = [dict(row) for row in rows]
        
        conn.close()
        return json.dumps(results, indent=2)
    except Exception as e:
        return json.dumps({"error": str(e)})

@mcp.resource("db://tables/{db_path}")
def list_tables(db_path: str) -> str:
    """List all tables in a SQLite database."""
    try:
        conn = sqlite3.connect(db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
        tables = [row[0] for row in cursor.fetchall()]
        conn.close()
        return "\n".join(f"- {table}" for table in tables)
    except Exception as e:
        return f"Error: {str(e)}"
```

---

## Wrapping Up

FastMCP makes it incredibly easy to build MCP servers. With just a few decorators, you can expose tools, resources, and prompts that AI agents can use to interact with your systems.

The key advantages of FastMCP:

- **Simple API**: Decorator-based design that's intuitive to use
- **FastAPI Foundation**: Built on FastAPI, so you get async support and great performance
- **Type Safety**: Leverages Python type hints for better validation
- **Active Development**: Regularly updated with the latest MCP protocol features

### Next Steps

- Build your own MCP server for a specific use case (database, API, file system, etc.)
- Test it with Claude Desktop or another MCP client
- Share your server with the community or use it in your own agent workflows

Thanks for reading! 🎉

If you build something cool with FastMCP, I'd love to hear about it. Happy building!

