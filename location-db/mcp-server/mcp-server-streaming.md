# 🚀 MCP Server — Streaming & Real-Time Agents

> Token streaming + tool execution UX patterns.

> Streaming makes the model send tokens as they're generated instead of waiting for the complete response.

---


# 📚 Table of Contents

- [1. What is Streaming](#1-what-is-streaming)
- [2. Basic Streaming Example](#2-basic-streaming-example)
- [3. Async Streaming](#3-async-streaming)
- [4. Tool + Streaming Flow](#4-tool--streaming-flow)
- [5. Production Architecture](#5-production-architecture)
- [6. Full Async Streaming Agent](#6-full-async-streaming-agent)
- [7. Stream Events to Frontend (FastAPI)](#7-stream-events-to-frontend-fastapi)
- [8. Recommended Production Architecture](#8-recommended-production-architecture)

---


# 1. What is Streaming

Instead of waiting for full response:

```
Tokens arrive gradually
````

---

# 2. Basic Streaming Example

```python
stream = client.chat.completions.create(stream=True)
````

```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_API_KEY"
)

stream = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {"role": "user", "content": "Explain AI agents"}
    ],
    stream=True
)

for chunk in stream:
    delta = (chunk.choices[0].delta.content)
    if delta:
        print(delta, end="", flush=True)
```

---

# 3. Async Streaming

```python
async for chunk in stream:
    print(chunk.choices[0].delta.content)
```

---

# 4. Tool + Streaming Flow

A common misconception:

```
Stream Tool Call
↓
Execute Tool
↓
Continue Stream
```
Most APIs don't work exactly like that.


Typical flow:
```
User Query
      ↓
Model decides tools
      ↓
Execute tools
      ↓
Send tool results
      ↓
Stream final answer
```

## Phase 1: Detect Tool Calls
```python
response = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=messages,
    tools=tools
)

message = response.choices[0].message
```
If:
```python
message.tool_calls
```
exists, execute the tools.

## Phase 2: Execute Tools
```python
tool_messages = []

for tool_call in message.tool_calls:
    result = execute_tool(tool_call)
    tool_messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": json.dumps(result)
    })
```

## Phase 3: Stream Final Response
```python
stream = client.chat.completions.create(
    model="gpt-4.1-mini",
    stream=True,
    messages=[
        *messages,
        message,
        *tool_messages
    ]
)

for chunk in stream:
    delta = (chunk.choices[0].delta.content)

    if delta:
        print(delta, end="", flush=True)
```

---


Example:
```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def stream_response():
    stream = await client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "user", "content": "Explain MCP"}
        ],
        stream=True
    )

    async for chunk in stream:
        content = (chunk.choices[0].delta.content)
        if content:
            print(content, end="")
```


---


# 5. Production Architecture

* FastAPI
* SSE / WebSockets
* AsyncOpenAI
* Tool executor

---

# 6. Full Async Streaming Agent
```python
import asyncio
import json

from openai import AsyncOpenAI

client = AsyncOpenAI()


async def get_weather(city):
    await asyncio.sleep(1)
    return {
        "city": city,
        "temp": "20°C"
    }


TOOLS = {
    "get_weather": get_weather
}


async def execute_tool(tool_call):
    name = tool_call.function.name
    args = json.loads(tool_call.function.arguments)
    func = TOOLS[name]
    result = await func(**args)
    return {
        "tool_call_id": tool_call.id,
        "content": json.dumps(result)
    }


async def chat(user_query):
    response = await (
        client.chat.completions.create(
            model="gpt-4.1-mini",
            messages=[
                {
                    "role": "user",
                    "content": user_query
                }
            ],
            tools=tools
        )
    )

    message = (response.choices[0].message)

    if message.tool_calls:
        results = await asyncio.gather(
            *[
                execute_tool(tc)
                for tc in message.tool_calls
            ]
        )

        tool_messages = [
            {
                "role": "tool",
                "tool_call_id": r["tool_call_id"],
                "content": r["content"]
            }
            for r in results
        ]

        stream = await (
            client.chat.completions.create(
                model="gpt-4.1-mini",
                stream=True,
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

        async for chunk in stream:
            delta = (chunk.choices[0].delta.content)

            if delta:
                print(delta, end="", flush=True)
    else:

        print(message.content)
```

---

# 7. Stream Events to Frontend (FastAPI)
For web apps, stream tokens through Server-Sent Events (SSE).
```python   
from fastapi.responses import StreamingResponse
```

Generator::
```python
async def generate():
    async for chunk in stream:
        delta = (
            chunk.choices[0]
            .delta
            .content
        )

        if delta:
            yield (
                f"data: {delta}\n\n"
            )
```

Endpoint:
```python
@app.get("/chat")
async def chat():
    return StreamingResponse(
        generate(),
        media_type=
        "text/event-stream"
    )
```
Frontend receives tokens in real time.

## Stream Tool Status Updates
A nice UX pattern:
```
🔍 Looking up weather...
📈 Fetching stock price...
✅ Results received

The weather in Hyderabad is 20°C...
```

Example:
```python
yield {
    "type": "status",
    "message":
    "Fetching weather..."
}
```
Then:
```python
yield {
    "type": "token",
    "content": token
}
```
This is how many modern AI agent UIs show progress while tools are running.

# 8. Recommended Production Architecture
```
User
 │
 ▼
LLM Request
 │
 ▼
Tool Calls?
 ├── No
 │     └── Stream Response
 │
 └── Yes
       │
       ▼
  Async Tool Execution
       │
       ▼
  Collect Results
       │
       ▼
  Stream Final Answer
       │
       ▼
    Frontend
```

# ⭐ Advanced Enhancements
Once basic streaming works, you can add:

1. Token streaming → stream generated text.
2. Tool status streaming → show progress while tools run.
3. Partial tool results → stream results as each tool finishes.
4. WebSocket streaming → bi-directional communication.
5. Multi-agent streaming → show which agent is currently working.
6. Reasoning/status events → "Searching...", "Analyzing...", "Generating answer...".
7. Cancellation support → stop generation when user clicks Cancel.

A common production setup is:
```
FastAPI
 + AsyncOpenAI
 + asyncio.gather()
 + SSE/WebSocket
 + Tool Calling
 + Streaming Tokens
```



# 🔗 Related

* [Async Execution → `mcp-server-async-execution.md`](mcp-server-async-execution.md)
* [Function Calling → `mcp-server-function-calling.md`](mcp-server-function-calling.md)
