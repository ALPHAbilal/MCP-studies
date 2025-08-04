# MCP Tool Definitions Guide

## Overview
Tools are the core functionality of MCP servers. This document explains how to define, implement, and document MCP tools effectively based on patterns observed across implementations.

## Basic Tool Definition

### Minimal Tool
```python
@mcp.tool()
def simple_tool(input: str) -> str:
    """Process input and return result."""
    return f"Processed: {input}"
```

### Complete Tool Definition
```python
@mcp.tool()
def complete_tool(
    required_param: str,
    optional_param: int = 10,
    flag: bool = False
) -> Dict[str, Any]:
    """
    Brief one-line description of the tool.
    
    Detailed explanation of what this tool does, when to use it,
    and any important context the user should know.
    
    Args:
        required_param: Description of the required parameter
        optional_param: Description with default value mentioned
        flag: Boolean flag to control behavior
    
    Returns:
        Dict containing:
        - status: Success or failure indicator
        - data: The processed result
        - metadata: Additional information
    
    Raises:
        ValueError: If required_param is empty
        RuntimeError: If processing fails
    
    Examples:
        >>> complete_tool("test", optional_param=20)
        {"status": "success", "data": "...", "metadata": {...}}
    """
    if not required_param:
        raise ValueError("required_param cannot be empty")
    
    try:
        # Process the input
        result = process_data(required_param, optional_param)
        
        return {
            "status": "success",
            "data": result,
            "metadata": {
                "processed_at": datetime.now().isoformat(),
                "flag_used": flag
            }
        }
    except Exception as e:
        return {
            "status": "error",
            "error": str(e)
        }
```

## Type Hints and Parameters

### Supported Parameter Types
```python
@mcp.tool()
def type_examples(
    # Basic types
    text: str,
    number: int,
    decimal: float,
    flag: bool,
    
    # Container types
    items: List[str],
    mapping: Dict[str, Any],
    
    # Optional types
    optional_text: Optional[str] = None,
    
    # Default values
    with_default: str = "default_value",
    
    # Variable arguments (rarely used)
    **kwargs: Any
) -> Any:
    """Demonstrate various parameter types."""
    pass
```

### Return Types
```python
# String return
@mcp.tool()
def return_string() -> str:
    return "text result"

# Dictionary return
@mcp.tool()
def return_dict() -> Dict[str, Any]:
    return {"key": "value", "count": 42}

# List return
@mcp.tool()
def return_list() -> List[Dict[str, str]]:
    return [{"item": "1"}, {"item": "2"}]

# Boolean return
@mcp.tool()
def return_bool() -> bool:
    return True

# Complex nested return
@mcp.tool()
def return_complex() -> Dict[str, List[Dict[str, Any]]]:
    return {
        "results": [
            {"id": 1, "data": "example"},
            {"id": 2, "data": "another"}
        ]
    }
```

## Documentation Standards

### Docstring Structure
Every tool should have a comprehensive docstring:

```python
@mcp.tool()
def well_documented_tool(param: str) -> str:
    """
    Brief description (one line, <80 chars).
    
    [Optional] Extended description providing more context,
    use cases, and important details. Can be multiple
    paragraphs if needed.
    
    Args:
        param: Clear description of what this parameter
            represents and any constraints or formats
            expected (e.g., "ISO date string", "file path")
    
    Returns:
        Description of what is returned, including format
        and structure if complex.
    
    Raises:
        ExceptionType: When and why this exception occurs
    
    Examples:
        >>> well_documented_tool("input")
        "Expected output"
    
    Note:
        Any additional notes, warnings, or important
        information for users.
    """
    pass
```

### Real-World Documentation Examples

#### From KitOps MCP:
```python
@mcp.tool()
def create_kitfile(working_dir: str, modelkit_name: str) -> str:
    """Create a new Kitfile at the specified directory with the given ModelKit name.

    Use this tool when you need to initialize a Kitfile for a new project or update an existing one with
    updated project code, model, dataset, or documentation. The tool will create or overwrite the Kitfile in the specified
    working directory.

    Args:
        working_dir: The absolute or relative path to the directory where the Kitfile should be created.
                    Must be an existing directory.
        modelkit_name: The name to use for the ModelKit reference in the Kitfile.
                      Cannot be empty.

    Returns:
        A message confirming the Kitfile is created or updated successfully.

    Raises:
        ValueError: If the working directory doesn't exist or modelkit_name is empty.
        RuntimeError: If the Kitfile creation fails for any reason.
    """
```

#### From SDV MCP:
```python
@mcp.tool()
def sdv_visualize(
    folder_name: str,
    table_name: str,
    column_name: str,
) -> str:
    """Generate visualization comparing real and synthetic data for a specific column.

    This tool creates a visual comparison between the real data in the specified folder
    and the synthetic data in the 'synthetic_data' folder for a particular table column.
    The visualization is saved as a PNG file in the 'evaluation_plots' folder.

    Args:
        folder_name (str): Path to folder containing the original CSV data files and metadata.json
        table_name (str): Name of the table to visualize (must exist in the metadata)
        column_name (str): Name of the column to visualize within the specified table

    Returns:
        str: Success message with the path to the saved visualization or error message
    """
```

## Error Handling Patterns

### Basic Error Handling
```python
@mcp.tool()
def safe_tool(input: str) -> str:
    """Tool with basic error handling."""
    try:
        result = risky_operation(input)
        return f"Success: {result}"
    except Exception as e:
        return f"Error: {str(e)}"
```

### Detailed Error Handling
```python
@mcp.tool()
def robust_tool(input: str) -> Dict[str, Any]:
    """Tool with comprehensive error handling."""
    try:
        # Validation
        if not input:
            return {"error": "Input cannot be empty"}
        
        if len(input) > 1000:
            return {"error": "Input too long (max 1000 chars)"}
        
        # Processing
        result = process(input)
        
        return {
            "success": True,
            "result": result
        }
        
    except ValueError as e:
        return {"error": f"Invalid input: {str(e)}"}
    except ConnectionError as e:
        return {"error": f"Connection failed: {str(e)}"}
    except Exception as e:
        return {"error": f"Unexpected error: {str(e)}"}
```

## Async Tools

### Async Tool Definition
```python
@mcp.tool()
async def async_tool(query: str) -> str:
    """Perform async operation."""
    result = await async_external_call(query)
    return result
```

### Async with Sync Wrapper
```python
import asyncio

@mcp.tool()
def sync_wrapper(query: str) -> str:
    """Sync wrapper for async operation."""
    return asyncio.run(async_operation(query))

async def async_operation(query: str) -> str:
    """Internal async operation."""
    # Async processing
    return result
```

## Tool Naming Conventions

### Good Tool Names
```python
# Action-oriented, clear purpose
@mcp.tool()
def search_documents(query: str) -> list: pass

@mcp.tool()
def create_record(data: dict) -> str: pass

@mcp.tool()
def validate_input(text: str) -> bool: pass

@mcp.tool()
def generate_report(params: dict) -> str: pass
```

### Avoid These Names
```python
# Too generic
def process(): pass
def handle(): pass
def do_thing(): pass

# Not descriptive
def func1(): pass
def tool_a(): pass

# Too long
def perform_complex_multi_step_operation_with_validation(): pass
```

## Advanced Tool Patterns

### Tools with Side Effects
```python
@mcp.tool()
def ingest_documents(directory: str) -> str:
    """
    Loads data from a directory into the index.
    
    Note: This operation modifies the global index state.
    Wait for completion before querying.
    
    Args:
        directory: Path to directory containing documents
    
    Returns:
        Success message with document count
    """
    try:
        clear_index()  # Side effect 1
        count = load_documents(directory)  # Side effect 2
        rebuild_index()  # Side effect 3
        return f"Loaded {count} documents successfully"
    except Exception as e:
        return f"Failed to load documents: {str(e)}"
```

### Tools with External Dependencies
```python
# Initialize dependency at module level
client = ExternalAPIClient(api_key=os.getenv("API_KEY"))

@mcp.tool()
def use_external_api(query: str) -> dict:
    """
    Query external API service.
    
    Requires API_KEY environment variable to be set.
    
    Args:
        query: Search query for the API
    
    Returns:
        API response as dictionary
    """
    if not client.is_authenticated():
        return {"error": "API authentication failed"}
    
    return client.search(query)
```

### Composite Tools
```python
@mcp.tool()
def workflow_tool(input_file: str, output_format: str = "json") -> str:
    """
    Execute multi-step workflow.
    
    This tool:
    1. Reads input file
    2. Validates content
    3. Processes data
    4. Formats output
    5. Saves result
    
    Args:
        input_file: Path to input file
        output_format: Output format (json, csv, xml)
    
    Returns:
        Path to output file
    """
    # Step 1: Read
    data = read_file(input_file)
    
    # Step 2: Validate
    if not validate_data(data):
        return "Validation failed"
    
    # Step 3: Process
    processed = transform_data(data)
    
    # Step 4: Format
    formatted = format_output(processed, output_format)
    
    # Step 5: Save
    output_path = save_output(formatted)
    
    return f"Workflow complete: {output_path}"
```

## Testing Tools

### Unit Testing Pattern
```python
# test_tools.py
import pytest
from server import mcp

def test_simple_tool():
    """Test simple tool functionality."""
    result = mcp.tools["simple_tool"]("test input")
    assert result == "Expected output"

def test_error_handling():
    """Test tool error handling."""
    result = mcp.tools["safe_tool"]("")
    assert "Error" in result

def test_async_tool():
    """Test async tool execution."""
    import asyncio
    result = asyncio.run(mcp.tools["async_tool"]("query"))
    assert result is not None
```

## Conclusion
Well-defined tools are the foundation of effective MCP servers. Following these patterns ensures:
- Clear, understandable tool interfaces
- Robust error handling
- Comprehensive documentation
- Consistent behavior across tools
- Easy maintenance and testing