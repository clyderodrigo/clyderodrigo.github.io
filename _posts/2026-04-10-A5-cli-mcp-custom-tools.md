# Beyond the Chat Box: Extending Claude and GPT Agents with CLI, MCP, and Custom Tools

## Key Takeaways

- Tools are the agent's hands. Without them, it's just text generation.
- Validate tool inputs at the execution layer, not in the prompt. The model can be wrong or manipulated.
- MCP decouples tool servers from agents — build once, connect anywhere. Invest in it for production toolchains.
- Keep tools single-responsibility. Multi-purpose tools produce ambiguous failure modes.
- Return structured output from tools. The model navigates JSON; it hallucinates around prose.
- Cap command execution time and output size. An agent that hangs on a tool call hangs forever.

---

A model without tools is a very expensive autocomplete. Tools are what transform a language model into an agent — something that can reach into systems, run code, query databases, and produce real outcomes.

This article covers three layers of tool integration: CLI execution for system-level access, MCP (Model Context Protocol) for composable tool servers, and custom function calling for domain-specific capabilities.

---

## The Tool Execution Model

Both OpenAI and Anthropic implement tool calling with the same fundamental loop:

```
1. Agent sends message + tool definitions to model
2. Model decides to call a tool — returns tool name + arguments
3. Agent executes the tool in its environment
4. Agent sends tool result back to model
5. Model continues — calls more tools or returns final answer
```

The model never executes tools. It only requests them. Your code runs the tool and feeds results back. This is the boundary you control.

---

## Function Calling: The Foundation

### Defining Tools (OpenAI)

```python
from openai import OpenAI
import json, subprocess

client = OpenAI()

tools = [
    {
        "type": "function",
        "function": {
            "name": "run_shell_command",
            "description": "Execute a shell command and return stdout/stderr. Use for system queries, file operations, process management.",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {
                        "type": "string",
                        "description": "The shell command to run. Must be non-destructive unless explicitly confirmed."
                    }
                },
                "required": ["command"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "query_database",
            "description": "Run a read-only SQL query against the operations database.",
            "parameters": {
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SELECT statement only."}
                },
                "required": ["sql"]
            }
        }
    }
]
```

### Defining Tools (Anthropic)

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "run_shell_command",
        "description": "Execute a shell command. Returns stdout and stderr.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "Shell command to execute."}
            },
            "required": ["command"]
        }
    }
]
```

---

## CLI Integration: System-Level Access

Shell access is the most direct tool — and the most dangerous if unsecured. Implement it right.

```python
import subprocess
import shlex

# Allowlist of permitted commands (enforce at execution layer, not prompt layer)
ALLOWED_COMMANDS = {
    "kubectl", "docker", "git", "npm", "pip",
    "ls", "cat", "grep", "find", "ps", "df", "top"
}

BLOCKED_PATTERNS = [
    "rm -rf", "mkfs", "dd if=", "> /dev/",
    "chmod 777", "curl | bash", "wget | sh"
]

def run_shell_command(command: str) -> dict:
    # Security: check for blocked patterns before executing
    for pattern in BLOCKED_PATTERNS:
        if pattern in command:
            return {"error": f"Command blocked: contains '{pattern}'"}

    # Validate base command is in allowlist
    base_cmd = shlex.split(command)[0] if command.strip() else ""
    if base_cmd not in ALLOWED_COMMANDS:
        return {"error": f"Command '{base_cmd}' not in allowlist"}

    try:
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=30,        # prevent hung processes
            cwd="/tmp/sandbox" # constrain working directory
        )
        return {
            "stdout": result.stdout[:8000],  # cap output size
            "stderr": result.stderr[:2000],
            "returncode": result.returncode
        }
    except subprocess.TimeoutExpired:
        return {"error": "Command timed out after 30 seconds"}
    except Exception as e:
        return {"error": str(e)}
```

**Security principle:** Never execute arbitrary shell commands from model output against a production system. The allowlist and blocked-pattern check are non-negotiable for any internet-facing or multi-tenant agent.

---

## The Agent Loop: Wiring Tools to Model

```python
def run_agent(user_request: str, max_turns: int = 10) -> str:
    messages = [{"role": "user", "content": user_request}]
    tool_registry = {
        "run_shell_command": run_shell_command,
        "query_database": query_database,
    }

    for turn in range(max_turns):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )

        message = response.choices[0].message
        messages.append(message)

        # No tool call — agent is done
        if not message.tool_calls:
            return message.content

        # Execute all requested tool calls
        for tool_call in message.tool_calls:
            fn_name = tool_call.function.name
            fn_args = json.loads(tool_call.function.arguments)

            if fn_name not in tool_registry:
                result = {"error": f"Unknown tool: {fn_name}"}
            else:
                result = tool_registry[fn_name](**fn_args)

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            })

    return "Max turns reached without resolution."
```

---

## MCP: Model Context Protocol

MCP is Anthropic's open standard for connecting models to external tool servers. Instead of hardcoding tools into your agent, you run MCP servers as independent processes that expose capabilities over a standard protocol. Your agent discovers and calls them dynamically.

**Architecture:**

```
┌─────────────┐    stdio/SSE    ┌──────────────────┐
│  AI Agent   │ ◄──────────── ► │  MCP Server      │
│ (Claude or  │                 │  (filesystem,    │
│  GPT-4o)   │                 │   database, API) │
└─────────────┘                 └──────────────────┘
```

### Running an MCP Server

```bash
# Install MCP servers
pip install mcp

# Official servers available:
# - @modelcontextprotocol/server-filesystem
# - @modelcontextprotocol/server-postgres
# - @modelcontextprotocol/server-github
# - @modelcontextprotocol/server-slack
```

### Building a Custom MCP Server

```python
# my_ops_server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types
import subprocess

server = Server("ops-tools")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_service_status",
            description="Get the current status of a named service",
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {"type": "string"}
                },
                "required": ["service_name"]
            }
        ),
        types.Tool(
            name="tail_logs",
            description="Get last N lines of service logs",
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {"type": "string"},
                    "lines": {"type": "integer", "default": 50}
                },
                "required": ["service_name"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_service_status":
        result = subprocess.run(
            ["systemctl", "status", arguments["service_name"]],
            capture_output=True, text=True
        )
        return [types.TextContent(type="text", text=result.stdout)]

    if name == "tail_logs":
        lines = arguments.get("lines", 50)
        result = subprocess.run(
            ["journalctl", "-u", arguments["service_name"], "-n", str(lines)],
            capture_output=True, text=True
        )
        return [types.TextContent(type="text", text=result.stdout)]

    raise ValueError(f"Unknown tool: {name}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(server))
```

### Connecting Claude to an MCP Server

```python
import anthropic
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run_with_mcp(user_request: str):
    server_params = StdioServerParameters(
        command="python",
        args=["my_ops_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Discover tools from MCP server
            tools_result = await session.list_tools()
            anthropic_tools = [
                {
                    "name": tool.name,
                    "description": tool.description,
                    "input_schema": tool.inputSchema
                }
                for tool in tools_result.tools
            ]

            client = anthropic.Anthropic()
            response = client.messages.create(
                model="claude-sonnet-4",
                max_tokens=2048,
                tools=anthropic_tools,
                messages=[{"role": "user", "content": user_request}]
            )

            # Handle tool calls
            for block in response.content:
                if block.type == "tool_use":
                    result = await session.call_tool(block.name, block.input)
                    print(f"Tool result: {result.content}")
```

### MCP Server Composition

The real power of MCP is composing multiple servers. One agent can connect to filesystem, database, GitHub, and Slack servers simultaneously:

```json
// mcp_config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {"POSTGRES_URL": "postgresql://localhost/ops_db"}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "ghp_..."}
    }
  }
}
```

---

## Custom Tool Design Principles

Tool quality directly determines agent reliability. A poorly designed tool produces confused, correct-sounding but wrong behavior.

**1. One responsibility per tool**

```python
# Bad — does too much, model can't reason about failure mode
def deploy_and_notify(service, environment, slack_channel): ...

# Good — separate concerns
def deploy_service(service, environment): ...
def send_slack_notification(channel, message): ...
```

**2. Return structured, navigable output**

```python
# Bad — model has to parse prose
return "The service is running and memory usage is 87%"

# Good — structured, queryable
return {
    "status": "running",
    "pid": 14823,
    "memory_percent": 87.3,
    "uptime_seconds": 86400
}
```

**3. Always return errors in-band**

```python
# Bad — exception propagates, breaks agent loop
def get_metrics(service_name):
    data = fetch_metrics(service_name)  # raises if service not found
    return data

# Good — agent can reason about and recover from errors
def get_metrics(service_name):
    try:
        return {"status": "ok", "data": fetch_metrics(service_name)}
    except ServiceNotFoundError:
        return {"status": "error", "error": f"Service '{service_name}' not found"}
```

