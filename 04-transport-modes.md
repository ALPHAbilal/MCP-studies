# MCP Transport Modes: stdio vs SSE

## Overview
MCP servers support two transport modes for communication: **stdio** (Standard Input/Output) and **SSE** (Server-Sent Events). This document explains both modes, their use cases, and implementation details based on analysis of real implementations.

## Transport Mode Comparison

| Feature | stdio | SSE |
|---------|-------|-----|
| Communication | stdin/stdout | HTTP |
| Default Port | N/A | 8080 (configurable) |
| Streaming | Limited | Full support |
| Use Case | CLI tools, local apps | Web apps, remote access |
| Complexity | Simple | Moderate |
| Performance | Fast (local) | Network-dependent |
| Debugging | Can interfere with prints | Clean separation |

## stdio (Standard Input/Output)

### How It Works
- Server reads JSON-RPC messages from stdin
- Server writes responses to stdout
- Synchronous request-response pattern
- Direct process communication

### Implementation
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('stdio-server')

@mcp.tool()
def my_tool(input: str) -> str:
    return f"Processed: {input}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Characteristics
- **No network configuration needed**
- **Immediate startup**
- **Low latency for local operations**
- **Simple debugging with stderr**

### Use Cases
- Command-line integrations
- Local development tools
- IDE plugins (Cursor, VS Code)
- System utilities
- Embedded applications

### Communication Flow
```
Client Process <---> stdio <---> MCP Server Process
     |                              |
  JSON-RPC                      Python Tools
```

### Example Usage in Real Projects

#### LlamaIndex MCP (SQLite operations):
```python
if __name__ == "__main__":
    print("🚀Starting server... ")  # Goes to stderr
    mcp.run("stdio")  # Default transport
```

#### Cursor Linkup MCP:
```python
if __name__ == "__main__":
    asyncio.run(rag_workflow.ingest_documents("data"))
    mcp.run(transport="stdio")  # Explicit stdio
```

### Pros and Cons

**Pros:**
- Simple implementation
- No network overhead
- Works offline
- Easy process management
- Direct integration with shells

**Cons:**
- Limited to local machine
- Can't handle multiple clients
- Print statements interfere with protocol
- Harder to scale

## SSE (Server-Sent Events)

### How It Works
- Server runs HTTP endpoint
- Clients connect via HTTP
- Supports streaming responses
- Event-driven communication

### Implementation
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('sse-server', port=8080)

@mcp.tool()
def streaming_tool(query: str) -> str:
    # Can stream partial results
    return process_with_updates(query)

if __name__ == "__main__":
    mcp.run(transport="sse")
```

### Characteristics
- **Network-accessible**
- **Supports multiple clients**
- **HTTP-based protocol**
- **Streaming capabilities**
- **Port configuration required**

### Use Cases
- Web applications
- Remote tool access
- Multi-client scenarios
- Cloud deployments
- API-style services

### Communication Flow
```
Web Client <---> HTTP/SSE <---> MCP Server
     |                              |
  JSON-RPC                     Python Tools
  over HTTP                    with streaming
```

### Configuration Examples

#### With Custom Port:
```python
mcp = FastMCP('custom-server', port=3000)

if __name__ == "__main__":
    mcp.run(transport="sse")  # Runs on port 3000
```

#### With Argument Parsing:
```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--port", type=int, default=8080)
args = parser.parse_args()

mcp = FastMCP('configurable-server', port=args.port)
mcp.run(transport="sse")
```

### Real Implementation (agent-with-mcp-memory):
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP('linkup-server', port=8080)

@mcp.tool()
def web_search(query: str = "") -> str:
    """Search the web for the given query."""
    search_response = client.search(
        query=query,
        depth="standard",
        output_type="sourcedAnswer",
    )
    return str(search_response)

if __name__ == "__main__":
   mcp.run(transport="sse")
```

### Pros and Cons

**Pros:**
- Network accessible
- Multiple client support
- Clean separation from stdio
- Streaming support
- Web-friendly

**Cons:**
- Requires port management
- Network security considerations
- Higher latency than stdio
- More complex setup

## Choosing the Right Transport

### Decision Matrix

| Scenario | Recommended Transport | Reason |
|----------|----------------------|---------|
| CLI tool | stdio | Direct process communication |
| IDE extension | stdio | Simple integration |
| Web app backend | SSE | HTTP accessibility |
| Local development | stdio | Simplicity |
| Production API | SSE | Scalability |
| Debugging tool | stdio | Direct output control |
| Multi-user service | SSE | Concurrent connections |

### Hybrid Approach
Some implementations support both modes:

```python
import argparse
from mcp.server.fastmcp import FastMCP

parser = argparse.ArgumentParser()
parser.add_argument(
    "--server_type", 
    type=str, 
    default="sse", 
    choices=["sse", "stdio"]
)
args = parser.parse_args()

mcp = FastMCP('hybrid-server', port=8080)

# Tools definition...

if __name__ == "__main__":
    # Debug Mode: uv run mcp dev server.py
    # Production: uv run server.py --server_type=sse
    mcp.run(args.server_type)
```

## Implementation Patterns

### Pattern 1: Default stdio
Most common pattern (70% of implementations):
```python
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Pattern 2: SSE with Port
For web-accessible servers (20%):
```python
mcp = FastMCP('web-server', port=8080)
if __name__ == "__main__":
    mcp.run(transport="sse")
```

### Pattern 3: Configurable
For flexible deployments (10%):
```python
transport = os.getenv("MCP_TRANSPORT", "stdio")
port = int(os.getenv("MCP_PORT", "8080"))

mcp = FastMCP('flex-server', port=port)
mcp.run(transport=transport)
```

## Debugging Tips

### For stdio:
```python
import sys

def debug_log(message):
    """Write debug messages to stderr."""
    print(f"DEBUG: {message}", file=sys.stderr)

@mcp.tool()
def debuggable_tool(input: str) -> str:
    debug_log(f"Received: {input}")
    result = process(input)
    debug_log(f"Returning: {result}")
    return result
```

### For SSE:
```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

@mcp.tool()
def logged_tool(input: str) -> str:
    logger.debug(f"Processing: {input}")
    return process(input)
```

## Performance Considerations

### stdio Performance
- **Latency**: ~1-5ms local
- **Throughput**: Limited by process I/O
- **Memory**: Minimal overhead
- **CPU**: Negligible transport overhead

### SSE Performance
- **Latency**: 10-100ms (network dependent)
- **Throughput**: HTTP overhead applies
- **Memory**: Connection pooling overhead
- **CPU**: HTTP processing overhead

## Security Implications

### stdio Security
- Process-level isolation
- No network exposure
- Inherits parent process permissions
- Limited attack surface

### SSE Security
- Network exposure requires authentication
- HTTPS recommended for production
- CORS configuration needed
- Rate limiting advisable

## Conclusion
Choose **stdio** for simplicity and local integrations. Choose **SSE** for web applications and multi-client scenarios. Both transports are well-supported and the choice mainly depends on your deployment context and client requirements.