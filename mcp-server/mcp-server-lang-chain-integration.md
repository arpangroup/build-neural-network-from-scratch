# LLM Tool Calling Frameworks: LangChain, CrewAI, and AutoGen

## Table of Contents

* [Overview](#overview)
* [LangChain Implementation](#langchain-implementation)

  * [Install Dependencies](#install-dependencies)
  * [Define Tools](#define-tools)
  * [Create the LLM](#create-the-llm)
  * [Bind Tools](#bind-tools)
  * [Invoke the Model](#invoke-the-model)
  * [Tool Calling Agent](#tool-calling-agent)
* [CrewAI Implementation](#crewai-implementation)

  * [Install Dependencies](#install-dependencies-1)
  * [Define Tools](#define-tools-1)
  * [Create an Agent](#create-an-agent)
  * [Create a Task](#create-a-task)
  * [Create a Crew](#create-a-crew)
* [AutoGen Implementation](#autogen-implementation)

  * [Install Dependencies](#install-dependencies-2)
  * [Register Functions](#register-functions)
  * [Create an Assistant Agent](#create-an-assistant-agent)
  * [Create a User Proxy Agent](#create-a-user-proxy-agent)
  * [Register Tools](#register-tools)
  * [Start the Conversation](#start-the-conversation)
* [Framework Comparison](#framework-comparison)
* [Which One Should You Choose?](#which-one-should-you-choose)
* [Summary](#summary)

---

# Overview

This guide demonstrates how to implement the same **Weather + Stock Price Tool Calling Example** using three popular AI frameworks:

1. **LangChain**
2. **CrewAI**
3. **AutoGen**

The example uses two tools:

* `get_weather(city)`
* `get_stock_price(company)`

and answers:

```text
Get Hyderabad weather and Nimbus stock price
```

---

# LangChain Implementation

## Install Dependencies

```bash
pip install langchain langchain-openai
```

---

## Define Tools

```python
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
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


@tool
def get_stock_price(company: str) -> str:
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
```

---

## Create the LLM

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4.1-mini",
    api_key="YOUR_API_KEY",
    base_url="https://api.groq.com/openai/v1"
)
```

---

## Bind Tools

```python
llm_with_tools = llm.bind_tools([
    get_weather,
    get_stock_price
])
```

### What Happens Here?

LangChain automatically:

* Inspects function signatures
* Generates JSON schemas
* Sends schemas to the model
* Parses tool calls returned by the model

No manual schema definition is required.

---

## Invoke the Model

```python
response = llm_with_tools.invoke(
    "Get Hyderabad weather and Apple stock price"
)

print(response)
```

---

## Tool Calling Agent

For automatic tool execution, use an Agent.

### Create Prompt

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant"),
    ("human", "{input}")
])
```

### Create Agent

```python
from langchain.agents import create_tool_calling_agent

agent = create_tool_calling_agent(
    llm,
    [get_weather, get_stock_price],
    prompt
)
```

### Create Executor

```python
from langchain.agents import AgentExecutor

executor = AgentExecutor(
    agent=agent,
    tools=[
        get_weather,
        get_stock_price
    ]
)
```

### Run

```python
result = executor.invoke({
    "input":
    "Get Hyderabad weather and Nimbus stock price"
})

print(result["output"])
```

### LangChain Handles

* Tool schema generation
* Tool selection
* Tool execution
* Result injection
* Multi-step reasoning

---

# CrewAI Implementation

CrewAI focuses on **collaborating agents** rather than individual tool execution.

---

## Install Dependencies

```bash
pip install crewai
```

---

## Define Tools

```python
from crewai.tools import tool

@tool
def get_weather(city: str):
    ...

@tool
def get_stock_price(company: str):
    ...
```

---

## Create an Agent

```python
from crewai import Agent

finance_agent = Agent(
    role="Financial and Weather Assistant",
    goal="Provide weather and stock information",
    backstory="Expert analyst",
    tools=[
        get_weather,
        get_stock_price
    ]
)
```

### Agent Components

| Component | Purpose                |
| --------- | ---------------------- |
| role      | Defines responsibility |
| goal      | Defines objective      |
| backstory | Provides context       |
| tools     | Available capabilities |

---

## Create a Task

```python
from crewai import Task

task = Task(
    description=
    "Get Hyderabad weather and Nimbus stock price",
    agent=finance_agent
)
```

---

## Create a Crew

```python
from crewai import Crew

crew = Crew(
    agents=[finance_agent],
    tasks=[task]
)
```

### Run

```python
result = crew.kickoff()

print(result)
```

### CrewAI Handles

* Task planning
* Tool selection
* Tool execution
* Agent collaboration
* Workflow orchestration

---

# AutoGen Implementation

AutoGen is designed for autonomous multi-agent systems.

---

## Install Dependencies

```bash
pip install pyautogen
```

---

## Register Functions

```python
def get_weather(city):
    weather_data = {
        "hyderabad": "20°C"
    }

    return weather_data.get(
        city.lower(),
        "not found"
    )


def get_stock_price(company):
    stock_data = {
        "nimbus": "120"
    }

    return stock_data.get(
        company.lower(),
        "not found"
    )
```

---

## Create an Assistant Agent

```python
from autogen import AssistantAgent

assistant = AssistantAgent(
    name="assistant",
    llm_config={
        "model": "gpt-4.1-mini",
        "api_key": "YOUR_API_KEY"
    }
)
```

---

## Create a User Proxy Agent

```python
from autogen import UserProxyAgent

user_proxy = UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER"
)
```

### Human Input Modes

| Mode      | Behavior             |
| --------- | -------------------- |
| ALWAYS    | Ask user every step  |
| TERMINATE | Ask only when needed |
| NEVER     | Fully autonomous     |

---

## Register Tools

```python
user_proxy.register_function(
    function_map={
        "get_weather": get_weather,
        "get_stock_price": get_stock_price
    }
)
```

---

## Start the Conversation

```python
user_proxy.initiate_chat(
    assistant,
    message=
    "Get Hyderabad weather and Nimbus stock price"
)
```

### AutoGen Handles

* Tool selection
* Tool execution
* Multi-turn conversations
* Tool chaining
* Agent collaboration
* Feeding results back to the model

---

# Framework Comparison

| Feature                | Raw Function Calling | LangChain    | CrewAI      | AutoGen            |
| ---------------------- | -------------------- | ------------ | ----------- | ------------------ |
| Complexity             | Low                  | Medium       | Medium      | High               |
| Tool Calling           | Manual               | Automatic    | Automatic   | Automatic          |
| Tool Schema Generation | Manual               | Automatic    | Automatic   | Automatic          |
| Multi-Step Reasoning   | Limited              | Good         | Good        | Excellent          |
| Multi-Agent Support    | No                   | Limited      | Yes         | Excellent          |
| Workflow Orchestration | Manual               | Moderate     | Strong      | Strong             |
| MCP Support            | Manual               | Good         | Emerging    | Good               |
| Learning Curve         | Easy                 | Moderate     | Moderate    | Steeper            |
| Best For               | Small Apps           | RAG & Agents | Agent Teams | Autonomous Systems |

---

# Which One Should You Choose?

## Use Raw Function Calling When

* Building small applications
* Only a few tools are needed
* Full control is required
* Simplicity is preferred

---

## Use LangChain When

* Building RAG systems
* Using vector databases
* Adding memory
* Creating tool-enabled assistants
* Building chains and workflows

### Best Choice For

```text
RAG + Tools + Memory
```

---

## Use CrewAI When

* Multiple specialized agents collaborate
* Agents perform different tasks
* Workflow orchestration is important
* Team-based AI systems are required

### Best Choice For

```text
Agent Teams
```

---

## Use AutoGen When

* Agents communicate with each other
* Long-running autonomous workflows exist
* Multi-turn reasoning is required
* Complex task decomposition is needed

### Best Choice For

```text
Autonomous Multi-Agent Systems
```

---

# Summary

For the Weather + Stock example:

### Raw OpenAI Function Calling

✅ Simplest approach

### LangChain

✅ Best when you'll later add:

* RAG
* Memory
* Vector stores
* Chains
* Retrieval systems

### CrewAI

✅ Best when:

* Multiple agents collaborate
* Tasks are divided among specialists

### AutoGen

✅ Best when:

* Agents talk to each other
* Workflows are autonomous
* Multiple tool calls occur across many steps

---

## Architecture Evolution

```text
Raw Function Calling
        ↓
     LangChain
        ↓
      CrewAI
        ↓
      AutoGen
```

As you move down the stack, you gain:

* More automation
* More orchestration
* More agent capabilities

but also:

* More abstraction
* More complexity
* More learning overhead

**Rule of Thumb:** Start with Raw Function Calling, move to LangChain when you need RAG or memory, adopt CrewAI for agent teams, and choose AutoGen for highly autonomous multi-agent workflows.
