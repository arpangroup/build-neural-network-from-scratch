# 🚀 MCP Server — Tool Calling with Python (Model Context Protocol)

> Core foundation for building AI agents using structured tool execution via the Model Context Protocol (MCP).

Modern AI systems use tool calling to interact with external APIs, databases, and services.
Model Context Protocol standardizes how these tools are exposed, discovered, and executed across AI applications.

---

# 📚 Table of Contents

## Core Concepts
- [1. Tool Calling Fundamentals](mcp-server-function-calling.md)
- [2. Single Tool Call](mcp-server-function-calling.md#2-single-tool-call)
- [3. Multiple Tool Calls](mcp-server-function-calling.md#3-multiple-tool-calls)
- [4. Async Tool Execution → `mcp-server-async-execution.md`](mcp-server-async-execution.md)
- [5. Tool Routing → `mcp-server-tool-routing.md`](mcp-server-tool-routing.md)
- [6. Streaming + Advanced Agents → (separate file)]()


## MCP Ecosystem (Integrations)
- [7. MCP Protocol Integration → `mcp-server-protocol-integration.md`](mcp-server-protocol-integration.md)
- [8. LangChain Integration → `mcp-server-lang-chain-integration.md`](mcp-server-lang-chain-integration.md)
- [9. Agent Memory → `mcp-server-agent-memory.md`](mcp-server-agent-memory.md)
- [10. Tool Routing → `mcp-server-tool-routing.md`](mcp-server-tool-routing.md)


---

# A Simple Tool Calling Mechanism
```python

def get_weather_details(city):
  weather_data = {
      "kolkata": "10°C",
      "mumbai": "12°C",
      "delhi": "15°C",
      "hyderabad": "20°C"
  }
  return weather_data.get(city.lower(), "Weather data not available")

def add_two_number(x, y):
  return x+y

system_prompt = """
You are an AI assistant who is designed to resolve user query.
You work on START, PLAN, ACTION, OBSERVE and OUTPUT mode.

In the start phase, user gives a query to you.
Then, you PLAN how to resolve that query atleast 3-4 times and make sure that all arguments are captured

After planning take the action with appropriate tools and wait for observation based on action.
If you think, user query needs a tool invocation, just tell me the tool name with parameters.
And then call an ACTION event with tool and input parameters
If there is an action call, wait for the OBSERVE that is output of the tool.

Based on the OBSERVE from previous step, you either OUTPUT or repeat the PLAN and ACTION steps.

Rules:
- Always wait for the next step.
- ALways output a single step and wait for the next step.
- Output must be strictly JSON
- Only call tool action from Available tools only.
- STrictly follow the output format in JSON

Available Tools:
- get_weather_details(city: string): string
- add_two_number(x: number, y: number): number

Example 1:
START: What is 2 + 3?
PLAN: From the available tools, I must call add_two_number tool for 2 and 3 as input
ACTION: Call Tool add_two_number(2, 3)
OBSERVE: 5
PLAN: The sum of 2 and 3 is 5.
OUTPUT: The sum of 2 and 3 is 5.

Example 2:
START: What is weather of Delhi?
PLAN: From the available tools, I must call get_weather_details tool for Delhi as input
ACTION: Call Tool get_weather_details(Delhi)
OBSERVE: 15 degree C
PLAN: The weather of Delhi is 15 degree C.
OUTPUT: The weather of Delhi is 15 degree C.

Output Format:
{"Step": "string", "tool": "string", "input": "string", "content": "string"}

"""


chat_completion = client.chat.completions.create(
    model="llama-3.3-70b-versatile",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system","content": system_prompt},
        {"role": "user","content": "what is weather of Delhi"},
        {"role": "assistant", "content": '{"Step": "PLAN", "tool": "get_weather_details", "input": "Delhi", "content": "From the available tools, I must call get_weather_details tool for Delhi as input"}'},
        {"role": "assistant", "content": '{"Step": "ACTION", "tool": "get_weather_details", "input": "Delhi", "content": "Call Tool get_weather_details(Delhi)"}'},
        {"role": "assistant", "content": '{"Step": "OBSERVE", "content": "15 degree C"}'},
    ]
)

print(chat_completion.choices[0].message.content)
```

output:
```json
{"Step": "OUTPUT", "content": "The weather of Delhi is 15 degree C"}
```

# Fully Automated Loop
A simple agent loop:
```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": "what is weather of Delhi"}
]

while True:
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        response_format={"type": "json_object"},
        messages=messages
    )

    output = json.loads(response.choices[0].message.content)
    messages.append({"role": "assistant", "content": json.dumps(output)})

    if output["Step"] == "OUTPUT":
        print(output["content"])
        break

    if output["Step"] == "ACTION":
        tool = output["tool"]

        if tool == "get_weather_details":
            result = get_weather_details(output["input"]["city"])

        elif tool == "add_two_number":
            result = add_two_number(output["input"]["x"], output["input"]["y"])

        messages.append({
            "role": "user",
            "content": json.dumps({"Step": "OBSERVE", "content": result})
        })
```