# ⚡ MCP Server — Async Tool Execution

> Parallel execution of tools using asyncio.

---

# 📚 Table of Contents

- [1. Why Async Matters](#1-why-async-matters)
- [2. Sequential vs Parallel](#2-sequential-vs-parallel)
- [3. Architecture](#3-architecture)
- [4. Code Example](#4-code-example)
- [5. Dynamic Tool Registry](#-5-even-better-dynamic-tool-registry)
- [6. Production Pattern](#6-production-pattern)

---

# 1. Why Async Matters

Sequential tools are slow:

```
Weather (2s)
Stock (2s)
News (2s)
= 6 seconds
```

Async reduces it:

```
All run together = 2 seconds
````

---

# 2. Sequential vs Parallel

### Sequential
```python
weather()
stock()
news()
````

### Parallel

```python
await asyncio.gather(
    weather(),
    stock(),
    news()
)
```

---

# 3. Architecture



## 🏗 Architecture
```
User Query
     │
     ▼
LLM Tool Calls
     │
     ▼
tool_calls[]
     │
     ├── get_weather()
     ├── get_stock_price()
     ├── get_news()
     │
     ▼
asyncio.gather()
     │
     ▼
Execute All Concurrently
     │
     ▼
Collect Results
     │
     ▼
Send Back To LLM
     │
     ▼
Final Response
```

**Simplified version:**
```
User → LLM → Tool Calls
                ↓
        asyncio.gather()
                ↓
        Parallel Execution
                ↓
        Return Results
```


---

# 4. Code Example

Key pattern:

```python
results = await asyncio.gather(
    *[execute_tool(tc) for tc in tool_calls]
)
```

---


## 📦 Complete Async Multi-Tool Example
```python
import asyncio
import json
import os

from openai import OpenAI

# ============================================
# STEP 1 — ASYNC TOOLS
# ============================================

async def get_weather(city):

    # Simulate API call
    await asyncio.sleep(2)

    weather_data = {
        "hyderabad": "20°C",
        "delhi": "15°C",
        "bangalore": "18°C"
    }

    return weather_data.get(
        city.lower(),
        "Weather not available"
    )


async def get_stock_price(company):

    # Simulate API call
    await asyncio.sleep(2)

    stock_data = {
        "apple": "100",
        "microsoft": "200",
        "nimbus": "120"
    }

    return stock_data.get(
        company.lower(),
        "Stock not available"
    )


# ============================================
# STEP 2 — OPENAI CLIENT
# ============================================

client = OpenAI(
    api_key=os.environ["OPENAI_API_KEY"]
)

# ============================================
# STEP 3 — TOOL EXECUTOR
# ============================================

async def execute_tool(tool_call):

    function_name = (
        tool_call.function.name
    )

    arguments = json.loads(
        tool_call.function.arguments
    )

    if function_name == "get_weather":

        result = await get_weather(
            arguments["city"]
        )

    elif function_name == "get_stock_price":

        result = await get_stock_price(
            arguments["company"]
        )

    else:

        result = {
            "error": "Unknown tool"
        }

    return {
        "tool_call_id": tool_call.id,
        "result": result
    }


# ============================================
# STEP 4 — MAIN AGENT LOOP
# ============================================

async def main():

    user_query = (
        "Get Hyderabad weather "
        "and Nimbus stock price"
    )

    response = (
        client.chat.completions.create(
            model="gpt-4.1-mini",
            messages=[
                {
                    "role": "user",
                    "content": user_query
                }
            ],
            tools=tools,
            tool_choice="auto"
        )
    )

    message = (
        response.choices[0].message
    )

    tool_calls = message.tool_calls

    if not tool_calls:

        print(message.content)
        return

    # ============================================
    # EXECUTE ALL TOOLS CONCURRENTLY
    # ============================================

    results = await asyncio.gather(
        *[
            execute_tool(tool_call)
            for tool_call in tool_calls
        ]
    )

    # ============================================
    # BUILD TOOL MESSAGES
    # ============================================

    tool_messages = []

    for item in results:

        tool_messages.append({
            "role": "tool",
            "tool_call_id":
                item["tool_call_id"],
            "content":
                json.dumps(item["result"])
        })

    # ============================================
    # SEND RESULTS BACK
    # ============================================

    second_response = (
        client.chat.completions.create(
            model="gpt-4.1-mini",
            messages=[
                {
                    "role": "user",
                    "content": user_query
                },

                message,

                *tool_messages
            ]
        )
    )

    print(
        second_response
        .choices[0]
        .message
        .content
    )


# ============================================
# RUN
# ============================================

if __name__ == "__main__":
    asyncio.run(main())
```

---

# ⚡ 5. Even Better: Dynamic Tool Registry
Instead of large if/elif blocks:
```python
TOOLS = {
    "get_weather": get_weather,
    "get_stock_price": get_stock_price,
    "get_news": get_news,
    "search_web": search_web,
}
```

Executor:
```python
async def execute_tool(tool_call):

    name = tool_call.function.name

    args = json.loads(
        tool_call.function.arguments
    )

    func = TOOLS.get(name)

    if not func:
        return {
            "error": f"Unknown tool {name}"
        }

    result = await func(**args)

    return {
        "tool_call_id": tool_call.id,
        "result": result
    }
```

---

# 6. Production Pattern
```python
results = await asyncio.gather(
    *tasks,
    return_exceptions=True
)
```
This prevents one failed tool from crashing all others.


Example:
```python
results = await asyncio.gather(
    *tasks,
    return_exceptions=True
)

for result in results:
    if isinstance(result, Exception):
        print(f"Tool failed: {result}")
```

---


# 🔗 Related

* [Function Calling → `mcp-server-function-calling.md`](mcp-server-function-calling.md)
* [Tool Routing → `mcp-server-tool-routing.md`](mcp-server-tool-routing.md)
