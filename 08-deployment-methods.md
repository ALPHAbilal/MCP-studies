# MCP Server Deployment Methods

## Overview
This document covers various deployment methods for MCP servers, based on analysis of real implementations in the AI Engineering Hub repository. Understanding these deployment patterns is crucial for making your MCP server accessible to AI assistants like Claude, Cursor, and VS Code.

## Deployment Methods Identified

### 1. Direct Python Execution (Most Common - 70%)
The simplest and most widely used method for Python-based MCP servers.

#### How It Works
```bash
# Using Python directly
python server.py

# Using UV package manager
uv run server.py

# With arguments
python server.py --server_type=stdio
```

#### Configuration Example
**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "your-server": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

**Cursor** (Settings):
```json
{
  "mcp.servers": {
    "your-server": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

#### Pros
- Simple setup
- No build process required
- Easy debugging
- Direct access to source code

#### Cons
- Requires Python environment
- Dependencies must be installed
- Path must be absolute

#### Real Examples
- `llamaindex-mcp`: `uv run server.py --server_type=sse`
- `kitops-mcp`: `python server.py`
- `agent-with-mcp-memory`: `python server.py`

### 2. NPM Package Distribution
Used for JavaScript/TypeScript servers or when wide distribution is needed.

#### How It Works
```bash
# Install globally
npm install -g @your-org/mcp-server

# Run with npx
npx -y @your-org/mcp-server
```

#### Configuration Example
```json
{
  "mcpServers": {
    "your-server": {
      "command": "npx",
      "args": ["-y", "@your-org/mcp-server"],
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

#### Publishing Process
1. Create `package.json`:
```json
{
  "name": "@your-org/mcp-server",
  "version": "1.0.0",
  "bin": {
    "mcp-server": "./dist/server.js"
  }
}
```

2. Publish to npm:
```bash
npm publish
```

#### Pros
- Easy installation for users
- Automatic dependency management
- Version control
- Wide distribution

#### Cons
- Requires npm account
- Build process needed for TypeScript
- Updates require republishing

#### Real Example
- Journey Log original: `npm install -g @journey-log/mcp-server`

### 3. Docker Deployment
For complex environments or when isolation is needed.

#### How It Works
Create a Dockerfile:
```dockerfile
FROM python:3.12-slim
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Expose port for SSE mode
EXPOSE 8080

# Run server
CMD ["python", "server.py"]
```

Build and run:
```bash
docker build -t mcp-server .
docker run -p 8080:8080 -e API_KEY=your-key mcp-server
```

#### Docker Compose Example
```yaml
services:
  mcp-server:
    build: .
    ports:
      - "8080:8080"
    environment:
      - API_KEY=${API_KEY}
    volumes:
      - ./data:/app/data
    command: ["python", "server.py", "--transport", "sse"]
```

#### Pros
- Complete isolation
- Consistent environment
- Easy scaling
- No dependency conflicts

#### Cons
- Requires Docker
- More complex setup
- Larger resource footprint
- Harder to debug

#### Real Example
- `pixeltable-mcp`: Uses docker-compose for multiple services

### 4. PyPI Package Distribution
For Python servers that need wide distribution.

#### Setup Process

1. Create `setup.py`:
```python
from setuptools import setup, find_packages

setup(
    name="your-mcp-server",
    version="0.1.0",
    packages=find_packages(),
    install_requires=[
        "mcp[cli]>=1.10.0",
        # other dependencies
    ],
    entry_points={
        "console_scripts": [
            "your-mcp-server=server:main",
        ],
    },
    python_requires=">=3.12",
)
```

2. Build and publish:
```bash
python setup.py sdist bdist_wheel
twine upload dist/*
```

3. Users install:
```bash
pip install your-mcp-server
```

#### Configuration
```json
{
  "mcpServers": {
    "your-server": {
      "command": "your-mcp-server",
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

#### Pros
- Professional distribution
- Automatic dependency resolution
- Version management
- Easy updates via pip

#### Cons
- Requires PyPI account
- More complex setup
- Publishing process needed

### 5. Executable Script Wrapper
For simplifying complex startup procedures.

#### Create Wrapper Script

**Linux/Mac** (`run-mcp.sh`):
```bash
#!/bin/bash
cd /path/to/mcp-server
source venv/bin/activate
source .env
python server.py "$@"
```

**Windows** (`run-mcp.bat`):
```batch
@echo off
cd C:\path\to\mcp-server
call venv\Scripts\activate
python server.py %*
```

Make executable:
```bash
chmod +x run-mcp.sh
```

#### Configuration
```json
{
  "mcpServers": {
    "your-server": {
      "command": "/path/to/run-mcp.sh",
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

#### Pros
- Simplifies complex startup
- Can handle environment setup
- Platform-specific optimizations
- Easy to modify

#### Cons
- Platform-specific scripts needed
- Additional file to maintain

### 6. UV Tool Installation
Modern Python package management with UV.

#### Setup
```bash
# Install as editable package
uv pip install -e /path/to/mcp-server

# Or from git
uv pip install git+https://github.com/user/mcp-server.git
```

#### Running
```bash
# Direct execution
uv run server.py

# With dependencies
uv run --with httpx server.py
```

#### Configuration
```json
{
  "mcpServers": {
    "your-server": {
      "command": "uv",
      "args": ["run", "/path/to/server.py"],
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

#### Pros
- Fast dependency resolution
- Built-in virtual environment
- Lock file support
- Modern Python tooling

#### Cons
- Requires UV installation
- Less widespread than pip

## Deployment Decision Matrix

| Method | Best For | Complexity | Distribution | Isolation |
|--------|----------|------------|--------------|-----------|
| Direct Python | Development/Local | ⭐ | ❌ | ❌ |
| NPM Package | JS/Wide distribution | ⭐⭐ | ✅ | ❌ |
| Docker | Production/Complex | ⭐⭐⭐ | ✅ | ✅ |
| PyPI Package | Python distribution | ⭐⭐ | ✅ | ❌ |
| Script Wrapper | Local convenience | ⭐ | ❌ | ❌ |
| UV Tool | Modern Python | ⭐⭐ | ⭐ | ⭐ |

## Platform-Specific Paths

### Claude Desktop Configuration
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux**: `~/.config/Claude/claude_desktop_config.json`

### Cursor Configuration
- Access via: Settings → MCP Servers
- Or edit settings.json directly

### VS Code Configuration
- Via MCP extension settings
- Or workspace `.vscode/settings.json`

## Best Practices for Deployment

### 1. Environment Variables
Always use environment variables for sensitive data:
```json
{
  "env": {
    "API_KEY": "your-key",
    "DEBUG": "false"
  }
}
```

### 2. Absolute Paths
Always use absolute paths in configurations:
```json
{
  "args": ["/home/user/mcp-server/server.py"]  // Good
  "args": ["./server.py"]  // Bad - relative path
}
```

### 3. Error Handling
Ensure your server handles missing dependencies gracefully:
```python
try:
    import required_module
except ImportError:
    print("Please install dependencies: pip install -r requirements.txt", file=sys.stderr)
    sys.exit(1)
```

### 4. Logging
Use stderr for logs to avoid interfering with stdio transport:
```python
import sys
import logging

logging.basicConfig(
    stream=sys.stderr,
    level=logging.INFO
)
```

### 5. Version Pinning
Pin your dependencies for consistent deployments:
```txt
mcp[cli]==1.10.0  # Exact version
httpx>=0.27.0,<1.0.0  # Version range
```

## Quick Start Templates

### For New Python MCP Server
```bash
# 1. Clone/create your server
git clone your-mcp-server
cd your-mcp-server

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure
cp .env.example .env
# Edit .env with your settings

# 4. Test
python server.py

# 5. Configure in Claude/Cursor
# Add to config with absolute path
```

### For Production Deployment
```bash
# 1. Dockerize
docker build -t mcp-server .

# 2. Run with compose
docker-compose up -d

# 3. Configure Claude/Cursor to connect to localhost:8080
```

## Troubleshooting Common Deployment Issues

### Issue: "Command not found"
**Solution**: Use absolute paths or ensure command is in PATH

### Issue: "Module not found"
**Solution**: Install dependencies in correct environment

### Issue: "Permission denied"
**Solution**: Make scripts executable with `chmod +x`

### Issue: "Cannot connect to server"
**Solution**: Check firewall, ports, and transport mode

### Issue: "API key not found"
**Solution**: Verify environment variables are passed correctly

## Conclusion
Choose deployment method based on:
- **Development**: Direct Python execution
- **Distribution**: NPM or PyPI packages
- **Production**: Docker containers
- **Convenience**: Script wrappers
- **Modern Python**: UV tools

Most MCP servers in the repository use direct Python execution for simplicity, with configuration files pointing to the absolute path of the server script.