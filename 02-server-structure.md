# MCP Server Structure and Architecture

## Overview
This document provides a detailed analysis of MCP server architecture, components, and how they work together to create functional MCP servers.

## Basic Server Anatomy

### Minimal Server Structure
The simplest possible MCP server:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('minimal-server')

@mcp.tool()
def hello(name: str) -> str:
    """Say hello to someone."""
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Standard Server Structure
Most production servers follow this structure:

```python
# 1. Imports
from typing import Optional, List, Dict, Any
from dotenv import load_dotenv
from mcp.server.fastmcp import FastMCP
import os

# 2. Configuration
load_dotenv()

# 3. Initialize MCP and external clients
mcp = FastMCP('production-server')
client = ExternalService(api_key=os.getenv("API_KEY"))

# 4. Helper functions (optional)
def validate_input(data: str) -> bool:
    """Validate input data."""
    return len(data) > 0

# 5. Tool definitions
@mcp.tool()
def process_data(input: str) -> str:
    """Process input data."""
    if not validate_input(input):
        return "Invalid input"
    return client.process(input)

# 6. Main execution
if __name__ == "__main__":
    # Optional initialization
    setup_database()
    
    # Run server
    mcp.run(transport="stdio")
```

## Component Breakdown

### 1. FastMCP Instance
The core component that manages the server:

```python
mcp = FastMCP('server-name', port=8080)  # Optional port for SSE
```

**Properties:**
- Handles tool registration via decorators
- Manages communication protocol
- Provides transport layer abstraction
- Handles request/response serialization

### 2. Tool Registry
Tools are automatically registered when decorated:

```python
@mcp.tool()  # Registers function as MCP tool
def my_tool(): 
    pass
```

**Registration process:**
- Decorator captures function metadata
- Extracts type hints for validation
- Parses docstring for descriptions
- Adds to internal tool registry

### 3. Transport Layer
Two transport modes available:

#### stdio (Standard I/O)
```python
mcp.run(transport="stdio")
```
- Communication via stdin/stdout
- Synchronous message passing
- Default for most implementations
- Best for CLI integrations

#### SSE (Server-Sent Events)
```python
mcp.run(transport="sse", port=8080)
```
- HTTP-based communication
- Supports streaming responses
- Requires port specification
- Better for web integrations

### 4. External Service Integration
Common pattern for integrating external services:

```python
# Initialize at module level
client = ServiceClient(
    api_key=os.getenv("SERVICE_API_KEY"),
    endpoint=os.getenv("SERVICE_ENDPOINT", "default_url")
)

@mcp.tool()
def use_service(query: str) -> str:
    """Use external service."""
    return client.query(query)
```

## Advanced Server Structures

### Multi-Module Structure
For complex servers with multiple components:

```
project/
├── server.py           # MCP server definition
├── tools.py            # Tool implementations
├── workflows.py        # Business logic
├── utils.py            # Helper functions
├── config.py           # Configuration management
└── pyproject.toml      # Dependencies
```

**server.py:**
```python
from mcp.server.fastmcp import FastMCP
from tools import create, read, update, delete

mcp = FastMCP('complex-server')

# Register imported functions as tools
mcp.tool()(create)
mcp.tool()(read)
mcp.tool()(update)
mcp.tool()(delete)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**tools.py:**
```python
def create(data: dict) -> str:
    """Create a new record."""
    # Implementation
    return "Created"

def read(id: str) -> dict:
    """Read a record."""
    # Implementation
    return {"id": id, "data": "example"}
```

### Async Server Structure
For servers with async operations:

```python
import asyncio
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('async-server')

@mcp.tool()
async def async_operation(query: str) -> str:
    """Perform async operation."""
    result = await external_async_call(query)
    return result

if __name__ == "__main__":
    # Initialize async resources
    asyncio.run(initialize_async_resources())
    
    # Run server
    mcp.run(transport="stdio")
```

## Server Lifecycle

### 1. Initialization Phase
```python
# Load configuration
load_dotenv()

# Create MCP instance
mcp = FastMCP('server')

# Initialize external resources
database = connect_database()
cache = initialize_cache()
```

### 2. Tool Registration Phase
```python
# Tools are registered when decorators are evaluated
@mcp.tool()
def tool1(): pass

@mcp.tool()
def tool2(): pass
```

### 3. Runtime Phase
```python
if __name__ == "__main__":
    # Pre-runtime setup
    perform_setup()
    
    # Start server (blocking)
    mcp.run(transport="stdio")
    
    # Post-runtime cleanup (rarely reached)
    cleanup_resources()
```

## Communication Flow

### Request Processing
1. Client sends JSON-RPC request
2. Transport layer receives message
3. FastMCP deserializes request
4. Validates tool name and parameters
5. Invokes corresponding Python function
6. Serializes response
7. Transport layer sends response

### Example Request/Response
**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tool/call",
  "params": {
    "name": "search_doc",
    "arguments": {
      "query": "MCP patterns"
    }
  },
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "output": "Found 5 documents about MCP patterns..."
  },
  "id": 1
}
```

## Resource Management

### Connection Pooling
```python
# Global connection pool
connection_pool = create_pool(max_connections=10)

@mcp.tool()
def database_query(sql: str) -> list:
    with connection_pool.get_connection() as conn:
        return conn.execute(sql).fetchall()
```

### Cleanup Patterns
```python
import atexit

def cleanup():
    """Cleanup resources on exit."""
    database.close()
    cache.clear()
    
atexit.register(cleanup)
```

## Server Configuration

### Environment-Based Configuration
```python
import os
from dotenv import load_dotenv

load_dotenv()

CONFIG = {
    "api_key": os.getenv("API_KEY"),
    "endpoint": os.getenv("ENDPOINT", "https://default.api"),
    "timeout": int(os.getenv("TIMEOUT", "30")),
    "debug": os.getenv("DEBUG", "false").lower() == "true"
}
```

### Argument-Based Configuration
```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--transport", choices=["stdio", "sse"], default="stdio")
parser.add_argument("--port", type=int, default=8080)
args = parser.parse_args()

mcp.run(transport=args.transport, port=args.port)
```

## Conclusion
MCP server structure follows a consistent pattern that balances simplicity with flexibility. The FastMCP abstraction handles complex protocol details while allowing developers to focus on tool implementation. Understanding these structural patterns is key to building robust and maintainable MCP servers.