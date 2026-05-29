# 🚀 MCP Server — Function / Tool Calling in Python

> Core foundation of AI agents using tool calling.

> Learn how modern AI models use **Function Calling / Tool Calling** to interact with external tools, APIs, databases, and services.

---

# 📚 Table of Contents

## Core Concepts
- [1. Introduction](#1-introduction)
- [2. Single Tool Call](#2-single-tool-call)
- [3. Multiple Tool Calls](#3-multiple-tool-calls)
- [4. Important Concepts](#4-important-concepts)
- [5. Real-World Use Cases](#5-real-world-use-cases)
- [Next Steps](#next-steps)

## Advanced Architectures
- [6. Async Tool Execution → `mcp-server-async-execution.md`](mcp-server-async-execution.md)
- [7. Tool Routing → `mcp-server-tool-routing.md`](mcp-server-tool-routing.md)
- [8. Streaming + Advanced Agents → (separate file)]()


## MCP Ecosystem (External Modules)
- [MCP Protocol Integration → `mcp-server-protocol-integration.md`](mcp-server-protocol-integration.md)
- [LangChain Integration → `mcp-server-lang-chain-integration.md`](mcp-server-lang-chain-integration.md)
- [Agent Memory → `mcp-server-agent-memory.md`](mcp-server-agent-memory.md)
- [Tool Routing → `mcp-server-tool-routing.md`](mcp-server-tool-routing.md)


---

# 1. Introduction

Modern LLMs can:

- Call functions
- Use tools
- Access APIs
- Query databases

This is called:

- **Function Calling**
- **Tool Calling**
- **AI Agent Tooling**

It is the foundation of MCP-style systems.

---

# 2. Single Tool Call

In this example:

* User asks a question
* Model decides which function to call
* Backend executes the function
* Result is sent back to the model
* Model generates final human-readable response

Step-by-Step Flow:

```
User Query → LLM → Tool Selection → Execution → Response
```

Example use case:
Weather or stock lookup.

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

## 🖥️ Output

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



# 3. Multiple Tool Calls

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


## 🧠 How Multi-Tool Agents Work
```
User Query
      ↓
Model Chooses Multiple Tools
      ↓
Backend Executes All Tools
      ↓
Tool Results Returned
      ↓
Model Generates Final Answer

```
Example:
- Weather + Stock price in one query
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

# 5. Real-World Use Cases

- Weather APIs
- Stock APIs
- DB queries
- RAG systems
- Email automation
- Scheduling agents

---

# Next Steps

- [Async execution → `mcp-server-async-execution.md`](mcp-server-async-execution.md)
- [Tool routing → `mcp-server-tool-routing.md`]()
- [Streaming response → `mcp-server-streaming.md`]()
- [MCP protocol integration → `mcp-server-protocol-integration.md`]()
- [LangChain / CrewAI / Autogen → ``]()
- [Agent Memory → ``]()
- Retry handling
- Error recovery



