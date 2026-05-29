# Tool Routing in LLM Systems

## Table of Contents

* [Why Tool Routing](#why-tool-routing)
* [1. Rule-Based Tool Router](#1-rule-based-tool-router-fast--deterministic)

  * [Router Logic](#router-logic)
  * [Executor](#executor)
  * [Usage Example](#usage-example)
  * [Pros and Cons](#pros-and-cons)
* [2. LLM-Based Tool Router](#2-llm-based-tool-router-recommended-baseline)

  * [Router Prompt](#router-prompt)
  * [LLM Planning Step](#llm-planning-step)
  * [Plan Execution](#plan-execution)
  * [Final LLM Synthesis](#final-llm-synthesis)
  * [Why This Works](#why-this-works)
* [3. Production Tool Router (Graph / DAG)](#3-production-tool-router-graph-based--dag-execution)

  * [Graph Concept](#graph-concept)
  * [Implementation](#implementation-simple-dag)
  * [Execution Engine](#execution-engine)
* [4. Best Practice (Real Systems)](#4-best-practice-what-real-systems-do)
* [5. Minimal Upgrade to Your Original Code](#5-upgrade-your-original-code-minimal-change-version)
* [Summary](#summary)

---

# Why Tool Routing

Tool routing is the logic that determines:

* Which tool(s) should run
* In what order they should run
* Whether tools run sequentially or in parallel
* How outputs are combined

Avoid long if/elif chains:

Bad:
```python
if tool == "weather": ...
elif tool == "stock": ...
```


## Basic Registry

```python
TOOLS = {
    "get_weather": get_weather,
    "get_stock": get_stock_price
}
```


## Dynamic Execution

```python
func = TOOLS.get(name)

result = await func(**args)
```


## Production Design

* Plugin-based tools
* Auto-registration
* Versioned tools
* Sandbox execution


---

## Current Simple Flow (Baseline)

Most basic systems follow this pattern:

```text id="rt1"
User
 ↓
LLM decides tool_calls
 ↓
You execute tools sequentially
 ↓
Return final response
```

---

## Production-Grade Flow

Real systems introduce a routing layer:

```text id="rt2"
User
 ↓
Router (rules + LLM + heuristics)
 ↓
Selected tools (single or multiple)
 ↓
Executor (sync / async / parallel)
 ↓
LLM final response
```

---

# 1. Rule-Based Tool Router (fast + deterministic)

## Idea

Use deterministic logic (no LLM) to select tools.

Best for small systems with known entities.

---

## Router Logic

```python id="rt3"
def tool_router(query: str):

    query_lower = query.lower()

    routes = []

    # WEATHER ROUTING
    weather_cities = ["hyderabad", "delhi", "mumbai", "kolkata", "bangalore"]

    if any(city in query_lower for city in weather_cities):
        routes.append({
            "tool": "get_weather",
            "args": {
                "city": next(city for city in weather_cities if city in query_lower)
            }
        })

    # STOCK ROUTING
    stock_companies = ["apple", "microsoft", "google", "meta", "nimbus"]

    if any(stock in query_lower for stock in stock_companies):
        routes.append({
            "tool": "get_stock_price",
            "args": {
                "company": next(stock for stock in stock_companies if stock in query_lower)
            }
        })

    return routes
```

---

## Executor

```python id="rt4"
def execute_routes(routes):

    results = []

    for route in routes:

        if route["tool"] == "get_weather":
            results.append(get_weather(**route["args"]))

        elif route["tool"] == "get_stock_price":
            results.append(get_stock_price(**route["args"]))

    return results
```

---

## Usage Example

```python id="rt5"
query = "Get Hyderabad weather and Nimbus stock price"

routes = tool_router(query)

results = execute_routes(routes)

print(results)
```

---

## Pros and Cons

### Pros

* Very fast
* No LLM cost
* Fully deterministic

### Cons

* Doesn’t scale well
* No reasoning ability
* Hard to maintain for large tool sets

---

# 2. LLM-Based Tool Router (recommended baseline)

Here the LLM only **plans**, not executes.

---

## Router Prompt

```python id="rt6"
router_prompt = """
You are a tool routing system.

Available tools:
1. get_weather(city)
2. get_stock_price(company)

Return JSON only:

{
  "steps": [
    {
      "tool": "...",
      "args": {...}
    }
  ]
}
"""
```

---

## LLM Planning Step

```python id="rt7"
import json

response = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {"role": "system", "content": router_prompt},
        {"role": "user", "content": user_query}
    ]
)

plan = json.loads(response.choices[0].message.content)
```

---

## Plan Execution

```python id="rt8"
def execute_plan(plan):

    outputs = []

    for step in plan["steps"]:

        if step["tool"] == "get_weather":
            outputs.append(get_weather(**step["args"]))

        elif step["tool"] == "get_stock_price":
            outputs.append(get_stock_price(**step["args"]))

    return outputs
```

---

## Final LLM Synthesis

```python id="rt9"
final = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": f"Tool outputs: {outputs}"}
    ]
)

print(final.choices[0].message.content)
```

---

## Why This Works

* LLM handles reasoning
* Code handles execution
* Easy debugging
* Works with MCP / LangChain / AutoGen

---

# 3. Production Tool Router (Graph / DAG Execution)

## Graph Concept

Tools are modeled as nodes in a DAG.

```text id="rt10"
User Query
   ↓
Router Node
   ↓
Weather Node ───┐
                 ├──→ Aggregator → Final Answer
Stock Node   ────┘
```

---

## Implementation (Simple DAG)

```python id="rt11"
from collections import defaultdict

class ToolGraph:

    def __init__(self):
        self.nodes = defaultdict(list)

    def add_edge(self, from_node, to_node):
        self.nodes[from_node].append(to_node)
```

---

## Define Execution Graph

```python id="rt12"
graph = ToolGraph()

graph.add_edge("router", "weather")
graph.add_edge("router", "stock")
graph.add_edge("weather", "final")
graph.add_edge("stock", "final")
```

---

## Execution Engine

```python id="rt13"
def run(query):

    tasks = []

    if "weather" in query.lower():
        tasks.append("weather")

    if "stock" in query.lower():
        tasks.append("stock")

    results = {}

    if "weather" in tasks:
        results["weather"] = get_weather("hyderabad")

    if "stock" in tasks:
        results["stock"] = get_stock_price("nimbus")

    return results
```

---

# 4. Best Practice (What Real Systems Do)

Production systems combine all three approaches:

```text id="rt14"
Rule-based router
        ↓
LLM planner (structured JSON)
        ↓
Graph executor (parallel + retry + caching)
        ↓
Memory + context injector
        ↓
Final LLM response
```

---

# 5. Upgrade Your Original Code (Minimal Change Version)

Instead of:

```python id="rt15"
tool_calls = message.tool_calls
```

Use routing:

---

## Step 1: Routing Layer

```python id="rt16"
routes = tool_router(user_query)
```

---

## Step 2: Execution Layer

```python id="rt17"
tool_messages = []

for route in routes:

    if route["tool"] == "get_weather":
        result = get_weather(**route["args"])

    elif route["tool"] == "get_stock_price":
        result = get_stock_price(**route["args"])

    tool_messages.append(result)
```

---

## Step 3: Final LLM Call

```python id="rt18"
final = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": str(tool_messages)}
    ]
)
```

---

# Summary

| Approach    | Who Decides Routing          | Best For             |
| ----------- | ---------------------------- | -------------------- |
| Rules       | You (code)                   | Simple apps          |
| LLM Planner | Model                        | Flexible reasoning   |
| Graph/DAG   | System                       | Production workflows |
| Frameworks  | LangChain / CrewAI / AutoGen | Full orchestration   |

---

## Final Insight

Modern AI systems rarely rely on a single routing strategy.

Instead they combine:

* Fast rule-based filtering
* LLM-based planning
* Graph execution engines

to achieve both **speed + intelligence + scalability**.


---

# 🔗 Related

* [Async Execution → `mcp-server-async-execution.md`](mcp-server-async-execution.md)
* [Function Calling → `mcp-server-function-calling.md`](mcp-server-function-calling.md)
