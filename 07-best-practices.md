# MCP Server Best Practices

## Overview
This document compiles best practices for building MCP servers, derived from analyzing 15+ implementations in the AI Engineering Hub. Following these practices ensures robust, maintainable, and user-friendly MCP servers.

## Code Organization

### ✅ DO: Modular Structure
Separate concerns for complex servers:
```
project/
├── server.py          # MCP server definition
├── tools.py           # Tool implementations
├── workflows.py       # Business logic
├── utils.py           # Helper functions
├── config.py          # Configuration
├── .env.example       # Environment template
├── README.md          # Documentation
└── pyproject.toml     # Dependencies
```

### ❌ DON'T: Monolithic Files
Avoid putting everything in one file for complex servers.

### ✅ DO: Clear Naming
```python
# Good: Descriptive, action-oriented names
@mcp.tool()
def search_documents(query: str) -> list:
    pass

@mcp.tool()
def generate_summary(text: str, max_length: int = 100) -> str:
    pass
```

### ❌ DON'T: Vague Names
```python
# Bad: Generic, unclear purpose
@mcp.tool()
def process(data):
    pass

@mcp.tool()
def do_thing(x):
    pass
```

## Documentation

### ✅ DO: Comprehensive Docstrings
```python
@mcp.tool()
def create_report(
    data_source: str,
    report_type: str = "summary",
    include_charts: bool = False
) -> Dict[str, Any]:
    """
    Generate a report from the specified data source.
    
    This tool analyzes data from the source and creates a formatted
    report based on the specified type. Reports can include optional
    visualizations.
    
    Args:
        data_source: Path to data file or database connection string
        report_type: Type of report - "summary", "detailed", or "executive"
        include_charts: Whether to generate visualization charts
    
    Returns:
        Dictionary containing:
        - report: The generated report content
        - metadata: Report generation metadata
        - charts: List of chart URLs (if requested)
    
    Raises:
        FileNotFoundError: If data_source doesn't exist
        ValueError: If report_type is invalid
    
    Example:
        >>> create_report("/data/sales.csv", "summary", True)
        {"report": "...", "metadata": {...}, "charts": [...]}
    """
    pass
```

### ❌ DON'T: Missing Documentation
```python
# Bad: No docstring or minimal description
@mcp.tool()
def create_report(data_source, report_type="summary"):
    # Creates a report
    pass
```

## Error Handling

### ✅ DO: Graceful Error Handling
```python
@mcp.tool()
def safe_operation(input_path: str) -> Dict[str, Any]:
    """Perform operation with comprehensive error handling."""
    # Input validation
    if not input_path:
        return {"error": "Input path is required"}
    
    if not os.path.exists(input_path):
        return {"error": f"File not found: {input_path}"}
    
    try:
        # Main operation
        result = process_file(input_path)
        return {
            "success": True,
            "result": result,
            "processed_at": datetime.now().isoformat()
        }
    except PermissionError:
        return {"error": "Permission denied accessing file"}
    except ValueError as e:
        return {"error": f"Invalid data format: {str(e)}"}
    except Exception as e:
        # Log unexpected errors for debugging
        logger.error(f"Unexpected error: {e}")
        return {"error": "An unexpected error occurred"}
```

### ❌ DON'T: Let Exceptions Bubble Up
```python
# Bad: No error handling
@mcp.tool()
def unsafe_operation(input_path: str) -> str:
    data = open(input_path).read()  # Can throw multiple exceptions
    return process(data)  # Uncaught exceptions crash the server
```

## Type Hints

### ✅ DO: Complete Type Annotations
```python
from typing import List, Dict, Optional, Any

@mcp.tool()
def analyze_data(
    dataset: List[Dict[str, Any]],
    columns: Optional[List[str]] = None,
    threshold: float = 0.5
) -> Dict[str, Union[float, List[str]]]:
    """Analyze dataset with optional column filtering."""
    pass
```

### ❌ DON'T: Missing Types
```python
# Bad: No type hints
@mcp.tool()
def analyze_data(dataset, columns=None, threshold=0.5):
    pass
```

## Environment Configuration

### ✅ DO: Use Environment Variables
```python
from dotenv import load_dotenv
import os

load_dotenv()

# Configuration with defaults
CONFIG = {
    "api_key": os.getenv("API_KEY"),
    "endpoint": os.getenv("API_ENDPOINT", "https://api.default.com"),
    "timeout": int(os.getenv("TIMEOUT", "30")),
    "debug": os.getenv("DEBUG", "false").lower() == "true"
}

# Validate required configuration
if not CONFIG["api_key"]:
    raise ValueError("API_KEY environment variable is required")
```

### ❌ DON'T: Hardcode Secrets
```python
# Bad: Hardcoded credentials
API_KEY = "sk-1234567890abcdef"  # NEVER DO THIS
client = APIClient(api_key=API_KEY)
```

### ✅ DO: Provide .env.example
```env
# .env.example
# Copy this file to .env and fill in your values

# Required
API_KEY=your_api_key_here

# Optional (defaults shown)
API_ENDPOINT=https://api.example.com
TIMEOUT=30
DEBUG=false
```

## Resource Management

### ✅ DO: Proper Cleanup
```python
import atexit

# Global resources
connection_pool = None

def initialize_resources():
    global connection_pool
    connection_pool = create_pool(max_connections=10)

def cleanup_resources():
    """Clean up resources on exit."""
    if connection_pool:
        connection_pool.close()
    # Clean up temp files
    cleanup_temp_directory()

# Register cleanup
atexit.register(cleanup_resources)

@mcp.tool()
def database_operation(query: str) -> list:
    """Execute database query with connection pooling."""
    with connection_pool.get_connection() as conn:
        return conn.execute(query).fetchall()
```

### ❌ DON'T: Resource Leaks
```python
# Bad: No cleanup, connection leak
@mcp.tool()
def bad_database_operation(query: str) -> list:
    conn = create_connection()  # Never closed
    return conn.execute(query).fetchall()
```

## Security

### ✅ DO: Validate and Sanitize Input
```python
import re
from pathlib import Path

@mcp.tool()
def read_file(file_path: str) -> str:
    """Safely read a file with path validation."""
    # Validate path
    try:
        safe_path = Path(file_path).resolve()
        
        # Ensure path is within allowed directory
        allowed_dir = Path("/allowed/directory").resolve()
        if not safe_path.is_relative_to(allowed_dir):
            return "Error: Access denied - path outside allowed directory"
        
        # Check file extension
        if safe_path.suffix not in ['.txt', '.json', '.csv']:
            return "Error: Unsupported file type"
        
        with open(safe_path, 'r') as f:
            return f.read()
            
    except Exception as e:
        return f"Error reading file: {str(e)}"
```

### ❌ DON'T: Trust User Input
```python
# Bad: Direct execution of user input
@mcp.tool()
def dangerous_query(sql: str) -> list:
    # SQL injection vulnerability
    return database.execute(f"SELECT * FROM users WHERE {sql}")
```

## Performance

### ✅ DO: Optimize for Common Cases
```python
# Cache expensive operations
from functools import lru_cache

@lru_cache(maxsize=100)
def expensive_computation(input_data: str) -> str:
    """Cache results of expensive computations."""
    # Complex processing
    return result

@mcp.tool()
def process_with_cache(data: str) -> str:
    """Process data using cached computation."""
    return expensive_computation(data)
```

### ✅ DO: Use Connection Pooling
```python
# Initialize once, reuse connections
pool = ConnectionPool(max_size=10)

@mcp.tool()
def efficient_query(query: str) -> list:
    with pool.get_connection() as conn:
        return conn.execute(query).fetchall()
```

## Testing

### ✅ DO: Write Tests
```python
# test_server.py
import pytest
from server import mcp

def test_tool_success():
    """Test successful tool execution."""
    result = mcp.tools["my_tool"]("valid_input")
    assert result["success"] == True
    assert "result" in result

def test_tool_error_handling():
    """Test tool error handling."""
    result = mcp.tools["my_tool"]("")
    assert "error" in result
    assert result.get("success") == False

def test_tool_with_mock():
    """Test with mocked external service."""
    with patch('server.external_client') as mock_client:
        mock_client.search.return_value = {"data": "mocked"}
        result = mcp.tools["search_tool"]("query")
        assert result == {"data": "mocked"}
```

## Logging

### ✅ DO: Structured Logging
```python
import logging
import sys

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stderr)  # Use stderr for stdio transport
    ]
)

logger = logging.getLogger(__name__)

@mcp.tool()
def logged_operation(input: str) -> str:
    """Operation with structured logging."""
    logger.info(f"Starting operation with input: {input[:50]}...")
    
    try:
        result = process(input)
        logger.info("Operation completed successfully")
        return result
    except Exception as e:
        logger.error(f"Operation failed: {e}", exc_info=True)
        return f"Error: {str(e)}"
```

## Deployment

### ✅ DO: Provide Clear Setup Instructions
```markdown
# README.md

## Installation

1. Clone the repository
2. Install dependencies: `uv pip install -e .`
3. Copy `.env.example` to `.env` and configure
4. Run the server: `uv run server.py`

## Configuration

Set the following environment variables:
- `API_KEY`: Your API key (required)
- `DEBUG`: Enable debug mode (optional, default: false)
```

### ✅ DO: Version Dependencies
```toml
[project]
dependencies = [
    "mcp[cli]>=1.10.0,<2.0.0",  # Pin major version
    "requests>=2.28.0,<3.0.0",
]
```

## Async Operations

### ✅ DO: Handle Async Properly
```python
import asyncio

@mcp.tool()
async def async_tool(query: str) -> str:
    """Async tool with proper error handling."""
    try:
        result = await async_operation(query)
        return result
    except asyncio.TimeoutError:
        return "Error: Operation timed out"
    except Exception as e:
        return f"Error: {str(e)}"

# Initialize async resources properly
if __name__ == "__main__":
    # Run async initialization
    asyncio.run(initialize_async_resources())
    
    # Start server
    mcp.run(transport="stdio")
```

## Common Pitfalls to Avoid

1. **Don't use print() with stdio transport** - Use stderr or logging
2. **Don't forget to close resources** - Use context managers or cleanup handlers
3. **Don't expose sensitive data in errors** - Sanitize error messages
4. **Don't block the event loop** - Use async for I/O operations
5. **Don't ignore type hints** - They help with validation and documentation
6. **Don't skip error handling** - Always handle expected failures gracefully
7. **Don't hardcode configuration** - Use environment variables
8. **Don't trust user input** - Always validate and sanitize

## Checklist for Production-Ready MCP Server

- [ ] Comprehensive docstrings for all tools
- [ ] Type hints for all parameters and returns
- [ ] Error handling for all operations
- [ ] Environment-based configuration
- [ ] Resource cleanup mechanisms
- [ ] Input validation and sanitization
- [ ] Logging configuration (stderr for stdio)
- [ ] Tests for core functionality
- [ ] Clear README with setup instructions
- [ ] .env.example file
- [ ] Proper dependency versioning
- [ ] Security considerations addressed
- [ ] Performance optimizations (caching, pooling)

## Conclusion
Following these best practices ensures your MCP server is robust, secure, and maintainable. The patterns identified from real implementations provide a solid foundation for building production-ready MCP servers that are easy to use and integrate.