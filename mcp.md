
Here's the same minimal MCP server in Python:

**Step 1: Create the files**

Create a folder `my-test-mcp/` with these files:

**`requirements.txt`**
```
mcp
```

**`server.py`**
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("test-server")

@mcp.tool()
def hello_world(name: str) -> str:
    """Say hello and confirm MCP is working."""
    return f"Hello, {name}! MCP is working ✅"

if __name__ == "__main__":
    mcp.run()
```

**Step 2: Install dependencies**
```bash
cd my-test-mcp
pip install -r requirements.txt
```

---

**Step 3: Add it to VS Code settings**

Open VS Code settings (`Ctrl+Shift+P` → *Open User Settings JSON*) and add:

```json
{
  "mcp": {
    "servers": {
      "test-server": {
        "type": "stdio",
        "command": "python",
        "args": ["/absolute/path/to/my-test-mcp/server.py"]
      }
    }
  }
}
```

> On some systems you may need `python3` instead of `python`.

---

**Step 4: Test it**

Call the `hello_world` tool with any name. You should get back:
```
Hello, [name]! MCP is working ✅
```

That's it — only 8 lines of actual code. Let me know if you run into anything!
