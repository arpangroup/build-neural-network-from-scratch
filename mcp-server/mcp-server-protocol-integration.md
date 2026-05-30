# MCP Protocol Integration

> **Related Example:** [Function Calling Example](mcp-server-function-calling.md)

## Table of Contents

* [Overview](#overview)
* [Traditional Function Calling](#traditional-function-calling)
* [MCP-Based Architecture](#mcp-based-architecture)
* [Option 1: OpenAI Agents SDK (Recommended)](#option-1-openai-agents-sdk-recommended)

  * [Install Dependencies](#install-dependencies)
  * [Weather MCP Server](#weather-mcp-server)
  * [Stock MCP Server](#stock-mcp-server)
  * [Start the Servers](#start-the-servers)
  * [Client Example](#client-example)
  * [**Complete MCP Example Using OpenAI Agents SDK**](#complete-mcp-example-using-openai-agents-sdk)
  * [What the SDK Handles Automatically](#what-the-sdk-handles-automatically)
* [Option 2: Chat Completions API](#option-2-chat-completions-api)

  * [Architecture Flow](#architecture-flow)
  * [Step 1: Connect to MCP](#step-1-connect-to-mcp)
  * [Step 2: Discover Available Tools](#step-2-discover-available-tools)
  * [Step 3: Convert MCP Tools to OpenAI Tool Schema](#step-3-convert-mcp-tools-to-openai-tool-schema)
  * [Step 4: Send Tools to the Model](#step-4-send-tools-to-the-model)
  * [Step 5: Execute Tool Calls Through MCP](#step-5-execute-tool-calls-through-mcp)
  * [Minimal Change Version](#minimal-change-version)
* [DIY: Build your own MCP Server from Scratch](#build-your-own-mcp-server-from-scratch) 
* [When Should You Use MCP?](#when-should-you-use-mcp)

---

## Overview

The **Model Context Protocol (MCP)** provides a standardized way for LLMs to discover and invoke tools exposed by external services.

Instead of manually defining tool schemas and implementing tool routing logic, MCP allows tools to be dynamically discovered from MCP servers.

```mermaid

```


```
                        ┌──────────────────────────┐
                        │        MCP Client        │
                        │ (Claude / Agent / SDK)   │
                        └───────────┬──────────────┘
                                    │
                                    │ discovers
                                    ▼
                    ┌─────────────────────────────────┐
                    │         MCP Server              │
                    │   (FastMCP / custom server)     │
                    └───────────┬─────────┬───────────┘
                                │         │
          ┌─────────────────────┘         └─────────────────────┐
          │                                                     │
          ▼                                                     ▼

┌──────────────────────┐                         ┌──────────────────────┐
│        TOOLS         │                         │      RESOURCES       │
│                      │                         │                      │
│ "Do something"       │                         │ "Read context"       │
│                      │                         │                      │
│ - get_weather()      │                         │ - file://readme      │
│ - get_stock_price()  │                         │ - git://repo/main.py │
│ - create_ticket()    │                         │ - db://schema        │
└──────────┬───────────┘                         └──────────┬───────────┘
           │                                                │
           │ tool call                                      │ resource read
           ▼                                                ▼

     ┌────────────────┐                            ┌──────────────────┐
     │  Executes code │                            │ Returns content  │
     │  / API / DB    │                            │ (read-only data) │
     └──────┬─────────┘                            └─────────┬────────┘
            │                                               │
            └──────────────────────┬────────────────────────┘
                                   ▼

                     ┌──────────────────────────┐
                     │         PROMPTS          │
                     │                          │
                     │ "How to think / format"  │
                     │                          │
                     │ - investment_brief()     │
                     │ - report_template()      │
                     └──────────┬───────────────┘
                                │
                                │ injected instruction
                                ▼

                     ┌──────────────────────────┐
                     │     LLM (Reasoning)      │
                     │  combines all inputs     │
                     │                          │
                     │ tools + resources +      │
                     │ prompts → final answer   │
                     └──────────────────────────┘
```

---

## Traditional Function Calling

In a standard function-calling setup:

```text
LLM
  └── Function Calling
         ├── get_weather()
         └── get_stock_price()
```

The tools are:

* Hardcoded inside the application
* Manually defined using JSON schemas
* Passed through the `tools=` parameter

---

## MCP-Based Architecture

With MCP:

```text
LLM Client
     │
     ├── MCP Server (Weather)
     │        └── get_weather
     │
     └── MCP Server (Stocks)
              └── get_stock_price
```

### Key Difference

The model discovers tools from MCP servers dynamically instead of requiring manually maintained tool definitions.

---

# Option 1: OpenAI Agents SDK (Recommended)

The Agents SDK provides built-in MCP integration and handles most of the complexity automatically.

## Install Dependencies

```bash
pip install openai-agents mcp
```

---

## Weather MCP Server

**weather_server.py**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def get_weather(city: str):
    weather_data = {
        "delhi": "15°C",
        "mumbai": "12°C",
        "kolkata": "10°C",
        "hyderabad": "20°C",
        "bangalore": "18°C"
    }

    return weather_data.get(
        city.lower(),
        "Weather data not available"
    )

if __name__ == "__main__":
    mcp.run()
```

---

## Stock MCP Server

**stock_server.py**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("stocks")

@mcp.tool()
def get_stock_price(company: str):
    stock_data = {
        "apple": "100",
        "microsoft": "200",
        "google": "150",
        "meta": "180",
        "nimbus": "120"
    }

    return stock_data.get(
        company.lower(),
        "Stock data not available"
    )

if __name__ == "__main__":
    mcp.run()
```

---

## Start the Servers

```bash
python weather_server.py
python stock_server.py
```

---

## Client Example

```python
import asyncio

from agents import Agent, Runner
from agents.mcp import MCPServerStdio

async def main():

    weather_server = MCPServerStdio(
        params={
            "command": "python",
            "args": ["weather_server.py"]
        }
    )

    stock_server = MCPServerStdio(
        params={
            "command": "python",
            "args": ["stock_server.py"]
        }
    )

    agent = Agent(
        name="Assistant",
        model="gpt-4.1-mini",
        mcp_servers=[
            weather_server,
            stock_server
        ]
    )

    result = await Runner.run(
        agent,
        "Get Hyderabad weather and Nimbus stock price"
    )

    print(result.final_output)

asyncio.run(main())
```

# Complete MCP Example Using OpenAI Agents SDK
Step1: `tools.py`
```python
from mcp.server.fastmcp import FastMCP

# Create one MCP server
mcp = FastMCP("data_services")

# -------------------
# TOOLS (actions)
# -------------------
@mcp.tool()
def get_weather(city: str) -> str:
    ...

@mcp.tool()
def get_stock_price(company: str) -> str:
    ...


# -------------------
# RESOURCES (context)
# -------------------
@mcp.resource("market://overview")
def market_overview() -> str:
    return """
    Global Market Overview:
    - Tech sector is bullish
    - Energy sector is stable
    - Inflation trending down
    """

@mcp.resource("weather://notes")
def weather_notes() -> str:
    return """
    Weather Notes:
    - Monsoon expected in South India
    - North India has cold wave conditions
    """

@mcp.resource("file://readme")
def readme_file() -> str:
    """
    Expose a local README.md file as a resource
    """
    return Path("README.md").read_text(encoding="utf-8")

@mcp.resource("file://config")
def config_file() -> str:
    """
    Expose a config file
    """
    return Path("config.json").read_text(encoding="utf-8")

# -------------------
# OPTIONAL: PROMPTS
# -------------------
@mcp.prompt()
def investment_brief() -> str:
    return """
    You are a financial analyst.
    Summarize stock data and provide insights.
    """

if __name__ == "__main__":
    mcp.run()
```


Step2: `client.py`
```python
import asyncio

from agents import Agent, Runner
from agents.mcp import MCPServerStdio


async def main():

    data_server = MCPServerStdio(
        params={
            "command": "python",
            "args": ["data_server.py"]
        }
    )

    agent = Agent(
        name="Assistant",
        model="gpt-4.1-mini",
        mcp_servers=[data_server]
    )

    result = await Runner.run(
        agent,
        "What's the weather in Hyderabad and the stock price of Apple?"
    )

    print(result.final_output)


asyncio.run(main())
```

---

## What the SDK Handles Automatically

The Agents SDK:

1. Connects to MCP servers
2. Discovers available tools
3. Generates tool schemas
4. Executes tool calls
5. Sends tool results back to the model
6. Produces the final response

> No manual `tool_messages` handling is required.

---

# Option 2: Chat Completions API

If you want to keep your existing:

```python
client.chat.completions.create(...)
```

workflow, MCP can act as a dynamic tool registry.

---

## Architecture Flow

```text
User
 ↓
ChatCompletion
 ↓
Tool Call Request
 ↓
MCP Client
 ↓
MCP Server
 ↓
Tool Result
 ↓
ChatCompletion
 ↓
Final Answer
```

---

## Step 1: Connect to MCP

```python
from mcp import ClientSession
from mcp.client.stdio import stdio_client
```

---

## Step 2: Discover Available Tools

```python
tools = await session.list_tools()
```

Example response:

```python
[
    {
        "name": "get_weather",
        "description": "...",
        "inputSchema": {...}
    },
    {
        "name": "get_stock_price",
        "description": "...",
        "inputSchema": {...}
    }
]
```

---

## Step 3: Convert MCP Tools to OpenAI Tool Schema

```python
openai_tools = []

for tool in mcp_tools:
    openai_tools.append({
        "type": "function",
        "function": {
            "name": tool.name,
            "description": tool.description,
            "parameters": tool.inputSchema
        }
    })
```

---

## Step 4: Send Tools to the Model

```python
response = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=messages,
    tools=openai_tools
)
```

---

## Step 5: Execute Tool Calls Through MCP

Instead of manually routing tool calls:

```python
if function_name == "get_weather":
    result = get_weather(arguments["city"])

elif function_name == "get_stock_price":
    result = get_stock_price(arguments["company"])
```

execute them dynamically through MCP:

```python
result = await session.call_tool(
    function_name,
    arguments
)
```

---

## Minimal Change Version

Replace:

```python
if function_name == "get_weather":
    result = get_weather(arguments["city"])

elif function_name == "get_stock_price":
    result = get_stock_price(arguments["company"])
```

with:

```python
result = await mcp_session.call_tool(
    function_name,
    arguments
)
```

and replace:

```python
tools = [...]
```

with:

```python
tools = mcp_tools_to_openai_schema(
    await session.list_tools()
)
```

This allows your application to discover and execute tools dynamically through MCP.

---

# Build your own MCP Server from Scratch
Steps:
1. Register tools.
2. Expose tool metadata.
3. Accept tool calls.
4. Execute the matching function.
5. Return results.

Here's a simplified implementation.

### Step 1: Create a Mini MCP Framework
```python
import inspect
import json


class MiniMCP:
    def __init__(self, name):
        self.name = name
        self.tools = {}

    def tool(self):
        """
        Decorator to register a tool.
        """

        def decorator(func):
            self.tools[func.__name__] = func
            return func

        return decorator

    def list_tools(self):
        """
        Return metadata for all tools.
        """

        result = []

        for name, func in self.tools.items():
            sig = inspect.signature(func)

            properties = {}
            required = []

            for param_name, param in sig.parameters.items():
                properties[param_name] = {
                    "type": "string"
                }
                required.append(param_name)

            result.append({
                "name": name,
                "description": func.__doc__ or "",
                "inputSchema": {
                    "type": "object",
                    "properties": properties,
                    "required": required
                }
            })

        return result

    def call_tool(self, tool_name, arguments):
        if tool_name not in self.tools:
            raise ValueError(
                f"Unknown tool: {tool_name}"
            )

        func = self.tools[tool_name]
        return func(**arguments)
```

### Step 2: Create Your MCP Server
```python
mcp = MiniMCP("data_server")

@mcp.tool()
def get_weather(city: str):
    ...

@mcp.tool()
def get_stock_price(company: str):
    ...
```

### Step 3: Tool Discovery
```python
print(
    json.dumps(
        mcp.list_tools(),
        indent=2
    )
)
```
Output:
```json
[
  {
    "name": "get_weather",
    "description": "Get weather for a city.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string"
        }
      },
      "required": ["city"]
    }
  },
  {
    "name": "get_stock_price",
    "description": "Get stock price.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "company": {
          "type": "string"
        }
      },
      "required": ["company"]
    }
  }
]
```

### Step 4: Dynamic Tool Execution
Instead of:
```python
if function_name == "get_weather":
    ...
elif function_name == "get_stock_price":
    ...
```

you can do:
```python
result = mcp.call_tool(
    "get_weather",
    {"city": "Hyderabad"}
)

print(result)
```


Output:
```
20°C
```

### Step 5: Add a Simple JSON Protocol
A request:
```json
{
  "method": "call_tool",
  "tool": "get_weather",
  "arguments": {
    "city": "Hyderabad"
  }
}
```
can be handled like:
```python
def handle_request(request):
    method = request["method"]

    if method == "list_tools":
        return mcp.list_tools()

    if method == "call_tool":
        return mcp.call_tool(
            request["tool"],
            request["arguments"]
        )

    raise ValueError("Unknown method")
```

### Step 6: Run Over STDIO (Very Simplified)
This starts to resemble how MCP transports work.
```python
while True:
    line = input()
    request = json.loads(line)
    response = handle_request(request)
    print(
        json.dumps(response),
        flush=True
    )
```
Client sends:
```json
{"method":"list_tools"}
```
Server responds:
```json
[
  {
    "name":"get_weather",
    ...
  }
]
```
Client sends:
```json
{
  "method":"call_tool",
  "tool":"get_stock_price",
  "arguments":{
    "company":"Apple"
  }
}
```
Server responds:
```json
"100"
```

---

## Will this custom MCP client work with the Claude Desktop app?

**No it will not recognize by Claude, because Claude Desktop has no idea how to talk to that custom protocol.**

**Claude Desktop expects a real MCP server that implements the MCP protocol**, not just a custom JSON interface that happens to have list_tools() and call_tool() methods.

Above mini implementation demonstrates the concepts, but Claude Desktop won't recognize it because Claude expects:

- MCP protocol messages (JSON-RPC based)
- MCP initialization handshake
- Capability negotiation
- Standard MCP methods (tools/list, tools/call, etc.)
- Proper STDIO transport handling
- MCP-compliant message framing and lifecycle

## What works with Claude Desktop
✅ A server built with the official MCP SDK:
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-demo-server")

@mcp.tool()
def get_weather(city: str):
    return "20°C"

if __name__ == "__main__":
    mcp.run()
```


# When Should You Use MCP?

Use MCP when:

✅ Tools live in separate services

✅ Multiple applications need access to the same tools

✅ You want dynamic tool discovery

✅ You expose databases, APIs, file systems, GitHub, Slack, or internal services as tools

✅ You manage dozens or hundreds of tools

---

## When Not to Use MCP

For a small application with only a few helper functions:

```python
get_weather()
get_stock_price()
```

regular function calling is usually simpler and requires less infrastructure.

---

## Summary

| Feature                   | Function Calling | MCP          |
| ------------------------- | ---------------- | ------------ |
| Tool Discovery            | Manual           | Automatic    |
| Schema Definition         | Manual           | Generated    |
| Tool Routing              | Manual           | Standardized |
| Tool Reusability          | Limited          | High         |
| Multi-Application Support | Difficult        | Easy         |
| Scales to Many Tools      | Poorly           | Well         |

**Rule of Thumb:** Use regular function calling for a handful of local functions. Use MCP when tools become shared services that need to be discovered and consumed by multiple LLM clients.
