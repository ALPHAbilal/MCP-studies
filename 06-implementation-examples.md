# MCP Implementation Examples

## Overview
This document provides detailed analysis of real MCP server implementations from the AI Engineering Hub, showcasing different patterns and use cases.

## 1. Web Search Integration (agent-with-mcp-memory)

### Purpose
Provides web search capabilities using Linkup SDK with SSE transport.

### Implementation
```python
from dotenv import load_dotenv
from linkup import LinkupClient
from mcp.server.fastmcp import FastMCP

load_dotenv()

mcp = FastMCP('linkup-server', port=8080)
client = LinkupClient()

@mcp.tool()
def web_search(query: str = "") -> str:
    """Search the web for the given query."""
    search_response = client.search(
        query=query,
        depth="standard",  # "standard" or "deep"
        output_type="sourcedAnswer",  # Multiple output types
        structured_output_schema=None,
    )
    return str(search_response)

if __name__ == "__main__":
   mcp.run(transport="sse")
```

### Key Features
- External API integration
- Configurable search depth
- Multiple output formats
- SSE transport for web access

## 2. RAG with Document Processing (cursor_linkup_mcp)

### Purpose
Combines web search with RAG workflow for document Q&A.

### Implementation
```python
import asyncio
from dotenv import load_dotenv
from linkup import LinkupClient
from rag import RAGWorkflow
from mcp.server.fastmcp import FastMCP

load_dotenv()

mcp = FastMCP('linkup-server')
client = LinkupClient()
rag_workflow = RAGWorkflow()

@mcp.tool()
def web_search(query: str) -> str:
    """Search the web for the given query."""
    search_response = client.search(
        query=query,
        depth="standard",
        output_type="sourcedAnswer",
        structured_output_schema=None,
    )
    return search_response

@mcp.tool()
async def rag(query: str) -> str:
    """Use a simple RAG workflow to answer queries using documents."""
    response = await rag_workflow.query(query)
    return str(response)

if __name__ == "__main__":
    asyncio.run(rag_workflow.ingest_documents("data"))
    mcp.run(transport="stdio")
```

### Key Features
- Async tool support
- Document ingestion on startup
- Multiple tool types in one server
- Combines external API with local processing

## 3. Document Knowledge Base (eyelevel-mcp-rag)

### Purpose
Manages document ingestion and retrieval using GroundX API.

### Implementation
```python
import os
from dotenv import load_dotenv
from groundx import GroundX, Document
from mcp.server.fastmcp import FastMCP

load_dotenv()

mcp = FastMCP("eyelevel-rag")
client = GroundX(api_key=os.getenv("GROUNDX_API_KEY"))

@mcp.tool()
def search_doc_for_rag_context(query: str) -> str:
    """
    Searches and retrieves relevant context from a knowledge base,
    based on the user's query.
    Args:
        query: The search query supplied by the user.
    Returns:
        str: Relevant text content that can be used by the LLM.
    """
    response = client.search.content(
        id=17221,
        query=query,
        n=10,
    )
    return response.search.text

@mcp.tool()
def ingest_documents(local_file_path: str) -> str:
    """
    Ingest documents from a local file into the knowledge base.
    Args:
        local_file_path: Path to the local file to ingest.
    Returns:
        str: A message indicating the documents have been ingested.
    """
    file_name = os.path.basename(local_file_path)
    client.ingest(
        documents=[
            Document(
                bucket_id=17279,
                file_name=file_name,
                file_path=local_file_path,
                file_type="pdf",
                search_data=dict(key="value"),
            )
        ]
    )
    return f"Ingested {file_name} into the knowledge base."

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Key Features
- Document ingestion capabilities
- Knowledge base management
- Search with relevance ranking
- File type handling

## 4. ModelKit Management (kitops-mcp)

### Purpose
Manages ML model packages with comprehensive CRUD operations.

### Implementation
```python
from typing import List, Dict, Any, Optional
from mcp.server.fastmcp import FastMCP
import tools

mcp = FastMCP("kitops_mcp")

@mcp.tool()
def create_kitfile(working_dir: str, modelkit_name: str) -> str:
    """Create a new Kitfile at the specified directory.
    
    Use this tool when you need to initialize a Kitfile for a new project.
    
    Args:
        working_dir: Directory where Kitfile should be created.
        modelkit_name: Name for the ModelKit reference.
    
    Returns:
        A message confirming the Kitfile is created.
    
    Raises:
        ValueError: If directory doesn't exist or name is empty.
        RuntimeError: If creation fails.
    """
    return tools.create(working_dir, modelkit_name)

@mcp.tool()
def inspect_modelkit(modelkit_tag: str, **kwargs) -> Dict[str, Any]:
    """Inspect a ModelKit and return its detailed information."""
    return tools.inspect(modelkit_tag, **kwargs)

@mcp.tool()
def push_and_pack_modelkit(modelkit_tag: str, working_dir: str) -> str:
    """Pack and push a ModelKit to the remote registry."""
    return tools.push_and_pack(modelkit_tag, working_dir)

@mcp.tool()
def pull_and_unpack_modelkit(
    working_dir: str, 
    modelkit_tag: str, 
    filters: Optional[List[str]] = None
) -> str:
    """Pull a ModelKit from registry and unpack it."""
    return tools.pull_and_unpack(working_dir, modelkit_tag, filters)

@mcp.tool()
def remove_modelkit(modelkit_tag: str) -> str:
    """Remove a ModelKit from the remote registry."""
    return tools.remove(modelkit_tag)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Key Features
- Complete CRUD operations
- Modular tool organization
- Complex parameter handling
- Registry management

## 5. Database Operations (llamaindex-mcp)

### Purpose
SQLite database management with SQL query execution.

### Implementation
```python
import sqlite3
import argparse
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('sqlite-demo')

def init_db():
    conn = sqlite3.connect('demo.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS people (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            age INTEGER NOT NULL,
            profession TEXT NOT NULL
        )
    ''')
    conn.commit()
    return conn, cursor

@mcp.tool()
def add_data(query: str) -> bool:
    """Add new data to the people table using SQL INSERT.
    
    Args:
        query: SQL INSERT query
    
    Example:
        >>> query = '''
        ... INSERT INTO people (name, age, profession)
        ... VALUES ('Alice Smith', 25, 'Developer')
        ... '''
        >>> add_data(query)
        True
    """
    conn, cursor = init_db()
    try:
        cursor.execute(query)
        conn.commit()
        return True
    except sqlite3.Error as e:
        print(f"Error adding data: {e}")
        return False
    finally:
        conn.close()

@mcp.tool()
def read_data(query: str = "SELECT * FROM people") -> list:
    """Read data from the people table using SQL SELECT."""
    conn, cursor = init_db()
    try:
        cursor.execute(query)
        return cursor.fetchall()
    except sqlite3.Error as e:
        print(f"Error reading data: {e}")
        return []
    finally:
        conn.close()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--server_type", 
        type=str, 
        default="sse", 
        choices=["sse", "stdio"]
    )
    args = parser.parse_args()
    mcp.run(args.server_type)
```

### Key Features
- Database initialization
- SQL query execution
- Error handling with cleanup
- Configurable transport mode

## 6. Video Processing RAG (mcp-video-rag)

### Purpose
Video content retrieval and chunk extraction.

### Implementation
```python
from mcp.server.fastmcp import FastMCP
from main import clear_index, ingest_data, retrieve_data, chunk_video

mcp = FastMCP("ragie")

@mcp.tool()
def ingest_data_tool(directory: str) -> None:
    """Loads data from a directory into the Ragie index."""
    try:
        clear_index()
        ingest_data(directory)
        return "Data loaded successfully"   
    except Exception as e:
        return f"Failed to load data: {str(e)}"

@mcp.tool()
def retrieve_data_tool(query: str) -> list[dict]:
    """Retrieves data from the Ragie index.
    
    Returns:
        list[dict]: Each dict contains:
        - text: The text of the retrieved chunk
        - document_name: Name of the document
        - start_time: Start time of the chunk
        - end_time: End time of the chunk
    """
    try:
        content = retrieve_data(query)
        return content
    except Exception as e:
        return f"Failed to retrieve data: {str(e)}"

@mcp.tool()
def show_video_tool(
    document_name: str, 
    start_time: float, 
    end_time: float
) -> str:
    """Creates and saves a video chunk."""
    try:
        chunk_video(document_name, start_time, end_time)
        return "Video chunk created successfully"
    except Exception as e:
        return f"Failed to create video chunk: {str(e)}"

if __name__ == "__main__":
    mcp.run(transport='stdio')
```

### Key Features
- Video chunk extraction
- Time-based retrieval
- Index management
- Media processing integration

## 7. Synthetic Data Generation (sdv-mcp)

### Purpose
Generate and evaluate synthetic data using SDV library.

### Implementation
```python
from mcp.server.fastmcp import FastMCP
from tools import generate, evaluate, visualize

mcp = FastMCP("sdv_mcp")

@mcp.tool()
def sdv_generate(folder_name: str) -> str:
    """Generate synthetic data based on real data using SDV.
    
    This tool reads CSV files from the specified folder, creates
    synthetic version, and saves to 'synthetic_data' folder.
    
    Args:
        folder_name: Path to folder with CSV files and metadata.json
    
    Returns:
        Success message with information about generated tables
    """
    try:
        return generate(folder_name)
    except FileNotFoundError as e:
        return f"Error: {str(e)}"
    except RuntimeError as e:
        return f"Error: {str(e)}"

@mcp.tool()
def sdv_evaluate(folder_name: str) -> dict:
    """Evaluate synthetic data quality compared to real data."""
    try:
        result = evaluate(folder_name)
        return result
    except FileNotFoundError as e:
        return {"error": f"File not found: {str(e)}"}
    except RuntimeError as e:
        return {"error": f"Evaluation failed: {str(e)}"}

@mcp.tool()
def sdv_visualize(
    folder_name: str,
    table_name: str,
    column_name: str,
) -> str:
    """Generate visualization comparing real and synthetic data."""
    try:
        return visualize(folder_name, table_name, column_name)
    except FileNotFoundError as e:
        return f"Error: {str(e)}"
    except RuntimeError as e:
        return f"Error: {str(e)}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Key Features
- Data synthesis
- Quality evaluation
- Visualization generation
- Metadata-driven processing

## Common Implementation Patterns

### 1. Error Handling Strategy
All implementations use try-except blocks with descriptive error messages:
```python
try:
    result = operation()
    return f"Success: {result}"
except SpecificError as e:
    return f"Specific error: {str(e)}"
except Exception as e:
    return f"Unexpected error: {str(e)}"
```

### 2. External Service Pattern
Initialize clients at module level:
```python
load_dotenv()
client = ExternalClient(api_key=os.getenv("API_KEY"))

@mcp.tool()
def use_client(param: str) -> str:
    return client.operation(param)
```

### 3. Resource Management
Proper cleanup in database operations:
```python
conn = connect()
try:
    # Operations
    conn.commit()
finally:
    conn.close()
```

### 4. Async Support
When needed, async tools are supported:
```python
@mcp.tool()
async def async_tool(param: str) -> str:
    result = await async_operation(param)
    return result
```

### 5. Configuration Flexibility
Support multiple transport modes:
```python
parser = argparse.ArgumentParser()
parser.add_argument("--transport", choices=["stdio", "sse"])
args = parser.parse_args()
mcp.run(transport=args.transport)
```

## Implementation Complexity Levels

### Simple (1-2 tools, single file)
- agent-with-mcp-memory
- eyelevel-mcp-rag

### Medium (3-5 tools, external services)
- cursor_linkup_mcp
- mcp-video-rag
- sdv-mcp

### Complex (5+ tools, multiple modules)
- kitops-mcp
- llamaindex-mcp (with configuration)

## Conclusion
These real-world implementations demonstrate the flexibility and power of MCP servers. Common patterns include robust error handling, external service integration, and clear documentation. The examples range from simple single-tool servers to complex multi-module implementations, all following consistent patterns identified in our analysis.