# 🚀 MCP Server — Function Calling / Tool Calling in Python

> Learn how modern AI models use **Function Calling / Tool Calling** to interact with external tools, APIs, databases, and services.

---

# 📚 Table of Contents

* [1. Introduction](#-introduction)
* [2. Single Tool Call](#-2-single-tool-call)

  * [Step-by-Step Flow](#-step-by-step-flow)
  * [Complete Code](#-complete-code-single-tool-call)
  * [Output](#-output)
* [3. Multiple Tool Calls](#-3-multiple-tool-calls)

  * [How Multi-Tool Agents Work](#-how-multi-tool-agents-work)
  * [Complete Code](#-complete-code-multiple-tool-calls)
* [4. Important Concepts](#-4-important-concepts)
* [5. Real-World Use Cases](#-5-real-world-use-cases)
* [Next Steps](#-next-steps)
    * [6. Async Tool](#-6-async-tool-execution)
    * [7. Dynamic Tool Registry ](#-7-even-better-dynamic-tool-registry)    
    * [8. Streaming responses](#-8-implementing-streaming-responses-for-tool-calling-agents)

* [MCP protocol integration]()

---

# 📖 Introduction

Modern LLMs like GPT-4, Claude, Gemini, and Llama can:

* Call functions
* Use tools
* Access APIs
* Query databases
* Trigger workflows

This mechanism is called:

* **Function Calling**
* **Tool Calling**
* **AI Agent Tool Usage**

---

# ⚡ 2. Single Tool Call

In this example:

* User asks a question
* Model decides which function to call
* Backend executes the function
* Result is sent back to the model
* Model generates final human-readable response

---

## 🔄 Step-by-Step Flow

```text
User Query
    ↓
LLM Receives Tools
    ↓
LLM Chooses Tool
    ↓
Backend Executes Function
    ↓
Tool Result Returned
    ↓
LLM Generates Final Answer
```

---

## 🧠 Complete Code (Single Tool Call)

```python
from openai import OpenAI
import json
import os

# ============================================
# STEP 1 — DEFINE TOOL/FUNCTIONS
# ============================================

def get_weather(city):
    weather_data = {
        "delhi": "15°C",
        "mumbai": "12°C",
        "kolkata": "10°C",
        "hyderabad": "20°C",
        "bangalore": "18°C"
    }

    return weather_data.get(
        city.lower(),
        {"error": "Weather data not available"}
    )


def get_stock_price(company):
    stock_data = {
        "apple": "100",
        "microsoft": "200",
        "google": "150",
        "meta": "180",
        "nimbus": "120"
    }

    return stock_data.get(
        company.lower(),
        {"error": "Stock data not available"}
    )


# ============================================
# STEP 2 — CREATE OPENAI CLIENT
# ============================================

client = OpenAI(
    api_key=os.environ.get("YOUR_OPENAI_API_KEY"),
    base_url="https://api.groq.com/openai/v1",
)

# ============================================
# STEP 3 — DEFINE TOOL SCHEMAS
# ============================================

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather information for a city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "City name"
                    }
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_stock_price",
            "description": "Get stock price by company name",
            "parameters": {
                "type": "object",
                "properties": {
                    "company": {
                        "type": "string",
                        "description": "Company name"
                    }
                },
                "required": ["company"]
            }
        }
    }
]

# ============================================
# STEP 4 — USER QUERY
# ============================================

user_query = "What is the weather in Hyderabad?"
# user_query = "What is Apple stock price?"
# user_query = "Get Hyderabad weather and Nimbus stock price"

# ============================================
# STEP 5 — SEND QUERY TO MODEL
# ============================================

response = client.chat.completions.create(
    model="llama-3.3-70b-versatile",
    messages=[
        {
            "role": "user",
            "content": user_query
        }
    ],
    tools=tools,
    tool_choice="auto"
)

# ============================================
# STEP 6 — CHECK TOOL CALLS
# ============================================

message = response.choices[0].message

print(f"Message: {message}")

tool_calls = message.tool_calls

if tool_calls:

    tool_call = tool_calls[0]

    function_name = tool_call.function.name

    arguments = json.loads(
        tool_call.function.arguments
    )

    print("\nTool Selected:")
    print(function_name)

    print("\nArguments:")
    print(arguments)

    # ============================================
    # STEP 7 — EXECUTE FUNCTION
    # ============================================

    if function_name == "get_weather":

        result = get_weather(
            arguments["city"]
        )

    elif function_name == "get_stock_price":

        result = get_stock_price(
            arguments["company"]
        )

    else:
        result = {
            "error": "Unknown tool"
        }

    print("\nFunction Result:")
    print(result)

    # ============================================
    # STEP 8 — SEND TOOL RESULT BACK
    # ============================================

    second_response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[
            {
                "role": "user",
                "content": user_query
            },

            message,

            {
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            }
        ]
    )

    final_answer = (
        second_response
        .choices[0]
        .message
        .content
    )

    print("\nFinal AI Response:")
    print(final_answer)

else:

    print("No tool call required")
    print(message.content)
```

---

# 🖥️ Output

```console
Message: ChatCompletionMessage(
    content=None, refusal=None, 
    role='assistant', annotations=None, 
    audio=None, function_call=None,
    tool_calls=[
        ChatCompletionMessageFunctionToolCall(
            id='hn6esdr3n'
            function=Function(
                arguments='{"city":"Hyderabad"}',
                name='get_weather'
            ),
            type='function'
        )
    ]
)

Tool Selected:
get_weather

Arguments:
{'city': 'Hyderabad'}

Function Result:
20°C

Final AI Response:
The current weather in Hyderabad is 20°C.
```
- content = None: the model did not generate a final human response yet.
- role: this message came from the AI assistant model. [system, user, assistant, tool]


---

# 🔥 3. Multiple Tool Calls

Modern AI models can call **multiple tools simultaneously**.

Example:

```text
Get Hyderabad weather and Apple stock price
```

The model may generate:

```python
tool_calls = [
    get_weather(city="Hyderabad"),
    get_stock_price(company="Apple")
]
```

Your backend executes **both tools** and returns the results.

This is how advanced AI agents work.

---

## 🧠 How Multi-Tool Agents Work

```text
User Request
      ↓
Model Chooses Multiple Tools
      ↓
Backend Executes All Tools
      ↓
Tool Results Returned
      ↓
Model Generates Final Answer
```

---

## 🚀 Complete Code (Multiple Tool Calls)

```python
from openai import OpenAI
import json
import os

# ============================================
# STEP 1 — DEFINE TOOL FUNCTIONS
# ============================================

def get_weather(city):

    weather_data = {
        "delhi": "15°C",
        "mumbai": "12°C",
        "kolkata": "10°C",
        "hyderabad": "20°C",
        "bangalore": "18°C"
    }

    return weather_data.get(
        city.lower(),
        {"error": "Weather data not available"}
    )


def get_stock_price(company):

    stock_data = {
        "apple": "100",
        "microsoft": "200",
        "google": "150",
        "meta": "180",
        "nimbus": "120"
    }

    return stock_data.get(
        company.lower(),
        {"error": "Stock data not available"}
    )


# ============================================
# STEP 2 — CREATE CLIENT
# ============================================

client = OpenAI(
    api_key=os.environ.get("YOUR_OPENAI_API_KEY"),
    base_url="https://api.groq.com/openai/v1",
)

# ============================================
# STEP 3 — TOOL SCHEMAS
# ============================================

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather by city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string"
                    }
                },
                "required": ["city"]
            }
        }
    },

    {
        "type": "function",
        "function": {
            "name": "get_stock_price",
            "description": "Get stock price by company",
            "parameters": {
                "type": "object",
                "properties": {
                    "company": {
                        "type": "string"
                    }
                },
                "required": ["company"]
            }
        }
    }
]

# ============================================
# STEP 4 — USER QUERY
# ============================================

user_query = (
    "Get Hyderabad weather "
    "and Nimbus stock price"
)

# ============================================
# STEP 5 — SEND QUERY TO MODEL
# ============================================

response = client.chat.completions.create(
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

# ============================================
# STEP 6 — MODEL RESPONSE
# ============================================

message = response.choices[0].message

tool_calls = message.tool_calls

# ============================================
# STEP 7 — EXECUTE TOOLS
# ============================================

if tool_calls:

    tool_messages = []

    for tool_call in tool_calls:

        function_name = tool_call.function.name

        arguments = json.loads(
            tool_call.function.arguments
        )

        print("\nTool:", function_name)
        print("Arguments:", arguments)

        # ============================================
        # WEATHER TOOL
        # ============================================

        if function_name == "get_weather":

            result = get_weather(
                arguments["city"]
            )

        # ============================================
        # STOCK TOOL
        # ============================================

        elif function_name == "get_stock_price":

            result = get_stock_price(
                arguments["company"]
            )

        else:

            result = {
                "error": "Unknown tool"
            }

        print("Result:", result)

        # ============================================
        # STORE TOOL RESPONSE
        # ============================================

        tool_messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result)
        })

    # ============================================
    # STEP 8 — SEND RESULTS BACK
    # ============================================

    second_response = client.chat.completions.create(
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

    final_answer = (
        second_response
        .choices[0]
        .message
        .content
    )

    print("\nFinal Response:")
    print(final_answer)

else:

    print("No tool calls required")
```

user_query:
```
"Get Hyderabad weather and Nimbus stock price"
```

Expected output:
```console
Tool Selected: get_weather
Arguments: {'city': 'Hyderabad'}
Function Result: 20°C

Tool Selected: get_stock_price
Arguments: {'company': 'Nimbus'}
Function Result: 120

FINAL AI RESPONSE

The current weather in Hyderabad is 20°C.
The current stock price of Nimbus is 120.
```



---

# 🧩 4. Important Concepts

| Concept              | Description                                   |
| -------------------- | --------------------------------------------- |
| `tools`              | List of functions available to the model      |
| `tool_choice="auto"` | Allows model to decide whether tool is needed |
| `tool_calls`         | Functions selected by model                   |
| `tool_call_id`       | Unique ID used to map tool results            |
| `role="tool"`        | Sends tool output back to model               |
| `json.loads()`       | Parses tool arguments                         |
| Multi-tool calling   | Model can call several tools together         |

---

# 🌍 5. Real-World Use Cases

AI Agents use tool calling for:

* 🌦️ Weather APIs
* 📈 Stock Market APIs
* 🗄️ Database Queries
* 🔍 Web Search
* 🧠 RAG Systems
* 📧 Sending Emails
* 📅 Calendar Scheduling
* 💳 Payments
* 🤖 Autonomous AI Agents

---

# 🎯 Summary

Function Calling enables LLMs to:

* Think
* Decide
* Use tools
* Execute actions
* Return intelligent responses

This is the foundation of:

* MCP Servers
* AI Agents
* Autonomous Workflows
* Tool-Using LLM Applications

---

# ⭐ Next Steps

You can extend this project with:

* [Async tool execution](6. Async Tool Execution)
* Streaming responses
* Real APIs
* MCP protocol integration
* LangChain / CrewAI / Autogen
* Agent memory
* Tool routing
* Retry handling
* Error recovery


---

# 📌 Final Note

This architecture is used by modern AI systems including:

* ChatGPT
* Claude
* Gemini
* Cursor
* Windsurf
* Devin-style AI agents

---


# 🚀 6. Async Tool Execution
Without Async (Sequential)
```
Weather API     → 2 sec
Stock API       → 2 sec
News API        → 2 sec

Total = 6 sec
```

Execution:
```python
weather = get_weather()
stock = get_stock()
news = get_news()
```

## With Async (Concurrent)
```
Weather API     → 2 sec
Stock API       → 2 sec
News API        → 2 sec

Total = 2 sec
```
Execution:
```python
await asyncio.gather(
    get_weather(),
    get_stock(),
    get_news()
)
```

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

# ⚡ 7. Even Better: Dynamic Tool Registry
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

## 🔥 Production Pattern (Recommended)
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







# 🚀 8. Implementing Streaming Responses for Tool Calling Agents
Streaming makes the model send tokens as they're generated instead of waiting for the complete response.

## 8.1. Basic Streaming Example
```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_API_KEY"
)

stream = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {
            "role": "user",
            "content": "Explain AI agents"
        }
    ],
    stream=True
)

for chunk in stream:

    delta = (chunk.choices[0].delta.content)

    if delta:
        print(delta, end="", flush=True)
```
Output appears gradually:
```
AI agents are software systems that...
```

## 8.2. 2️⃣ Streaming with Async
```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def stream_response():

    stream = await client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {
                "role": "user",
                "content": "Explain MCP"
            }
        ],
        stream=True
    )

    async for chunk in stream:

        content = (chunk.choices[0].delta.content)

        if content:
            print(content, end="")
```

## 8.3. Streaming + Tool Calling
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

---

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

## 4️⃣ Full Async Streaming Agent
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

## Stream Events to Frontend (FastAPI)
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

## 🏗 Recommended Production Architecture
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

